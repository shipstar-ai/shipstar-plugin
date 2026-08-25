---
name: setup-shipstar
description: >
  Connect this agent to Shipstar and verify the project is ready for
  automated release marketing. Use when the user wants to set up Shipstar,
  connect the Shipstar MCP server, or when another Shipstar skill fails
  because the MCP connection or project setup is incomplete.
---

# Set up Shipstar

Goal: a working MCP connection and a project that can actually generate and
deliver content. Fix what you can; route the user to the dashboard for the
OAuth steps you can't do.

## 1. Connect the MCP server

If Shipstar MCP tools are already available, skip to step 2.

Otherwise the user needs their API token and backend URL from the Shipstar
dashboard (Dashboard → Agents shows a copy-paste command). The generic form
for Claude Code:

```bash
claude mcp add --transport http shipstar <BACKEND_URL>/mcp \
  --header "Authorization: Bearer <API_TOKEN>"
```

The token is project-scoped — every tool call operates on that project. For
other MCP clients, it's a streamable-HTTP server at `<BACKEND_URL>/mcp` with
the same Authorization header; Shipstar is also an OAuth 2.1 provider, so
clients that support MCP OAuth can connect without a static token.

## 2. Verify the connection

Call `get_project_context`. If it errors, the token is wrong or expired —
have the user re-copy it from the dashboard. If it succeeds, tell the user
which project you're connected to by name.

## 3. Audit readiness

From the same `get_project_context` result, check and report:

| Check | Ready when | If not |
|---|---|---|
| GitHub source | `github_connected` true, `github_suspended` false, `tracked_repositories` non-empty | Dashboard → Sources → Install GitHub App (or Link existing installation), pick repos. If `github_suspended` is true, an org admin must unsuspend the app on GitHub |
| Destinations | the channels the user wants show `connected: true` | Dashboard → Destinations (Slack, Intercom, X are OAuth flows) |
| Mailing lists | at least one list with `active_recipients > 0`, if they want release emails | Dashboard → Destinations → Email |
| Content guidelines | `content_guidelines.audience` (technical depth) / `.instructions` reflect how technical the content should read and anything to leave out (optional — both `null` means built-in defaults) | Dashboard → Settings → Content guidelines; or pass `audience` / `instructions` per generation call |

You cannot connect sources or destinations over MCP — the GitHub App install
and the destination OAuth flows happen in the dashboard (and on GitHub). Be
direct about which items need a dashboard visit.

## 4. Smoke test (optional but recommended)

With the user's go-ahead, run `generate_blog_post_ideas` — it's synchronous,
cheap, and exercises the GitHub source end to end. If it returns ideas, the
pipeline works. If it errors about GitHub or tracked repositories, step 3
wasn't actually complete.

Finish by telling the user what they can now do: "announce a release", "write
a blog post about what shipped", "email release notes to your users".
