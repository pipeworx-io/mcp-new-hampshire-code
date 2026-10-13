# @pipeworx/new-hampshire-code

New Hampshire RSA (Revised Statutes Annotated) — New Hampshire state
statutes — full section text by citation, and topic/keyword search over
official catchlines.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1784+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `nh_statute(citation)` — full current text of one RSA section (e.g.
  `"630:1"`, capital murder), with the official catchline, chapter/title
  context, and the amendment history taken verbatim from the section's own
  "Source." line (e.g. `"Source. 1971, 518:1. 1974, 34:1. ... 2019, 42:1,
  eff. May 30, 2019."`), split out into `history`.
- `nh_search(query, limit?)` — topic/keyword search over the General
  Court's own official section and chapter catchlines across all 64 titles
  (e.g. `"landlord"`, `"concealed weapon permit"`). Not full-text search —
  New Hampshire does not publish one.

## Auth

Keyless.

## Data sources

All under `https://gc.nh.gov/rsa/html/` — the canonical host; the legacy
`www.gencourt.state.nh.us` 301-redirects every path there (verified live
2026-10-07).

- `nhtoc.htm` — root Table of Contents: every TITLE (roman numeral I–LXIV)
  and its official name.
- `NHTOC/NHTOC-{ROMAN}.htm` — one page per title: every CHAPTER in it, with
  its official name.
- `NHTOC/NHTOC-{ROMAN}-{chapter}.htm` — one page per chapter: every SECTION
  in it, as `"Section: {chapter}:{section} {catchline}."`.
- `{ROMAN}/{chapter}/{chapter}-{section}.htm` — the live section page.
  `nh_statute` fetches this one, live, per request — never baked.

`scripts/bake-index.mjs` walks the first three (root + every title + every
chapter) ONCE, offline, into `src/nh-index-data.ts` — citation + official
catchline + chapter/title context, never the statutory text itself.

## Capabilities — three available, one is not

New Hampshire's own publication covers:

- **Citation lookup** — `nh_statute`.
- **Topic/keyword search** — `nh_search`, over catchlines only (see below).
- **Amendments/enactment history** — AVAILABLE. Every in-force section page
  carries a "Source." line inside its `sourcenote` tag, listing the
  session laws that created or amended it. `nh_statute` returns it verbatim
  (including the "Source." label) in `history`.
- **Historical version** (the text as it read before an amendment) — NOT
  AVAILABLE. Checked live: neither the root Table of Contents nor any
  section page links to a per-year archive or a "view as of" date picker.
  gc.nh.gov publishes the CURRENT online RSA only. The "Source." line says
  WHEN and by which session law a section was amended; it does not return
  what the text read before that amendment.

## Source and shape — read this before touching the parser

The August state-law survey (`docs/state-law-probe.md`) flagged New
Hampshire as needing "a ROMAN-NUMERAL title derived from the chapter," and
that every index URL it guessed returned the same generic 18.8 KB site page.
Both observations were correct as far as they went: NH's chapter numbers
are **not** partitioned by title through any arithmetic the way Idaho's
`chapter = floor(section / 100)` is, so there is no formula to derive a
title from a bare chapter number. What the survey had not found is that
gc.nh.gov's TOC is **three levels deep**, not one, and every level is a
small, static, keyless HTML page (see "Data sources" above) — so the
chapter → title-roman map, and every section's official catchline, both
come from walking all three levels once. This is the three-level version of
the two-level bake `mcps/idaho-code` does (title listing → chapter listing),
extended by one level because NH's chapter numbers give no shortcut at all.

**Citation → URL is then a direct rewrite, no guessing.** Split the
citation on its FIRST colon (`"632-A:1"` → chapter `"632-A"`, section
`"1"`) and the live page is
`https://gc.nh.gov/rsa/html/{titleRoman}/{chapter}/{chapter}-{section}.htm`
(`:` → `-`). Verified live across an ordinary chapter (630), a
hyphen-lettered chapter (632-A), and a hyphen-lettered section (630:1-a).
gc.nh.gov's IIS server is case-insensitive on this path (`630-1-A.htm` and
`630-1-a.htm` both 200), so the pack never needs to case-normalize what it
fetches — only what it matches a caller's typed citation against.

**Section page markup is simple and consistent** — every page carries the
statute body inside a bare `codesect` tag (lines separated by a plain `br`,
no nested tags observed across samples spanning multiple titles) and the
amendment history inside a `sourcenote` tag. Both are custom to this site,
not standard HTML, which is exactly why they are reliable anchors — no
WordPress/CMS chrome to strip around them.

**gc.nh.gov IS Cloudflare-egress-blocked — a different shape than Oklahoma's
or Utah's, confirmed live and requiring the same relay fix.** A laptop curl
reaches every URL shape fine, but a live Cloudflare Worker's egress gets HTTP
520 ("error code: 520", 16 bytes, Cloudflare's own generic edge-error body)
on EVERY ONE of 5 distinct URLs (root TOC, title TOC, chapter TOC, a real
section page, a nonexistent section) — reproduced TWICE with a throwaway
`wrangler dev --remote` Worker on the prod account, 2026-10-07:

```
                                                     laptop curl       wrangler dev --remote (CF edge, prod account), x2 runs
rsa/html/nhtoc.htm                 (root TOC)        200               520 / 16B "error code: 520"  (both runs)
rsa/html/NHTOC/NHTOC-LXII.htm       (title TOC)       200               520 / 16B "error code: 520"  (both runs)
rsa/html/NHTOC/NHTOC-LXII-630.htm   (chapter TOC)     200               520 / 16B "error code: 520"  (both runs)
rsa/html/LXII/630/630-1.htm         (real section)    200               520 / 16B "error code: 520"  (both runs)
rsa/html/LXII/630/630-999.htm       (bad section)     404               520 / 16B "error code: 520"  (both runs)
```

gc.nh.gov's own response headers (`Server: Microsoft-IIS/10.0`, `X-Powered-By:
ASP.NET`, no `cf-ray` or `Server: cloudflare`) show the origin itself is not
fronted by Cloudflare — so this is NH's state network refusing Cloudflare's
IP ranges at the connection level, not an application-layer challenge (unlike
oscn.net's Turnstile 2xx) and not a TLS chain gap (unlike le.utah.gov's 526).
Cloudflare's edge never gets an origin response at all to classify. Every
fetch in `src/index.ts` therefore routes through the Supabase egress relay
(`supabase/functions/egress-proxy`, `gc.nh.gov` added to its `ALLOWED_HOSTS`)
when the gateway injects `_proxyUrl`/`_proxyToken` for this slug — same
mechanism as `mcps/oklahoma-code` and `mcps/utah-code`, no `PINNED_CA` entry
needed since this is not a TLS issue. Absent the relay (local dev, this
pack's own laptop smoke test) it fetches direct.

**UNLIKE OKLAHOMA** (the pack that first forced this edge-probe rule),
gc.nh.gov gives an HONEST, distinct 404 (an IIS "File or directory not
found" template) for a nonexistent section or chapter path, confirmed to
differ in both status and body from a real page — no 200-with-page-furniture
trap observed; the ONLY transport problem here is the Cloudflare-specific
520 above, fixed by the relay. A **repealed** section is a different, legitimate case: its
page still 200s, but with an EMPTY `codesect` tag and an empty `sourcenote`
tag, and its own catchline (baked from the TOC, e.g. `"631:5 Repealed by 1992,
257:22, II, eff. Jan. 1, 1993."`) says so in plain text. `nh_statute`
distinguishes this from a genuine parse failure by checking the baked
catchline for a `Repealed|Redesignated|Transferred|Renumbered|Omitted`
prefix before accepting an empty body — an empty body on anything else is
treated as a loud parse failure, never a silent `found: false`.

**One recurring numeric entity:** `&#150;`, a decorative en dash after every
section's catchline. It is written as a Windows-1252 codepoint (150 = en
dash) used as if it were a Unicode code point — literally a control
character (U+0096) if decoded naively — the same remap browsers silently
apply. The pack's local entity decoder special-cases it.

**Refresh path:** `node scripts/bake-index.mjs > src/nh-index-data.ts`
whenever New Hampshire's RSA is recodified (session laws are folded in
several times a year). Nothing else in the pack needs to change —
`nh_statute` and `nh_search` both read whatever this file currently
exports.

## Known gaps

- `nh_search` matches official catchlines (section and chapter names), not
  full statutory prose — New Hampshire does not publish full-text search,
  and this pack does not claim to either. It says so in both the tool
  description and every "no match" response.
- A citation present in the baked index that the LIVE site 404s on (e.g. a
  section renumbered or repealed since the index was last baked) returns a
  distinctly-labelled `reason: "index_live_mismatch"`, never a plain
  `found: false` with no explanation.
- A non-404 transport failure or an unparseable 200 (a missing `codesect`
  or `sourcenote` tag, or an empty body on a non-disposition catchline) throws a
  loud error naming the upstream status/URL — never silently reshaped into
  `found: false`. See `src/index.test.ts` for the contract tests.

- This pack is dark on the live gateway until the Supabase `egress-proxy`
  function is REDEPLOYED with the `gc.nh.gov` entry this commit adds to its
  `ALLOWED_HOSTS` — deploying that function is `workflow_dispatch`-only
  (`.github/workflows/deploy-edge-function.yml`), never automatic on push,
  same as every other entry in that allow-list. Until it is deployed (or a
  gateway build predates the `_proxyUrl` injection for this slug),
  `nh_statute` throws a loud upstream error on every call rather than
  silently answering `found: false` — see the "Source and shape" section
  above and `src/index.test.ts`.

## Pre-deploy evidence

See `smoke.json` for the arguments smoke-tested and the pack's PR/close note
for the raw payloads and the edge probe (`wrangler dev --remote` against the
exact `gc.nh.gov` URLs this pack uses, run on the prod account per the
standing rule for any new upstream host) — see "Source and shape" above for
the table.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "new-hampshire-code": {
      "url": "https://gateway.pipeworx.io/new-hampshire-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/new-hampshire-code/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1784+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/nh_statute \
  -H 'Content-Type: application/json' \
  -d '{"citation":"630:1"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/nh_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "new-hampshire-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-new-hampshire-code"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-new-hampshire-code
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about New Hampshire Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
