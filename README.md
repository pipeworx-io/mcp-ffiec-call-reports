# FFIEC Call Reports — US bank line-item detail

Every US bank's quarterly balance sheet, income statement, regulatory capital, loan, deposit and credit-quality detail, as filed on the FFIEC Call Report (Consolidated Reports of Condition and Income, forms 031/041/051) — **4,336 filers**, line-item level.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Why this is not part of `fdic`, and not the same thing

Three packs touch US banks and they do different jobs:

| Pack | Gives you | Keys on |
|---|---|---|
| `fdic` | **Summary** financials, institution records, failures — 161 FDIC-derived fields | FDIC certificate |
| **this pack** | **Line items** — thousands of MDRM-coded fields per bank per quarter, as actually filed | Federal Reserve IDRSSD |
| `ncua` | The same shape for **credit unions**, which never appear in either of the above | NCUA charter |

The split is an **id-space** decision, not a preference. A bank carries an FDIC certificate number *and* a Federal Reserve IDRSSD, and they are different numbers for the same institution — JPMorgan Chase Bank is **cert 628** and **rssd 852218**. The CDR keys on IDRSSD; `fdic` keys on cert. Interleaving them in one pack is how a caller passes the id one tool returned into another and silently gets a different bank.

So: both identifiers are accepted here, both are returned, and **every response states which one it resolved on**.

## Tools

| Tool | Answers |
|---|---|
| `ffiec_search_banks` | *Banks in Ohio* — fuzzy name, city, state |
| `ffiec_call_report` | *Show me JPMorgan's call report* — one filer, one quarter, labeled |
| `ffiec_line_item` | One line item over time — assets, deposits, net income, any MDRM code |
| `ffiec_compare_banks` | Rank the industry or compare a named set |
| `ffiec_coverage` | Which quarters are loaded, the filing lag, and what a code means |

## Auth

Keyless.

## Units — the trap that breaks bank-vs-credit-union comparison

**Call reports are filed in thousands of dollars. NCUA files in whole dollars.** Comparing the two on raw numbers is wrong by 1000×, in the credit union's favour.

Every amount here is normalised to whole USD, with the as-filed figure kept alongside:

```json
{ "mdrm": "RCFD2170", "label": "TOTAL ASSETS",
  "value_usd": 4016571000000, "as_filed": 4016571000,
  "unit": "USD (normalised from the thousands banks file in)" }
```

Capital ratios arrive as percentages with a literal `%` and are passed through as `unit: "percent"` — never multiplied.

## What is mirrored

The **seven core schedules** — RC (balance sheet), RI (income statement), RC-R I (regulatory capital), RC-N (past due and nonaccrual), RC-C I (loans by category), RC-E (deposits), RC-K (quarterly averages). Measured: **1.49M values per quarter** of the 5.23M across all 46 schedules.

The full **3,843-code MDRM dictionary** loads regardless, so any code is explainable even where no value is held — `ffiec_coverage({code})` returns `mirrored_values: false` for those.

Non-numeric cells (17,100 a quarter) go to their own table rather than being dropped. `CONF` matters: it means the bank **filed and withheld** the value as confidential, which is a different answer from "not reported".

## Data source

- <https://cdr.ffiec.gov/public/PWS/DownloadBulkData.aspx> — FFIEC Central Data Repository bulk download.

### Things the next person would otherwise rediscover

- **There is no API.** The CDR download is ASP.NET WebForms: the period dropdown is empty until a series is selected via an AJAX postback, so one quarter takes three requests (GET for `__VIEWSTATE`, POST to populate periods, POST to download). No login needed. `www.ffiec.gov/npw/*` returns 403 to scripted clients, so that is not an alternative.
- **`RCFD` vs `RCON` is the consolidation basis**, not two different metrics — RCFD includes foreign offices, RCON is domestic-only. A bank files one or the other for a given line, so a lookup must try both or it returns "not reported" for a bank that reported perfectly well.
- **Line 2 of every schedule file is the human label**, not data. Parsing from line 2 shifts every value by one row.
- **Values are in thousands** (see above).
- **`CONF` is a value.** Do not coerce it to null.
- **The filing lag is ~106 days, measured** — the 03/31/2026 archive is stamped 07/15/2026 inside the ZIP. Not ~60, which an earlier draft of this file said. That matters for the freshness SLA: with a 91-day quarterly cadence on top of a 106-day lag, the newest data we can legitimately hold reaches ~197 days old just before the next release, so anything under 200 days false-alarms every cycle.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ffiec-call-reports": {
      "url": "https://gateway.pipeworx.io/ffiec-call-reports/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ffiec-call-reports/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ffiec_search_banks \
  -H 'Content-Type: application/json' \
  -d '{"name":"jpmorgan"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ffiec_search_banks`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "ffiec-call-reports": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ffiec-call-reports"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-ffiec-call-reports
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ffiec Call Reports data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
