# @pipeworx/arkansas-code

Arkansas Code of 1987 — section lookup by citation, and topic/keyword search
over official catchlines. Fleet #2881, one of the four remaining states with
no `<state>-code` pack (Mississippi now has one too — fleet #2883). Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1763+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ar_statute(citation)` — the text of one Ark. Code Ann. section by
  citation (`"5-10-101"`, capital murder; `"4-70-101"`; `"Ark. Code Ann. §
  5-10-101"`). Returns the catchline, the operative statutory text, the
  `History.` line (the Acts that enacted/amended it), and other legislative
  notes when present. Case-law annotations are deliberately excluded — they
  can run to hundreds of KB per heavily-litigated section.
- `ar_search(query, limit?)` — topic/keyword search over official section
  catchlines across all 28 titles (e.g. `"capital murder"`, `"security
  deposit"`). Not full-text search — see "Known gaps".

## Auth

Keyless. No signup, no login, on either side of the call.

## Data source, and why this isn't a Lexis call

The official Arkansas Code is published through LexisNexis's "free public
access" portal (`lexisnexis.com/hottopics/arcode` -> `advance.lexis.com`) —
verified live (2026-10-09): the landing page needs real browser JS to pass a
cookie/fingerprint check, and granting it cookies anyway just 302s to an
authentication endpoint (`/lnaccess/mip/authn?...`) that answers
`HTTP 405` with `x-amzn-waf-action: captcha`. `codes.findlaw.com/ar/` and
`law.justia.com/codes/arkansas/` both serve their actual title/section pages
behind a Cloudflare bot challenge (`cf-mitigated: challenge`, HTTP 403).

- <https://unicourt.github.io/cic-code-ar/> — a GitHub Pages copy of the
  Arkansas Code, built by UniCourt on behalf of Public.Resource.Org from the
  Code's own officially-released RTF volumes
  (`archive.org/download/gov.ar.code`), "in the public domain, NO RIGHTS
  RESERVED" per that repo's own README. One title per HTML page
  (`gov.ar.code.title.{NN}.html`, zero-padded to 2 digits, 28 titles),
  fetched LIVE per `ar_statute` call — this pack proxies it, it does not
  ship a copy of the Code in the repo
  (`mirror-needs-grant-proxy-does-not` in the Pipeworx CLAUDE.md licence
  section).

  The underlying statutory TEXT is public law regardless of what any
  publisher's licence says about its own rendering — enacted law carries no
  copyright (*Banks v. Manchester*, 128 U.S. 244 (1888); *Veeck v. SBCCI*,
  293 F.3d 791 (5th Cir. 2002) (en banc)).

**THE HONEST LIMIT.** This source carries three releases — r76 (2020-04-20),
r77 (2020-08-18), r78 (2020-11-06), confirmed live via the GitHub contents
API (2026-10-09) — and this pack uses **r78, the newest**. It is a static
snapshot, not a continuously updated feed: Arkansas amendments enacted after
2020-11-06 are **not reflected**. Every response carries `current_through`
and `data_as_of: "2020-11-06"` rather than defaulting to "now". For anything
enacted since, verify against the official LexisNexis-hosted Code or the
Arkansas Code Revision Commission.

(This pack originally shipped against a 2012 Public.Resource.Org XML scan —
the only Worker-reachable text known at the time. A same-day follow-up
pointed at this newer UniCourt mirror, 8+ years fresher; the switch is
confirmed live: Ark. Code Ann. § 5-10-101's `History.` line now runs through
"2019, No. 910, § 666" versus 2011 in the old source.)

## Known gaps

- `ar_search` matches official catchlines only, not full statutory text —
  fetching all 28 multi-MB title pages per query is not viable. A topic
  discussed inside a section's body but absent from its catchline will not
  be found.
- Every response is explicitly dated to Release 78 (2020-11-06). A pre-r78
  (or pre-2012) version of a section's text is not published anywhere free
  that was found.

## Refreshing the search index

`ar_search` reads a baked catchline index (`src/ar-index-data.ts`, 37,369
rows across 28 titles), injected from KV via
`workers/gateway/src/pack-baked-indexes.json` rather than bundled (fleet
#2754: a value-imported multi-MB table loads into every gateway isolate
whether or not it ever serves this pack). Rebuild it with:

```bash
node scripts/bake-index.mjs > src/ar-index-data.ts
```

Re-run only if `unicourt.github.io/cic-code-ar` publishes a release newer
than 78.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "arkansas-code": {
      "url": "https://gateway.pipeworx.io/arkansas-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/arkansas-code/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1763+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ar_statute \
  -H 'Content-Type: application/json' \
  -d '{"citation":"5-10-101"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ar_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "arkansas-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-arkansas-code"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-arkansas-code
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Arkansas Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
