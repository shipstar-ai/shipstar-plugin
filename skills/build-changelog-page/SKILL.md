---
name: build-changelog-page
description: >
  Build a server-rendered, agent-readable changelog page on the user's own
  website from their Shipstar changelog data: semantic HTML crawlers and
  answer engines can actually read, per-period permalink pages, an RSS
  feed, and structured data. Use when the user wants their changelog on
  their own domain, asks to make their changelog crawlable / SEO-friendly /
  readable by AI agents, or notes that the embed widget renders nothing
  for search engines. Requires a Shipstar API token; works in any
  server-rendering web stack (examples assume Next.js-style ISR).
---

# Build an agent-readable changelog page

The Shipstar embed widget renders client-side into shadow DOM — humans see
it, but crawlers and AI answer engines (GPTBot, ClaudeBot, PerplexityBot)
fetch the page HTML and see nothing. This skill implements what
shipstar.ai/changelog itself does: fetch the changelog data server-side and
ship it as real HTML. Adapt the specifics to the user's framework; the
invariants below are what make the page readable to machines.

## 1. Establish access

- Ask for (or locate in their env) a Shipstar API token — created in the
  dashboard under Keys. **It must stay server-side** (env var, never client
  code): `GET https://api.shipstar.ai/api/v1/changelogs` with
  `Authorization: Bearer <token>` returns *their* project's published
  changelogs, newest first. Without a token the endpoint serves Shipstar's
  own changelog — if the user sees Shipstar's entries instead of theirs,
  the token isn't being sent.
- Inspect the real payload before writing rendering code. Each period:
  `{ period_start, period_end, headline, entries, slug, commit_count }`;
  each entry: `{ id, category, title, description, date }` with category
  one of `new | improved | fixed | breaking`. Dates are date-only strings
  (`YYYY-MM-DD`).

## 2. Render server-side — the non-negotiables

Build the index page (e.g. `/changelog`) as a server-rendered route with
short-lived caching (revalidate ~300s). Verify by fetching the page with
curl: every entry must appear in the response HTML with JavaScript off.

- One `<article>` per period with a stable, human-readable anchor id
  (derive from `period_start`, e.g. `whats-new-2026-08-14`).
- The generated `headline` is the period's heading — but it is a full
  sentence, so set it at body-plus scale (not display type), or it becomes
  a multi-line wall on phones.
- Entries as a list with the category label *inside* each entry heading
  (screen-reader heading navigation should announce "Breaking Change …"),
  color-coded by severity, with per-entry anchor ids
  (`<period-anchor>-<entry.id>` — entry ids are only unique per period).
- Every date in a `<time dateTime="...">`, **formatted in UTC** — date-only
  strings parse as UTC midnight, and local formatting shows the previous
  day for visitors west of UTC.
- A provenance line per period when `commit_count` is a number
  ("Generated from N commits"); omit the number when it's null — never
  invent one.

## 3. Permalinks, feed, structured data

- **Permalink pages**: every period has a public unguessable `slug`;
  `GET /api/v1/changelog/{slug}` needs no auth. Create
  `/changelog/[slug]` routes (prerender from the list, revalidate ~300s)
  with a canonical URL, the period as `og:type: article`, and
  `TechArticle` JSON-LD. Add the permalinks to the sitemap with
  `lastModified` from `period_end`.
- **Feed**: `GET /api/v1/changelogs/feed` returns project-scoped RSS but
  requires the bearer token, which a feed reader can't send — so proxy it:
  implement `/changelog/feed.xml` on their site that fetches with the
  token server-side and serves the XML (rewrite item links to their
  permalink pages if they build the feed themselves). Advertise it with
  `<link rel="alternate" type="application/rss+xml">` on the changelog
  pages.
- **Index JSON-LD**: an `ItemList` of the periods, names from headlines,
  URLs to the permalinks.

## 4. Fail closed

An API hiccup must never render-and-cache an empty page, feed, or sitemap
— feed readers interpret a valid empty feed as "every entry was deleted",
and a cached empty sitemap delists the permalinks. On a failed fetch:
throw (so stale cached output keeps serving) or return an error status.
Reserve the empty state for a genuinely empty (200, `[]`) response.

## 5. Optional enhancements

- Keep the embed widget as progressive enhancement on top of the
  server-rendered list if they already use it — never as the only
  rendering path.
- A subscribe form can POST `{ email, public_slug }` to
  `POST /api/v1/changelog/subscribe` (no auth; double-opt-in; always
  answers with a neutral confirmation) using the newest period's slug.
- At 6+ periods, render the latest ~6 in full and collapse older ones to
  headline rows linking to their permalinks so the index stays bounded.

## 6. Verify before finishing

- `curl` the index and one permalink: full entry text present, `<time>`
  elements carry `dateTime`, JSON-LD parses as valid JSON.
- Titles: permalink `<title>` leads with the period headline (truncated
  sanely), not a generic template string.
- Feed validates and its item links resolve; sitemap lists the permalinks.
- Load the page with a phone-width viewport: no horizontal overflow, and
  the headline heading doesn't fill the whole first screen.
