# Shipstar plugin for Claude Code

Ship features — Shipstar handles the marketing. This plugin connects Claude Code to
[Shipstar](https://shipstar.ai)'s hosted MCP server and bundles guided skills for the
full pipeline: generate changelogs, blog posts, knowledge-base articles, release-notes
emails, and social posts from your recent commits, then review, publish, and send them.

## Install

```
/plugin marketplace add shipstar-ai/shipstar-plugin
/plugin install shipstar@shipstar
```

On first tool use, Claude Code opens Shipstar's OAuth sign-in in your browser — no API
keys or configuration needed. Each connection is scoped to one Shipstar project, chosen
on the consent screen. You'll need a [Shipstar account](https://shipstar.ai) (free plan
available) with a connected GitHub repository.

Prefer a static token (CI, scripts)? Mint one in the dashboard (API Tokens page) and add
the server manually instead:

```
claude mcp add --transport http shipstar https://mcp.shipstar.ai/mcp --header "Authorization: Bearer YOUR_API_TOKEN"
```

## Skills

| Skill | What it does |
|---|---|
| `/shipstar:announce-release` | Generate a changelog + release-notes email from recent commits, review with you, then publish and email subscribers. |
| `/shipstar:setup-shipstar` | Connect the MCP server, verify the project, and audit sources, destinations, and mailing lists. |
| `/shipstar:write-blog-post` | Brainstorm angles from commits, generate a draft, revise together, and publish. |
| `/shipstar:email-your-users` | Send release notes to your mailing lists, with confirmation gates and unsubscribe handling. |
| `/shipstar:build-changelog-page` | Implement a server-rendered, agent-readable changelog on your own site — semantic HTML, permalinks, RSS, and structured data from your changelog API. |

All 23 MCP tools are also available directly — read tools are annotated read-only;
publishing and email sends only ever happen through explicit tool calls you approve.

## Links

- [MCP docs](https://docs.shipstar.ai/mcp/overview) · [OAuth details](https://docs.shipstar.ai/mcp/oauth)
- [Shipstar](https://shipstar.ai) · [Support](mailto:support@shipstar.ai)
