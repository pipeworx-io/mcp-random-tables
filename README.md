# @pipeworx/random-tables

Pipeworx original composable random tables for tabletop and narrative play —
encounters, loot, weather, complications, NPC traits, plot hooks, tavern
scenes, rumours and settlement quirks — with nested `{{table:<id>}}`
references resolved recursively, plus caller-supplied custom tables. Keyless,
no upstream, no COGS.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `roll_table(table, count?, seed?, session_id?)` — roll 1-20 entries from a
  built-in table. Returns both the raw entry text and the fully resolved text
  (nested `{{table:<id>}}` references expanded). Reproducible via `seed`.
- `roll_custom_table(entries, count?, seed?)` — roll from a table YOU supply
  (`[{text, weight?}]`); entries may reference built-in tables the same way.
- `list_random_tables(genre?)` — every built-in table's id, title, genre and
  entry count, optionally filtered to `fantasy` | `scifi` | `modern`.
- `get_random_table(table)` — one built-in table in full, unresolved (raw
  entry text, so any `{{table:<id>}}` references are visible as written).

## Built-in tables

`encounters_fantasy`, `encounters_scifi`, `encounters_modern`, `loot`,
`weather`, `complication`, `npc_personality`, `npc_secret`, `npc_occupation`,
`plot_hook`, `tavern`, `rumour`, `settlement_quirk`, `npc_full` (a composed
sentence that nests `npc_personality` + `npc_occupation` + `npc_secret` in one
roll — the worked example of nested references).

Genre-tagged tables (`encounters_*`, `loot`, `tavern`) are fantasy/scifi/modern
flavored; the rest are genre-agnostic and apply under any `list_random_tables`
genre filter.

## Nested references

An entry's text may embed `{{table:<id>}}`, which is resolved by rolling a
weighted entry from that table and substituting it in, recursively, up to 6
levels of nesting. A reference to an unknown table id, or a cycle (a table
that ends up referencing itself through a chain of other references), is a
loud error naming the offending id or the full chain — never a silent
truncation or an empty substitution. `roll_custom_table`'s caller-supplied
entries may reference built-in tables the same way, but not other entries in
the same custom table (a custom table has no id of its own to reference).

## Randomness and reproducibility

Every rolling tool takes an optional `seed`; the seed actually used is always
returned (in `meta.seed`), whether you supplied one or not. The same seed and
arguments reproduce the same draw. These are content generators, not
divination — there is no separate "meaning" layer to keep apart from
interpretation; the table text *is* the result, and how you use it at the
table is up to you.

## Sessions

`session_id` on `roll_table` draws without replacement **across calls** —
useful when you don't want the same loot item or the same rumour to repeat
while a session is live. It is **hosted-only**, backed by the gateway's
shared KV (`injectCacheKv: true`); the npm standalone package has no KV, so a
call that passes `session_id` there gets back `session_supported: false` and
a plain seeded roll instead (every tool still works without `session_id`
regardless of environment).

There is no separate "open session" tool. Pass any string as `session_id` on
your first call for a table — the response tells you plainly whether it
continued an existing session or had to open a new one, and gives you the
real `session_id` to reuse on your next call. Omit `session_id` entirely for
an ordinary independent weighted roll (the common case; no KV needed).
`roll_custom_table` has no session support — sessions only track built-in
tables, which have a stable id and size to key a session on.

## Auth

Keyless.

## Data sources

Pipeworx original content. No upstream API, no third-party data. Every table
and every entry is written for this pack — none of it is copied, paraphrased
or adapted from any commercial tabletop RPG product, sourcebook or
supplement.

## What else to know

- Zero-row answers are loud errors, not quiet empty lists: an unknown table
  id, an unknown genre filter, an unknown or cyclic nested reference, or an
  empty/invalid `entries` array for `roll_custom_table` all throw a specific
  error naming the valid set, rather than returning `[]` or `{}`.
- `roll_table`'s default `count` is 1; the cap is 20 for both rolling tools.
- `roll_custom_table` accepts up to 500 caller-supplied entries.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "random-tables": {
      "url": "https://gateway.pipeworx.io/random-tables/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/random-tables/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/roll_table \
  -H 'Content-Type: application/json' \
  -d '{"table":"npc_full","count":2,"seed":"pw-smoke-1"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/roll_table`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-random-tables` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/random-tables/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "random-tables": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-random-tables"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Random Tables data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
