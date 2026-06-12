---
name: publish-with-pagelive
description: >
  Use the Pagelive MCP connector to publish an HTML page (proposal, deck, report, demo) as a
  private, branded, view-tracked link — then update it in place, read who opened it, and collect
  reply-form submissions. Trigger when the user says "publish this", "make a shareable link",
  "put this on a page", "update that page", "who opened my deck", or "did anyone reply". Covers
  every tool, its parameters and return shape, the free/Pro plan limits, and the common failure
  modes so the full connector can be driven correctly.
---

# Publish with Pagelive

Pagelive is a hosted MCP connector (`io.pagelive/pagelive`, remote at
`https://app.pagelive.io/api/mcp`, OAuth or API-key auth) that turns HTML into a **private,
shareable, view-tracked link**. Reach for it whenever the user wants to **send** something they
built — not just see it in the conversation.

Pages are served from an isolated content plane (`pagelive.site`, or a connected custom domain),
are **`noindex` and private by default**, can be **password-gated**, carry a small "Made with
Pagelive" badge, and keep a **version history** (every publish/update is a new version; the link
never changes).

## When to use
- "Publish this / make a shareable link / host this page." → `publish_page`
- "Update that page / change X but keep the same link." → `update_page`
- "Who opened it? How many views?" → `get_page_stats`
- "What have I published? Which got views?" → `list_pages`
- "Did anyone reply to the form?" → `list_form_submissions`
- Publishing into a team or on a custom domain → `list_workspaces` / `list_domains` first.

## Core workflow
1. **Publish** with `publish_page` (full HTML). You get back a live `url` + `slug`. Give the user
   the URL plainly — it's stable.
2. **Revise** with `update_page` — prefer `edits` (send only the changed regions), not a full
   resend. The URL never changes.
3. **Report** results with `get_page_stats` (opens) and `list_form_submissions` (replies).
   Only report what the tools actually return — never invent numbers.

## Tool reference

### `publish_page` — create or replace a page
| Param | Required | Notes |
|---|---|---|
| `html` | ✅ | A **complete HTML document** (`<!doctype html>…`). This is the page. |
| `title` | – | Dashboard label only (≤200 chars). Not shown on the page. |
| `password` | – | Gates the page behind a password. **Free = 1 protected page**; paid = unlimited. |
| `slug` | – | Two meanings — see below. |
| `workspace` | – | Team id/slug from `list_workspaces`. Omit = your **personal** workspace. |
| `domain` | – | A connected, **active** custom-domain hostname from `list_domains`. **Paid only**, and the domain must belong to the chosen workspace. Omit = `pagelive.site`. |

**`slug` has two behaviours** (slugs are unique *per host*):
- **Re-publish / replace:** pass the slug of a page you already own on that host → its content is
  replaced as a new version. Returns `replaced: true`. Works on **Free** (this is how "update the
  same link" stays available to everyone).
- **Custom slug (new page):** only on **custom domains**. `pagelive.site` is *always*
  auto-generated, so a new `slug` there is rejected for **every** plan — omit it. To choose a
  slug, pass a connected `domain` (paid) together with the `slug`.

Returns: `{ slug, url, workspace, replaced? }`. Default host is `pagelive.site`, personal
workspace. Pages publish as `live`, `noindex`, badge on.

### `update_page` — edit an existing page
| Param | Required | Notes |
|---|---|---|
| `slug` | ✅ | The page to edit. Must be unambiguous across your hosts/workspaces. |
| `edits` | preferred | Array of `{ old_str, new_str }` find-and-replace ops. **Send only the changed regions.** |
| `html` | alt | A full document that replaces the whole page. Use only for a complete rewrite. |

**The edit contract (important):** each `old_str` must appear **exactly once** in the current
page. If it's missing or appears more than once, the **entire** update is rejected and the page is
left untouched (never corrupted). To fix: re-read the page (its current HTML) and copy exact text,
adding surrounding context to disambiguate. Returns `{ slug, url, updated: true, mode: 'patched' | 'replaced' }`.

### `get_page_stats` — analytics for one page
Param: `slug` (required). Returns:
`{ slug, url, totalViews, recordedEvents, uniqueViewers, topCountries: [{ country, n }] }`
- `totalViews` — lifetime view counter for the page.
- `recordedEvents` — detailed, **bot-filtered** view events.
- `uniqueViewers` — distinct visitors (by hashed IP).
- `topCountries` — up to 5, busiest first.
> Note: this tool does **not** return per-visitor dwell time — don't claim dwell from it. Report
> the four fields above. (Dwell/engaged-time lives in the dashboard, not this endpoint.)

### `list_pages` — everything you can access
No params. Returns up to 200 pages, newest first:
`{ pages: [{ slug, title, viewCount, status, workspace, url }] }`. Includes your personal pages
plus pages in teams you belong to. Good for "which pages got views" (read `viewCount`).

### `list_form_submissions` — read reply-form responses
Params: `slug` (required), `limit` (default 50, max 200). Returns:
`{ slug, count, replies: [{ submittedAt, country, fields }], tip? }` — newest first. `fields` is
the submitted form data as key/value pairs. If `count` is 0, `tip` explains how to add a form.

### `list_workspaces` — where you can publish
No params. Returns `{ workspaces: [{ id, kind, plan, … }] }` — always a `personal` entry plus any
`team` workspaces (with `role`). Use an `id`/`slug` as the `workspace` arg to `publish_page`.

### `list_domains` — connected custom domains
Optional `workspace` filter. Returns `{ domains: [{ hostname, workspace }] }` (active domains
only). Use a `hostname` as the `domain` arg to `publish_page`.

## Recipes
- **Private page:** `publish_page { html }` → share the `url`. (Private/noindex by default.)
- **Password-gate it:** `publish_page { html, password }`. On Free, only the first protected page.
- **Update in place:** `update_page { slug, edits: [{ old_str: "Q2", new_str: "Q3" }] }` — the
  link stays the same. Never republish a whole new page when the user means "update".
- **Publish into a team:** `list_workspaces` → `publish_page { html, workspace: "<id>" }`.
- **Publish on a custom domain (paid):** `list_domains` → `publish_page { html, domain: "deck.acme.com" }`.
- **Collect replies:** include a plain `<form data-pagelive>` with named inputs in the HTML — no
  `action`/endpoint needed; Pagelive wires it on serve, emails the owner, and stores submissions.
  Then `list_form_submissions { slug }`. Example:
  ```html
  <form data-pagelive>
    <input name="name" placeholder="Your name">
    <input name="email" type="email" placeholder="Email">
    <textarea name="message" placeholder="Reply"></textarea>
    <button type="submit">Send</button>
  </form>
  ```
- **See who opened it:** `get_page_stats { slug }`, or `list_pages` for a portfolio view.

## Authoring good pages
- Pass a **complete, self-contained HTML document** — inline the CSS, no build step. It's served
  verbatim. External assets (fonts, images) load fine via absolute URLs.
- Give forms named inputs (`name="…"`) so submissions come back as labelled `fields`.
- The "Made with Pagelive" badge stays on for MCP-published pages (toggling it is a dashboard-only
  control — don't promise it can be removed via MCP).

## Plans & limits
- **Free:** up to **10 pages per workspace**, **1 password-protected page**, auto-generated slugs
  on `pagelive.site`. No custom slugs, no custom domains.
- **Pro / Team:** unlimited pages & passwords, **custom domains** (which unlock **custom slugs** —
  slugs are still auto-generated on `pagelive.site`), team workspaces.
- Publishing is rate-limited; a burst returns a "publishing too fast" error — back off and retry.

## Failure modes & how to handle them
- **"Custom slugs are only available on custom domains"** — you passed a *new* slug for
  `pagelive.site`. Omit `slug` (links there are auto-generated), pass the slug of a page you
  already own to replace it, or publish to a connected `domain` to choose the slug.
- **"Free plan keeps up to 10 pages"** — workspace cap hit. Update an existing page, or upgrade.
- **"old_str not found / appears multiple times"** — your `edits` didn't match exactly once.
  Re-read the page and copy exact text with more context. The page was not modified.
- **"slug … is ambiguous"** — the same slug exists on more than one of your hosts/workspaces.
  Disambiguate (the page may live on a custom domain).
- **"flagged as potential phishing"** — content was blocked by the safety scan; revise it.
- **"No active custom domain"** — connect the domain in the dashboard first, or omit `domain`.

## Don'ts
- Don't fabricate analytics or replies — only report what `get_page_stats` /
  `list_form_submissions` return.
- Don't resend the whole document to `update_page` when a few `edits` will do.
- Don't create a new page when the user means "update" — reuse the slug.
