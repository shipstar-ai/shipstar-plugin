---
name: build-changelog-page
description: >
  Build a server-rendered, agent-readable changelog page on the user's own
  website from their Shipstar changelog data: semantic HTML crawlers and
  answer engines can actually read, per-period permalink pages, an RSS
  feed, structured data, and the SEO / AEO layer (titles, descriptions,
  Open Graph, crawler policy, llms.txt, answer-shaped copy) that makes it
  rank and get cited. Use when the user wants their changelog on their
  own domain, asks to make their changelog crawlable / SEO-friendly /
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
- On every server-side fetch, send `X-Shipstar-Page-Url` with the
  canonical URL of the page being rendered (the index URL when fetching
  the list, the permalink URL when fetching a period by slug). Shipstar
  stores it, so the user's dashboard shows "Published · their-site/…"
  automatically — server-rendered pages send no Referer, and without this
  header the dashboard keeps saying "not on your site yet".
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
- A **"Powered by Shipstar" badge** at the bottom of the index and of every
  permalink page — the same small link shipstar.ai's own changelog and the
  embed widget show. Free-plan projects must keep it; paid plans may remove
  it. Render it as a quiet, centered text link (muted colour, ~12px,
  brighter on hover), never a banner:

  ```html
  <p class="shipstar-badge">
    <a href="https://shipstar.ai?utm_source=changelog&utm_medium=badge"
       target="_blank" rel="noopener noreferrer">
      <svg width="14" height="14" viewBox="0 0 195 195" aria-hidden="true"
           xmlns="http://www.w3.org/2000/svg">
        <path fill="currentColor" d="M123.215 24.3304C125.235 21.8047 129.145 21.8047 128.921 24.3304L124.507 73.9972C124.406 75.1267 125.218 75.8915 126.518 75.8915H183.666C186.572 75.8915 186.27 79.2898 183.225 80.8508L123.349 111.547C121.987 112.245 120.997 113.482 120.897 114.612L116.482 164.278C116.258 166.804 112.161 168.904 110.504 167.343L77.9123 136.648C77.1711 135.949 75.7473 135.949 74.3856 136.648L14.5092 167.343C11.4643 168.904 9.23429 166.804 11.2549 164.278L50.9888 114.612C51.8924 113.482 52.0024 112.245 51.2612 111.547L18.6698 80.8508C17.0125 79.2898 19.7312 75.8915 22.6373 75.8915H79.7856C81.0852 75.8915 82.577 75.1267 83.4806 73.9972L123.215 24.3304Z"/>
      </svg>
      Powered by Shipstar
    </a>
  </p>
  ```

  Keep the star and the wording exactly; adapt only the styling to the
  site. Place it after the last period on the index (before any subscribe
  form or site footer) and after the entries on a permalink page.

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

One exception: a `404` from a single-slug endpoint
(`GET /api/v1/changelog/{slug}`, or a knowledge-base set by slug) is the
API's real answer — that item is unpublished, not the API down — so treat
it as "nothing here" (a 404 page, or no entries), never as a failure. Only
network errors and non-404 error statuses fail closed.

## 5. Optional enhancements

- Keep the embed widget as progressive enhancement on top of the
  server-rendered list if they already use it — never as the only
  rendering path.
- A subscribe form can POST `{ email, public_slug }` to
  `POST /api/v1/changelog/subscribe` (no auth; double-opt-in; always
  answers with a neutral confirmation) using the newest period's slug.
- At 6+ periods, render the latest ~6 in full and collapse older ones to
  headline rows linking to their permalinks so the index stays bounded.

## 6. Optimise for search and answer engines

Sections 2–3 make the changelog *readable*; this makes it *rank* and *get
cited*. Do all of it — it is mostly metadata and copy shape, not new pages.

### SEO

- **Titles and descriptions**: index `<title>` = "Changelog — {Product}"
  (or "What's new in {Product}"); permalink `<title>` = the period headline
  truncated at ~60 chars, then " — {Product}". Every page gets a
  `<meta name="description">` of 150–160 chars: for permalinks, the headline
  plus the first entry's title; for the index, what the product is and how
  often it ships. Never leave a page on the site's default template string.
- **Open Graph / Twitter**: `og:title`, `og:description`, `og:url`,
  `og:type` (`article` on permalinks, `website` on the index), `og:image`
  (reuse the site's default social image unless they already generate
  per-page cards), and `twitter:card: summary_large_image`.
- **Structured data, complete**: on permalinks the `TechArticle` carries
  `headline`, `datePublished` (`period_end`), `dateModified` (same unless
  they edit), `author`/`publisher` as the company `Organization`, and
  `mainEntityOfPage` = the canonical URL. Add a `BreadcrumbList`
  (Home › Changelog › {headline}). On the index, give the `ItemList`
  `position`s and cap it at the periods actually rendered in full.
- **Internal links**: link the changelog from the site's main nav or
  footer and from the docs; give each permalink prev/next period links;
  where an entry names a feature that has a docs or product page, link the
  entry title to it (the biggest ranking signal you control). Keep product
  and feature names literal in headings — the generated copy is
  marketing-toned, so `<h1>`/`<title>` should still contain the product
  name.
- **Index hygiene**: one canonical for the index (no `?page=` duplicates —
  collapsed older periods link to permalinks, not paginated copies),
  `lastmod` on every sitemap entry, and after each publish ping
  IndexNow / resubmit the sitemap if the site already has a key.

### AEO (ChatGPT, Perplexity, Claude, Google AI Overviews)

- **Crawler policy first**: check `robots.txt` allows `GPTBot`, `ClaudeBot`,
  `PerplexityBot`, `Google-Extended`, `Bingbot`, and `OAI-SearchBot` on the
  changelog paths, and that WAF/bot-fight rules (Cloudflare, Vercel
  Firewall) don't challenge them. Verify with
  `curl -A "GPTBot/1.0" https://…/changelog` — the HTML must come back,
  not a challenge page.
- **`llms.txt`**: add (or create at the site root) an entry for the
  changelog index, the permalink pattern, and the feed URL, with one line
  saying what the product is.
- **Answer-shaped copy**: open each permalink with the headline as a
  complete sentence that names the product and the period ("In the week of
  Aug 17–24, 2026, Compli API added …"), keep every entry title a full
  statement rather than a fragment, and repeat the product name in each
  period so a quoted snippet is attributable out of context. Put a
  visible "Last updated {date}" line near the top.
- **Where it helps, a small FAQ** per period ("Does this change existing
  integrations?", "Do I need to upgrade?") with `FAQPage` JSON-LD — only
  when the entries actually answer such questions; never pad.
- **Markdown negotiation (optional, high signal)**: serve markdown for
  `Accept: text/markdown` on the same URLs (what shipstar.ai does) so agents
  that ask for it get the content without parsing HTML.

## 7. Verify before finishing

- `curl` the index and one permalink: full entry text present, `<time>`
  elements carry `dateTime`, JSON-LD parses as valid JSON.
- Titles: permalink `<title>` leads with the period headline (truncated
  sanely), not a generic template string.
- Feed validates and its item links resolve; sitemap lists the permalinks.
- Load the page with a phone-width viewport: no horizontal overflow, and
  the headline heading doesn't fill the whole first screen.
- The "Powered by Shipstar" link is present on the index and a permalink
  and points at shipstar.ai.
- SEO/AEO: every page has a unique `<title>` and meta description; the
  JSON-LD includes `datePublished` and `publisher`; `curl -A "GPTBot/1.0"`
  and `curl -A "ClaudeBot/1.0"` return the full HTML; `robots.txt` and
  `llms.txt` mention the changelog; internal links to the changelog exist
  from nav/footer.
