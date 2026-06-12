# Pagelive MCP connector

**Publish the pages you build with AI — straight from the conversation — as private, branded, view‑tracked links.**

Pagelive turns any HTML page (a proposal, deck, report, or demo) into a clean link you can
actually send. Publish it from Claude, get a link that's **private by default** (noindex,
optional password, served from an isolated content plane on Cloudflare), then **see who opened
it** — opens, unique viewers, dwell time, by day, country, referrer — bot‑filtered, with an
email the moment it's opened. Pages can carry a reply form whose submissions come back into the
conversation. Update anytime; the link never changes.

> This is the **connector manifest + docs** for Pagelive's hosted MCP server. The server is
> hosted by Pagelive (you don't run or install anything); this repo is the public home for the
> `server.json`, the tool reference, and example usage. Sign in with your Pagelive account —
> a free account works.

- **Website:** https://pagelive.io/mcp
- **Docs / quickstart:** https://pagelive.io/docs/quickstart-mcp
- **Connector URL (remote, OAuth):** `https://app.pagelive.io/api/mcp`

## Connect

The server is **remote** and uses **OAuth 2.0** — there's nothing to install. Add the URL to any
MCP‑capable client and sign in with your Pagelive account.

**Claude (web / desktop)** — Settings → Connectors → **Add custom connector** → paste
`https://app.pagelive.io/api/mcp` → sign in when prompted.

**Claude Code**
```bash
claude mcp add --transport http pagelive https://app.pagelive.io/api/mcp
```

**Any client that reads `server.json`** — point it at this repo's [`server.json`](./server.json),
or find Pagelive in the [MCP Registry](https://registry.modelcontextprotocol.io) under
`io.pagelive/pagelive`.

## Tools (7)

| Tool | What it does |
|---|---|
| `publish_page` | Publish an HTML page and get a live, shareable, view‑tracked URL. Pass an existing `slug` to replace a page instead of duplicating it. |
| `update_page` | Edit a page you've published — targeted text edits or a full‑HTML replace. The link never changes. |
| `list_pages` | List the pages you can access (personal + team) with their URLs, workspace, and view counts. |
| `get_page_stats` | View analytics for a page — opens, unique viewers, dwell, top countries. |
| `list_form_submissions` | Read the replies submitted through a page's reply form. |
| `list_domains` | List your connected custom domains (Pro/Team). |
| `list_workspaces` | List the workspaces you belong to (personal + teams). |

## Example prompts

1. "Publish this page with Pagelive and give me the link."
2. "Update my Q3 proposal page with this new version — keep the same link."
3. "How many people opened my deck this week, and how long did they read?"
4. "Did anyone reply to the proposal form? Show me the responses."
5. "List my published pages and tell me which got views yesterday."

## Live demos

- https://pagelive.site/p/vantage-series-a-deck
- https://pagelive.site/p/q2-performance-report
- https://pagelive.site/p/q3-growth-proposal

## Skill

This repo includes an optional [Agent Skill](./skills/publish-with-pagelive/SKILL.md) that teaches
the publish → share → track workflow, so an agent reaches for Pagelive at the right moment.

## Auth & privacy

OAuth 2.0 sign‑in (free account works); no API key juggling. Published pages are `noindex` by
default and can be password‑gated; analytics are bot‑filtered and contain no PII.

## License

[MIT](./LICENSE) — applies to the contents of this repo (manifest, docs, examples, skill). The
hosted Pagelive service and its server source are separate and proprietary.
