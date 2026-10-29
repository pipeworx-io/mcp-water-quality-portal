# @pipeworx/water-quality-portal

US ambient water-chemistry monitoring — station metadata and individual sample
results from the joint USGS / EPA / NWQMC Water Quality Portal (NWIS, STORET
and STEWARDS in one query surface).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1857+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `wqp_search_sites(statecode?, countycode?, huc?, bbox?, siteType?, characteristicName?, organization?, limit?)` — monitoring stations with coordinates, site type and how many results each holds. At least one filter is required.
- `wqp_get_results(siteid?, characteristicName?, statecode?, countycode?, startDate?, endDate?, sampleMedia?, limit?)` — individual sample values with unit, method, depth and date.
- `wqp_results_near_point(latitude, longitude, within_miles?, characteristicName?, startDate?, endDate?, limit?)` — the same results inside a radius.
- `wqp_list_characteristics(domain?, text?, limit?)` — the controlled-vocabulary spellings the portal demands ("Nitrate", not "nitrate").

## Auth

Keyless.

## Data sources

- `https://www.waterqualitydata.us/data/Station/search` — station metadata (GeoJSON).
- `https://www.waterqualitydata.us/data/Result/search` — sample results (CSV only).
- `https://www.waterqualitydata.us/Codes/<domain>` — controlled vocabularies (JSON).

Things the next person would otherwise rediscover:

- **`Result/search` has no JSON profile.** `mimeType=geojson` works for Station
  and is rejected for Result; the CSV is parsed in-pack.
- **Dates on the wire are MM-DD-YYYY** (`startDateLo=01-01-2020`), not ISO. The
  tools take `YYYY-MM-DD` and convert, so callers never see it.
- **`zip=no` is mandatory** or the response is a zip archive.
- Responses are large — one statewide station query is ~750 KB. Every tool caps
  its rows and reports `truncated`.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "water-quality-portal": {
      "url": "https://gateway.pipeworx.io/water-quality-portal/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/water-quality-portal/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1857+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/wqp_search_sites \
  -H 'Content-Type: application/json' \
  -d '{"statecode":"US:06","countycode":"US:06:081","characteristicName":"Nitrate","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/wqp_search_sites`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-water-quality-portal` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/water-quality-portal/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "water-quality-portal": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-water-quality-portal"]
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
ask_pipeworx({ question: "your question about Water Quality Portal data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
