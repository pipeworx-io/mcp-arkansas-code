# @pipeworx/arkansas-code

Arkansas Code of 1987 — section lookup by citation, and topic/keyword search
over official catchlines. Fleet #2881, one of the four remaining states with
no `<state>-code` pack (Mississippi now has one too — fleet #2883). Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1784+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ar_statute(citation)` — the text of one Ark. Code Ann. section by
  citation (`"5-10-101"`, capital murder; `"4-70-101"`; `"Ark. Code Ann. §
  5-10-101"`). Returns the catchline, the operative statutory text, the
  `History.` line (the Acts that enacted/amended it), other legislative notes
  when present, AND a `later_acts` overlay — every act the Arkansas
  legislature's own index shows touching this exact citation since the
  2020-11-06 snapshot (fleet #2906, see "The overlay" below). Case-law
  annotations are deliberately excluded — they can run to hundreds of KB per
  heavily-litigated section.
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

## The overlay: later acts, never merged (fleet #2906)

Bruce kept `ar_statute` on the frozen 2020-11-06 snapshot ("we'll need to
figure something out") rather than chase a fully reconciled current code.
This overlay is the cheapest honest improvement on top of that decision —
design and measurements in `docs/current-statute-routes.md` (#2902) — and is
the first of three planned (Mississippi and Tennessee follow with their own
home-built indexes once this proves out).

**The rule: never rewrite the snapshot `text`. Only ever ADD a dated, linked
list beside it, explicitly labelled as not merged.** `ar_statute`'s response
carries:

- `later_acts` — one entry per act that touched this exact citation since
  the snapshot date: `act_number`, `session` (human-readable, e.g. "2025 -
  Regular Session, 2025"), `link` (the act's own PDF), `bill` (the measure
  number), `flags` (any subset of `EC`/`RP`/`NS`/`RN` — Emergency Clause /
  Repealed / New Section / Renumbered), `source: "arkleg"`, and
  `merged: false`. Empty (`[]`) means the index shows no later act for this
  citation across every session checked — not that the overlay failed to
  load (a missing/malformed KV index throws loudly instead, same as
  `ar_search`'s catchline index below).
- `sessions_checked` — every biennium/session the index covers, named even
  when `later_acts` is empty, so a caller can tell "nothing changed" from
  "we didn't check."
- `later_acts_data_as_of` — when the overlay was last baked.
- `date: null` on every act entry, **honestly** — the source index gives
  session + act number + PDF link, not a per-act enactment date (that line
  lives inside each act's own PDF, not in the index); guessing a date from
  the session's year would be worse than saying so.

**Source**: the Arkansas General Assembly's own "Code Sections Amended" tool
(`arkleg.state.ar.us/Acts/CodeSectionsAmended`) calls a keyless endpoint —
`/Acts/CodeSection?section=<title>&ddBienniumSession=<session>` — that
returns every citation touched in a biennium/session: the citation, the
amending Act's number and PDF link, EC/RP/NS/RN flags, and the bill number.
Confirmed live and from a throwaway `wrangler dev --remote` edge probe on the
prod Cloudflare account (2026-10-09/10): byte-identical to a laptop `curl`,
HTTP 200, no bot wall, no JS rendering required. We are calling the index the
state already maintains — the proxy shape (`mirror-needs-grant-proxy-does-
not` in root CLAUDE.md), same as `ar_statute`'s own live fetch above.

**A section not found in the Release-78 snapshot but present in the overlay**
is flagged specially: `ar_statute`'s `found: false` response carries
`later_acts_if_created` — if the legislature's index shows acts against a
citation that does not exist in the 2020 snapshot, that is a signal the
section may have been CREATED after the snapshot date, not necessarily a bad
citation.

**For Mississippi and Tennessee (next): reuse, don't re-derive.** Arkansas is
the cheap case because the legislature publishes the citation→act index
directly — Mississippi and Tennessee do not (per docs/current-statute-
routes.md), so their overlays need a home-built index crawled from signed-
bill text instead of one state-maintained endpoint. What DOES carry over
directly: the response shape (`later_acts`/`sessions_checked`/
`later_acts_merged_into_text: false`/the "never rewrite text" rule), the
baked-index + KV injection pattern (append new exports to the SAME
`src/<xx>-index-data.ts` file the catchline index already uses — one file per
pack slug in `pack-baked-indexes.json`, not a second registry entry), the
session-discovery-over-hardcoding trick (read the live `<select>` rather than
a fixed session list, so the index self-extends as new sessions are held),
and the `scripts/bake-later-acts.mjs` → `scripts/<xx>-later-acts-refresh.yml`
shape for the refresh workflow.

## Known gaps

- `ar_search` matches official catchlines only, not full statutory text —
  fetching all 28 multi-MB title pages per query is not viable. A topic
  discussed inside a section's body but absent from its catchline will not
  be found.
- Every response is explicitly dated to Release 78 (2020-11-06). A pre-r78
  (or pre-2012) version of a section's text is not published anywhere free
  that was found.

## Refreshing the baked indexes

`src/ar-index-data.ts` carries TWO independently-baked tables, both injected
from KV via `workers/gateway/src/pack-baked-indexes.json` as one combined
`_bakedIndex` object rather than bundled (fleet #2754: a value-imported
multi-MB table loads into every gateway isolate whether or not it ever
serves this pack):

- `AR_SECTIONS`/`AR_TITLES` — `ar_search`'s catchline index, 37,369 rows
  across 28 titles. Rebuild with `node scripts/bake-index.mjs >
  src/ar-index-data.ts` (OVERWRITES the file — see below). Re-run only if
  `unicourt.github.io/cic-code-ar` publishes a release newer than 78.
- `AR_LATER_ACTS`/`AR_LATER_ACTS_SESSIONS_CHECKED` — `ar_statute`'s overlay
  index (fleet #2906), ~8,900 rows across the 12 sessions held since
  2020-11-06. Rebuild with `node scripts/bake-later-acts.mjs >>
  src/ar-index-data.ts` (APPENDS). Re-run after every Arkansas legislative
  session adjourns and Acts are signed — see
  `.github/workflows/arkansas-later-acts-refresh.yml` for the scheduled
  version. Session discovery is automatic (reads the live `<select>` on
  `arkleg.state.ar.us/Acts/SearchByRange`), so a new session shows up with no
  code change.

**Full regen (both tables)**:

```bash
node scripts/bake-index.mjs > src/ar-index-data.ts
node scripts/bake-later-acts.mjs >> src/ar-index-data.ts
```

Run `node ../../scripts/check-baked-indexes.mjs` after either (from the repo
root) — it dynamically imports the regenerated file the same way the KV
uploader does, and refuses an empty exported array before it ever reaches
deploy.

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

Both URLs reach the same gateway and the same 1784+ data sources. The
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
