---
name: sync-your-users
description: >
  Mirror the user base of the user's own product into a Shipstar mailing
  list so release-notes and changelog emails reach exactly the people who
  should get them: an idempotent nightly reconcile keyed by the product's
  own user id, best-effort hooks on signup / preference change / account
  deletion, the attested resubscribe call for people who opt back in, and
  a signed webhook handler that mirrors Shipstar-side unsubscribes and
  resubscribes into the product's own "product updates" flag. Use when the
  user wants their customers or users on a Shipstar list automatically,
  asks to "sync users to Shipstar", to wire a "product updates" setting
  to Shipstar, or to handle unsubscribe webhooks. Requires a Shipstar API
  token; works in any backend stack (examples are language-neutral HTTP).
---

# Sync your users into a Shipstar mailing list

A Shipstar mailing list is the audience for release-notes emails and for
the optional summary a changelog schedule sends. For a real product the
list should be a mirror of the product's own users: sign up and the
emails start, close the account and they stop, change address and the
next issue reaches the new one, click unsubscribe and nothing in the
product ever puts you back without your say-so. This skill builds that
mirror in the user's codebase against the public REST API
(`https://api.shipstar.ai/api/v1`, Bearer token). The invariants are what
keep the list honest; adapt the plumbing to their stack.

Read the API docs when a detail here is not enough:
`https://docs.shipstar.ai/guides/sync-mailing-list` and
`https://docs.shipstar.ai/api-reference/mailing-lists/upsert-recipients`.

## 1. Establish access and the list

- Ask for (or locate in their env) a Shipstar **API token**, created in the
  dashboard under API Tokens. It is scoped to one Shipstar project. **Server
  side only** — an env var such as `SHIPSTAR_API_TOKEN`, never client code.
- Find or create the list: `GET /email/lists` returns the project's lists
  with counts and `max_active_recipients` (the plan cap); `POST /email/lists
  {"name": "Users"}` creates one. Store the id as config
  (`SHIPSTAR_MAILING_LIST_ID`), not as a hardcoded literal, and make every
  sync path a **no-op while the token or list id is unset**, so staging and
  tests never touch a real list.
- Decide, with the user, the **list rule**: which accounts belong. The
  usual one is *active + email verified + "product updates" preference on*.
  Write it once as a single query/predicate and use it everywhere below;
  two definitions drift. If the product has no such preference yet, add
  one (default on for existing accounts is a soft opt-in; check with the
  user) with a switch in account settings.

## 2. Key by the product's own user id — the non-negotiables

- Every entry carries `external_id` = **the product's user id**, plus
  `email` and an optional `name`. Shipstar matches on `external_id` first,
  so an email change updates the same recipient instead of creating a
  second one, and removals/erasures can be keyed by id without knowing the
  current email.
- Add or update with `PUT /email/lists/{id}/recipients
  {"recipients": [...]}`, up to 1000 entries per call, batches of ~500 in
  practice. It is idempotent per entry; the response counts `added`,
  `updated`, `reactivated`, `unchanged`, `skipped_unsubscribed`, `invalid`
  and `conflicts`. Log the counts.
- **`skipped_unsubscribed` is final.** That person clicked the unsubscribe
  link in an email. No add, upsert, restore or retry will change it, and
  the code must never try to; the only way back is section 4.
- **Upsert never deletes.** Departed users have to come off explicitly:
  `POST .../recipients/remove {"external_ids": [...]}` (soft: a later
  upsert puts them back) for people who no longer qualify, and
  `POST .../recipients/erase {"external_ids": [...]}` (permanent, any
  status — GDPR Art. 17) when the **account itself is deleted**. Never
  "remove" a deleted account; erase it, so nothing about the person stays.

## 3. The two sync paths

**Nightly reconcile (the backstop).** A scheduled job, once a day, that
makes the list equal the list rule:

1. Page every *active* recipient with `GET .../recipients?status=active&
   limit=500` following `next_cursor`, and collect their `external_id`s.
   Ignore rows with no `external_id` (hand-added or CSV imports are not
   the sync's to manage).
2. Diff against the ids the list rule returns. Ids on the list but not in
   the rule: erase the ones whose account is gone, remove the rest. **Do
   this first** — it frees room under the plan cap and honours opt-outs
   even when the list is full.
3. Upsert everyone the rule returns, in batches.
4. Record the totals somewhere a human will see them (job result, log
   line, metric). After a run, `GET /email/lists/{id}` → `active_count`
   should equal the rule's count minus the unsubscribed.

**Hooks (fast path, best effort).** On email verification, on the
preference switch, and on account deletion, push *that one user's current
state*: erase if the account is gone, remove if they no longer qualify,
otherwise upsert. Run it **off the request path** (a queued job, a
background task) so a Shipstar outage never slows or fails the user's own
action; two quick toggles then converge on whatever the state is when the
job runs. The reconcile covers anything a hook misses (OAuth signups,
failed jobs, bulk changes), so hooks do not need retries beyond their queue.

## 4. The way back from an unsubscribe

When a person who unsubscribed later turns the product's "product
updates" switch **on**, do not upsert. Call

```
POST /email/lists/{id}/recipients/resubscribe
{"external_id": "<user id>", "email": "<email>", "name": "<name>",
 "consent": {"source": "<where they opted in, in words>",
             "recorded_at": "<ISO 8601 of the click>",
             "ip": "<request ip>", "user_agent": "<request user agent>"}}
```

One recipient per call; `consent.source` is required and is stored on the
recipient as the consent record (e.g. `"Product updates switch, account
settings"`). The response `status` is `resubscribed` (they had
unsubscribed), `added` (not on the list or erased earlier), `reactivated`
(you had removed them) or `already_active` — treat all four as success.
Pass the switch-on timestamp from the hook into the job so a delayed job
still records the real moment. The switch-**off** path stays a plain
remove; the person's own unsubscribe link keeps working regardless.

Never expose a bulk or admin path to this call: it is an attestation that
*this person* opted in *here*, and a job that sends it for everyone would
be re-subscribing people against their choice.

## 5. The webhook — mirror Shipstar's side into the product

Two things change on the list without the product knowing: the person
clicks the unsubscribe link in an email, or clicks **Resubscribe** on the
page that link opens. Shipstar POSTs `mailing_list.unsubscribed` and
`mailing_list.resubscribed` to the team's webhook endpoints.

- Add an endpoint in the product (e.g. `POST /api/external/shipstar/webhook`)
  and register it in the Shipstar dashboard under Destinations → Webhooks,
  ticking **both** events. Store the endpoint's signing secret as config.
- **Verify the signature before anything else.** Header
  `X-Shipstar-Signature: t=<unix seconds>,v1=<hex>`, where `v1` is
  HMAC-SHA256 of `"<t>.<raw request body>"` under the signing secret.
  Compare in constant time, reject `t` older than ~5 minutes, and use the
  raw body bytes, not a re-serialised JSON object.
- **Filter on `data.list_id`.** Events are dispatched for every list the
  team has, so ignore any whose `list_id` is not the configured one.
- Resolve the user by `data.external_id` (fall back to `data.email` only for
  rows the sync did not create), then set the preference: `unsubscribed` →
  off, `resubscribed` → on. On a `resubscribed` event with
  `data.source == "api"` the product's own call caused it; setting the flag
  on again is harmless, so no special case is needed.
- Answer **2xx quickly** and do the work inline only if it is one database
  write; otherwise enqueue. Deliveries retry on network errors and 5xx; a
  4xx is treated as final, so a handler that 400s on an unknown user has
  lost that event for good — log and return 200 instead.
- `X-Shipstar-Event` names the event and `X-Shipstar-Delivery` is a
  per-delivery id you can use for idempotency.

## 6. The plan cap

Each list holds at most `max_active_recipients` active recipients (500 on
Free, 2,500 on Solo, 10,000 on Team). A write that would cross it is
rejected whole with HTTP 400 and this body:

```json
{"detail": "...", "code": "recipient_cap_exceeded",
 "active_count": 498, "limit": 500, "incoming": 5}
```

Nothing is written, and a retry cannot help. Detect `code ==
"recipient_cap_exceeded"`, stop the run, and **alert** with
`active_count` and `limit` (the fix is a bigger Shipstar plan or a smaller
list, a human decision). Because the reconcile removes before it upserts,
a full list still honours departures. Optionally read
`max_active_recipients` from the list and warn at ~90 %.

## 7. Verify before finishing

- With the token and list id **unset**, every path is a no-op and the
  test suite passes without network access (the Shipstar client is faked
  in tests).
- Run the reconcile once by hand: the counts it logs match
  `GET /email/lists/{id}` (`active_count`), and a user who fails the list
  rule is not on the list.
- Flip one test account's switch off, then on: the first push is a remove,
  the second goes through **resubscribe** (not upsert) and the response
  `status` is `reactivated` or `resubscribed`.
- Use the dashboard's **Send test** on the webhook endpoint: the handler
  verifies the signature and answers 2xx. Then unsubscribe a test
  recipient through the real link in a sent email, confirm the product's
  flag flipped off, click **Resubscribe** on the page, confirm it flipped
  back on.
- Deliberately send a batch that would exceed the cap in a test (fake the
  400 body): the job stops and alerts rather than looping.
- Delete a test account: the recipient is **erased** (gone from every
  status filter), not merely removed.
- Secrets: the API token and webhook secret exist only as server-side
  config; nothing in client bundles or logs.
