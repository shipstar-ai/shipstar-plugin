---
name: email-your-users
description: >
  Send a release-notes email to the project's mailing lists via Shipstar:
  generate the email from recent commits, review it with the user, pick the
  lists, and send with one-click unsubscribe handling. Use when the user
  wants to email customers/users about a release or send release notes.
  Requires the Shipstar MCP server.
---

# Email release notes to your users

This flow sends real email to real people. It has two hard rules: the user
sees the final draft and the exact audience before anything sends, and each
release email can only ever be sent once.

## 1. Check the audience first

Call `list_mailing_lists`. If there are no lists, or every list has zero
`active_recipients`, don't generate an email nobody will receive. Instead
offer to build the audience:

- No list yet → `create_mailing_list(name)`.
- The user has addresses to hand (pasted, from a CSV, from their CRM) →
  `upsert_mailing_list_recipients(mailing_list_id, recipients)` with
  `{email, name?, external_id?}` objects, ≤ 500 per call. Include their own
  user id as `external_id` when they have one so later syncs update rather
  than duplicate. Show the returned counts (`added`, `updated`,
  `skipped_unsubscribed` …) back to the user.
- For an ongoing sync from their own system, point them at the docs guide
  "Sync your users into a mailing list" (`PUT /api/v1/email/lists/{id}/recipients`)
  or the dashboard's CSV import (Destinations → Email → the list).

Removing people is `remove_mailing_list_recipients` (soft; confirm the exact
people first). Contacts who unsubscribed themselves can never be re-added by
any tool — they are only ever reported as skipped.

## 2. Generate

Call `generate_release_notes_email` with the commit window (default: last 7
days, or since the last release if the user says so). If the user says who
the recipients are or what to leave out, pass `audience` (`"technical"`,
`"business"`, or `"mixed"`) and/or `instructions` (free text, ≤ 2000 chars);
otherwise omit both and the project's `content_guidelines` defaults apply. Poll
`get_generation_status` every 20–30 seconds until `completed` (normally 1–3
minutes); on `failed` report `error_message`.

If a completed release-notes email already exists for this window (the user
mentions one, or a generate attempt is redundant), you can reuse its
`content_id` — but check `is_published` / send state first: an already-sent
email cannot be re-sent.

## 3. Review

Show the draft from `get_content_draft`. The content is structured JSON
(subject/headline, summary, items) — render it as the email will read, not as
raw JSON. Apply edits with `update_content`, preserving the JSON shape
exactly. Iterate until the user approves the copy.

## 4. Send — explicit confirmation, stated audience

- Have the user pick the mailing list(s) by name.
- State the deduplicated recipient reach before sending, e.g. "This goes to
  ~140 active recipients across Customers and Beta Users. Send it?"
- Only on a clear yes: `send_release_email(content_id, mailing_list_ids)`.

The send is queued and fans out in the background; every email carries a
one-click unsubscribe link (unsubscribes deactivate the recipient
permanently — Shipstar never re-adds an opted-out address). The response's
`recipient_count` is the deduplicated audience.

If the tool errors with "already been sent": that content record was used.
Sending the same news again requires generating a fresh email — confirm with
the user that a duplicate send is really intended before doing that.

## 5. Report

Confirm queued delivery and the recipient count. Delivery stats (sent/failed)
appear in the dashboard once the queue drains.
