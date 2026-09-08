---
name: add-announcement-banner
description: >
  Put a Shipstar announcement banner at the top of the user's website: a
  one-line bar (headline, one sentence, call-to-action) announcing the most
  impactful change of the period, generated from their commits, reviewed,
  published, and auto-updated by a weekly or monthly schedule. Covers
  generating the first banner over MCP, embedding it with one script tag
  (or rendering it server-side from the API), and setting up the
  schedule. Use when the user wants to highlight what shipped at the top
  of their site, asks for an announcement bar / site banner / "what's new"
  strip, or wants visitors to notice a feature this week or this month.
---

# Add an announcement banner

Shipstar generates a `banner` content type: `{ badge ≤ 16 chars ("New" by
default, "" for none), headline ≤ 80 chars, body ≤ 160 chars (one sentence),
cta_label ≤ 24 chars, link_url }` for the single
most impactful user-facing change in a commit window. **Exactly one banner is
live per project.** Publishing a new one replaces the current one and keeps
the same public slug, so the customer's site embeds the key once and every
later banner appears with no change to their page. This skill sets that up
end to end.

## 1. Check the starting point

- Call `get_project_context` and confirm a GitHub source with tracked
  repositories. Without one, stop and point the user at the Sources page.
- Call `get_banner`. If one is live, tell the user what it says and that a
  new publish will replace it; otherwise the first publish will mint the
  project's banner slug (`<project-slug>-banner`).
- Ask where the banner should click through to. Default is the project's
  Changelog page URL (Destinations → Website), then the period's published
  changelog permalink; the user can name any URL (a launch page, docs,
  pricing). Ask whether it should come down on its own (7 / 14 / 30 days)
  or stay until the next banner replaces it (the default).
- The bar shows a small "New" badge before the headline. Keep it unless
  the user wants different wording (`badge`, e.g. "Beta") or none
  (`show_badge: false`) — don't ask unless they bring it up.

## 2. Generate and review the first banner

- `generate_banner` with `link_url` (if the user named one), optional
  `expires_after_days`, `badge` / `show_badge` if they changed the badge,
  and `repos` if they want one repo only. It costs 25
  credits. Poll `get_generation_status` until `completed`.
- `get_content_draft` and show the user the badge, headline, body, and CTA
  label exactly as they will render — the bar is one line, so read it aloud
  as a sentence. Offer edits; apply them with `update_content` (the draft
  is JSON: keep `link_url` in it, and never invent a link the user didn't
  give — an empty `link_url` is filled in at publish from their website
  settings; set `badge` to `""` to drop the tag).
- Only after the user confirms, `publish_content`. Read back `get_banner`
  and quote the `slug` — it is the embed key.

## 3. Put it on the site

Fastest path (client-side, sticks to the top of the page, dismissable —
a dismissed banner stays hidden for that visitor until a new one is
published):

```html
<script
  src="https://shipstar.ai/embed.js"
  data-shipstar-key="<slug from get_banner>"
  data-type="banner"
  data-theme="auto"
></script>
```

Paste it anywhere in the page (in `<head>` is fine — the bar mounts at the
top of `<body>` once it exists). Options: `data-position="inline"` renders it
where the tag is instead of sticking to the top; `data-dismissible="false"`
removes the close button; `data-branding="false"` hides the small Shipstar
mark (paid plans). The bar's colours follow CSS variables on the mount
element: `--ss-banner-bg` and `--ss-banner-text`; with `data-container` you
can set them on your own element.

Server-rendered path (no client JS, banner in the HTML): fetch
`GET https://api.shipstar.ai/api/v1/banner` with `Authorization: Bearer
<API token>` (server-side only) and `X-Shipstar-Page-Url: <the page's
URL>`; a **404 means no banner is live — render nothing**, it is not an
error. Cache for ~5 minutes. Render `badge` (a small tag before the headline; skip it when empty),
`headline`, `body`, and a link with `cta_label` → `link_url` (omit the link
when `link_url` is null). Respect
`expires_at` if you cache longer than its remaining window.

## 4. Keep it fresh

Offer a schedule so the banner updates itself: the dashboard's Generate
page → Announcement Banner → weekly (covers the week since the last run) or
monthly, with the "Link to", "Badge", and "Take down after" choices saved
on the schedule. Each occurrence generates a draft, goes through review (or
auto-publishes when review is off for banners under Destinations →
Website), and replaces the live banner on publish. There is no MCP tool for
creating schedules yet — link the user to `/dashboard/generate?schedule=banner`.

## Rules

- Never publish without the user seeing the exact text first; the bar is
  the most visible thing on their site.
- Never paste a `link_url` you made up. If the user has no page in mind,
  leave it empty and let Shipstar resolve it at publish.
- Don't describe fixes as outages ("broken for weeks"); say what now works.
