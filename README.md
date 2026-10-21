# @pipeworx/datacommons

Harmonised public statistics for any place on Earth from Google Data Commons:
population, income, health, employment, emissions and thousands more indicators.
Every value names the original publisher (US Census, World Bank, Eurostat, CDC,
BLS, ...).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1854+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `datacommons_find_place(name, place_type?, limit?)`: turns a place name into
  Data Commons place ids (dcids), with name and type for each candidate.
  "California" + State gives `geoId/06`. "India" gives `country/IND`.
- `datacommons_find_variable(query, limit?)`: turns a plain-language indicator
  into statistical variable ids, with a match score. "median household income"
  gives `Median_Income_Household`.
- `datacommons_observation(variable, place, date?, limit?)`: one variable at one
  place. Returns the latest value (default), the value at a date, or the whole
  series with `date=all`. The response carries the publisher of the value and how
  many other sources report it, plus the latest value from up to five of them.
- `datacommons_child_places(variable, parent_place, child_type, date?, sort?, limit?)`:
  one variable across every child place of a parent, sorted by value. Use it for
  all US states, every county in Texas (`geoId/48`, `County`) or every country in
  `europe`.

A place/variable pair with no data is an error that names both ids. It never
comes back as an empty list.

## Auth

Platform key (`PLATFORM_DATACOMMONS_KEY`) with a BYO override through
`_apiKey`, which is sent upstream as the `X-API-Key` header. Through the gateway,
callers without a key use ours. Anyone importing the package directly needs a
free Data Commons key from https://apikeys.datacommons.org (self-serve, no
approval step). Without one, the pack refuses with "requires an API key" before
any upstream request.

## Data sources

- Google Data Commons, the Data Commons knowledge graph (REST v2). It aggregates
  official statistics from national and international publishers. Each value's
  `publisher` object names the dataset, measurement method and the publisher's
  provenance page.

Reuse terms: this pack calls the API live for each request and keeps no copy.
Each value stays under its original publisher's terms, and the `publisher`
object on every response links to that publisher's provenance page.

Notes for the next person:

- Most indicators have several sources ("facets") that disagree. In 2024, US
  population was 340.0M from Census PEP and 334.9M from the ACS 5-year survey.
  The pack takes Data Commons' own ranking (first facet with observations) and
  shows the alternates, so the caller can see the spread.
- Name resolution is fuzzy. "Paris" returns NUTS region FR101 first and then
  about a dozen US towns. Pass `place_type` to narrow it.
- The indicator resolver always returns something, even for nonsense, so it falls
  back to topics. The pack keeps only `StatisticalVariable` hits and sets
  `low_confidence` when the best score is under 0.7.
- Data Commons publishes a rate-limited trial key in its own docs. It is good
  for a one-off smoke test, not for serving callers.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "datacommons": {
      "url": "https://gateway.pipeworx.io/datacommons/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/datacommons/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1854+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/datacommons_find_place \
  -H 'Content-Type: application/json' \
  -d '{"name":"California","place_type":"State"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/datacommons_find_place`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "datacommons": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-datacommons"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-datacommons
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Datacommons data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
