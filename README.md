# BigIron

Search live and confirmed-sold ag/construction/livestock/transportation
equipment auctions from [bigiron.com](https://www.bigiron.com), a national
US online timed-auction platform.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

| Tool | What it returns |
|------|-----------------|
| `bigiron_search` | Active listings for a keyword query: current bid, Buy-It-Now asking price (if offered), make/model/year, seller, location, category, auction close time. |
| `bigiron_sold_search` | Confirmed-SOLD lots for a keyword query, newest closes first: title, make/model/year, quantity, seller, location, close date, bid count. `final_price` is always `null` — see the price gap below. |
| `bigiron_lot_detail` | Full detail for one lot by URL slug: description, sale status, bid count, make/model/year, seller, location, category and (for active lots) current bid / Buy-It-Now price. |

## The sold-price gap

BigIron does **not** expose the realized hammer price in the page shape this
pack reads. Every closed lot's `amount` field reads `0` regardless of bid
count — verified across 100 sampled sold lots, including several with 60+
recorded bids. The live figure is served by a bidding widget (SignalR hub)
this pack does not call. `bigiron_sold_search` still returns real,
verifiably-sold lots (title, make/model/year, quantity, seller, location,
close date, bid count) with `final_price: null` and an explicit
`price_note` — never a fabricated or implied number.

`buy_it_now_price` on active lots is a real **asking** price a buyer can pay
to end the auction immediately. It is not a sold comp and is labeled as such.

## Auth

None. Keyless, no login, no Cloudflare challenge.

## Data sources

- `bigiron.com/Search` — server-rendered ASP.NET HTML. Every lot card carries
  a `data-lot="{...}"` attribute holding the full lot record as
  HTML-entity-encoded JSON (title, price, location, category, make/model/
  year, seller, end time, buy-it-now, sale status).
- `bigiron.com/Lots/{slug}` — the same per-lot record plus a free-text
  description.

### Upstream quirks (measured 2026-09-13)

- `page=N` paginates (20 lots/page); without `filter=Sold` results skew
  overwhelmingly Active — 7 unfiltered pages of "tractor" turned up exactly 1
  Sold lot. `filter=Sold` (found via the site's own category nav links, e.g.
  `/sale/farm-equipment-for-sale?filter=Sold`) switches the whole result set
  to closed lots.
- `/Past` (the site's own "past auctions" link) requires login — redirects to
  `/MyAccount`. It is not usable keyless; `filter=Sold` on `/Search` is.
- robots.txt disallows AI crawlers by name (ClaudeBot/GPTBot/CCBot etc.), but
  the content is public and served without credentials — per Bruce's
  2026-09-01 ruling ("public data is public data, regardless of robots"),
  this does not block building the pack. This pack crawls politely and
  identifies honestly via its User-Agent.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "bigiron": {
      "url": "https://gateway.pipeworx.io/bigiron/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/bigiron/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "bigiron": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-bigiron"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-bigiron
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Bigiron data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
