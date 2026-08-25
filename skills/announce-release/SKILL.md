---
name: announce-release
description: >
  Announce a release with Shipstar: generate a changelog and release-notes
  email from recent commits, review the drafts with the user, then publish
  and email subscribers. Use when the user says they shipped, released, or
  launched something and wants it announced, or asks to "run the release
  marketing". Requires the Shipstar MCP server to be connected.
---

# Announce a release

You are driving Shipstar's content pipeline over MCP: generate → poll →
review → approve → deliver. The user stays in the loop at exactly one point —
approving drafts. Never publish or send email without their explicit
confirmation in this conversation.

## 1. Establish context

Call `get_project_context`. Confirm:
- `github_connected` is true, `github_suspended` is false, and
  `tracked_repositories` covers the repo the user shipped from. If not, stop
  and send them to the Shipstar dashboard (Sources → Install GitHub App, then
  pick repos) — you cannot connect repos over MCP.
- Note which destinations are connected and which mailing lists exist; you
  will offer these at delivery time.
- Note `content_guidelines` — the project's default technical depth
  (`audience`) and free-text instructions. They apply automatically; you only need to act if
  the user asks for something different.

Work out the commit window. Default: since the previous release/changelog
(check `list_changelogs` for the last published date) or the last 7 days,
whichever is shorter. If the user named a version, tag, or date range, use
that. Pass it as `start_date`/`end_date`.

## 2. Generate

Kick off what the user asked for. For a standard release announcement:
- `generate_changelog` — the public changelog entry
- `generate_release_notes_email` — only if mailing lists with active
  recipients exist

If the user says who the announcement is for or what to leave out ("keep it
non-technical", "don't mention the marketing site changes", "call out the
API changes"), pass `audience` (`"technical"`, `"business"`, or `"mixed"`)
and/or `instructions` (free text, ≤ 2000 chars) on the same call. Otherwise
omit both — the project defaults from `content_guidelines` apply. If a draft
comes back with the wrong emphasis, prefer regenerating with `instructions`
over hand-editing large sections.

If the project tracks several repositories and the user is announcing work
from one of them, pass `repos: ["owner/name"]` (names from
`tracked_repositories`) so the draft only covers that repo; otherwise omit it
and every tracked repo is summarized.

Each returns a `content_id` immediately; generation runs in the background.

## 3. Poll

Poll `get_generation_status` with each `content_id` every 20–30 seconds.
Generation normally completes in 1–3 minutes; the queue processes two jobs at
a time, so parallel kickoffs may finish sequentially. Stop polling on
`completed` or `failed` (report `error_message` and stop). Do not re-generate
on slowness — only on failure, and at most once.

## 4. Review with the user

Fetch each draft with `get_content_draft` and show it to the user in full.
Changelogs and release-notes emails are JSON — render them readably
(headline, summary, items), don't dump raw JSON.

Apply the user's edits with `update_content`. It replaces the whole draft:
send the complete revised text, and for JSON types keep the exact same JSON
shape. Re-show after editing. Iterate until the user says it's good.

## 5. Deliver — only on explicit confirmation

These are the irreversible steps. Get a clear "yes" for each:

- **Publish the changelog**: `approve_content` with the changelog's
  `content_id`. If its scheduled time has passed it goes live immediately;
  otherwise it publishes at the scheduled time — tell the user which.
- **Send the email**: `list_mailing_lists`, let the user pick, confirm the
  recipient count out loud, then `send_release_email`. Each release email can
  be sent exactly once — if it errors with "already been sent", it was; do
  not try to work around that.

## 6. Report

Summarize: what was published (with `public_slug` links where present), how
many recipients the email went to, and anything skipped or failed.
