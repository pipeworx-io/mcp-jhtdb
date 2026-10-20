# @pipeworx/jhtdb

Point queries (velocity, pressure, velocity gradient) on Johns Hopkins
Turbulence Databases (JHTDB) reference DNS/LES datasets, plus a catalog of
every dataset's spatial domain, grid resolution, time range, and available
physical fields.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1851+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `jhtdb_datasets(...)` — the full catalog of JHTDB datasets (forced isotropic
  turbulence, channel flow, MHD turbulence, transitional boundary layer,
  atmospheric boundary layer and wind-farm LES, and more): spatial domain,
  grid resolution, time range, and available fields for each.
- `jhtdb_point_query(...)` — interpolated velocity, pressure, or
  velocity-gradient values at up to 10 points and one time in a named
  dataset.

## Auth

Keyless from the caller's point of view. This pack uses JHTDB's own public
testing authorization token internally (documented by the publisher as valid
for small queries), and caps every call at 10 points — well under the
publisher's documented small-query allowance — so Pipeworx traffic does not
load that shared testing quota. There is no BYO-key override: a full
(non-testing) JHTDB token is an institutional request against the publisher
and is out of scope for this pack unless demand for larger queries shows up.

## Data sources

- Johns Hopkins University (Institute for Data Intensive Engineering and
  Science), Turbulence Databases (JHTDB) — direct numerical simulation (DNS)
  and large-eddy simulation (LES) of canonical turbulent flows (forced
  isotropic turbulence at several grid resolutions, channel flow, MHD
  turbulence, a homogeneous buoyancy-driven flow, a transitional boundary
  layer, rotating stratified turbulence, stably stratified atmospheric
  boundary layers, and wind-farm LES), queryable at arbitrary points and
  times via the publisher's own web service.

The publisher's query interface is SOAP 1.1, not REST/JSON — there is no
JSON point-query endpoint to fall back to. The SOAP envelope is built and
parsed directly in `src/index.ts` (regex-based XML matching, consistent with
how this codebase already handles SOAP/XML upstreams with no DOMParser
available in the Workers runtime — see `mcps/sdmx-structure`).

The dataset catalog (domain extents, grid resolution, time range, available
fields) is transcribed from the publisher's own machine-readable dataset
configuration, not derived by probing the service — it is 17 small records,
not a generated corpus, so it lives as a plain array in `src/index.ts` rather
than going through the KV baked-index path.

Values are returned in each dataset's native simulation units: dimensionless
DNS code units for the periodic-box datasets (isotropic turbulence, channel
flow, MHD turbulence), physical SI units (meters, seconds, Pascals) for the
atmospheric-boundary-layer and wind-farm LES datasets. `jhtdb_point_query`'s
response states which applies for the field just queried.

Dataset codes, enum values for spatial/temporal interpolation methods, and
which physical fields each dataset carries are all exactly as the publisher's
own API defines them — an interpolation-method typo (e.g. a plausible-looking
`FD4NoInt` instead of the real `None_Fd4`) fails loudly with the publisher's
own validation message rather than silently, which is how the correct enum
values here were confirmed.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "jhtdb": {
      "url": "https://gateway.pipeworx.io/jhtdb/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/jhtdb/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1851+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/jhtdb_datasets \
  -H 'Content-Type: application/json' \
  -d '{}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/jhtdb_datasets`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "jhtdb": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-jhtdb"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-jhtdb
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Jhtdb data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
