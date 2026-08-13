# Market Data Dashboard System

Three self-contained operations boards that pull live macroeconomic data from official
statistical agencies, reconcile it against itself, and show the pipeline doing the work.

Each board is a single `.dc.html` file. There is no build step and no server — open one in a
browser and it fetches, caches, validates and paints on its own.

| Board | Region | Providers | Series |
|---|---|---|---|
| `Euro Operations Board.dc.html` | Euro area | ECB SDMX, World Bank | 15 |
| `Singapore Operations Board.dc.html` | Singapore | SingStat Table Builder (MAS, DOS, MOM) | 6 tables |
| `US Operations Board.dc.html` | United States | US Treasury, BLS, Alpha Vantage | 8 |

## The idea

Two ideas, applied on every board.

**Nominal against real.** A yield, a wage or a growth rate means little on its own. Every
headline figure is shown next to the price level that deflates it, so the real number is never
left as an exercise for the reader.

**The pipeline is part of the product.** Most dashboards show numbers and hide how they were
obtained. These split the screen: what the data says on the left, what the system did to get it
on the right — cache hits, request latency, retries, rejected payloads, and which figures are
being served from last-known-good because a source is down.

## Data integrity

Each board cross-checks its own sources rather than trusting a single print:

- **Euro / Singapore** — the published annual inflation rate is recomputed from the index the
  publisher derived it from, and the gap is displayed. Same provider, so this catches vintage
  drift, rebasing and arithmetic errors, not agency error.
- **US** — the same headline series is pulled from two independent providers (BLS as source of
  record, Alpha Vantage as a mirror). Where they disagree, the gap is printed rather than
  averaged away.
- **Vintages are never mixed.** Where two indicators return different latest-published years,
  the board says so instead of summing across them.

## Pipeline

A shared engine behind all three boards:

- **TTL cache** per series, hydrated from `localStorage` so the board paints before its first
  request completes.
- **Bounded concurrency** — configurable, defaults to 3. The World Bank endpoint drops parallel
  connections, so its four series run in a single lane.
- **Retry with exponential backoff and jitter**, three attempts.
- **Schema validation** on every payload. Rejected responses are *not* retried; the last known
  good value is held and the rejection is logged.
- **Graceful degradation** — a dead source degrades to `stale` or `down` in the health panel
  rather than blanking the board.
- **Request budgeting** — the US board tracks BLS's 25-request/day unregistered cap and skips
  rather than burning it, which is what sets the TTL.

### Fault injection

Every board ships toggles that force the failure paths in a live session: `kill` (connection
refused), `slow` (2s stall), `corrupt` (poisoned observations, to exercise the validator), plus
`trunc` on Singapore and `budget` on the US board. Set the `showFaultControls` prop to `false`
to hide them.

## Running

Open any `.dc.html` file in a browser. `support.js` and `_ds/` must sit alongside it — they are
the runtime and the design system.

All data comes from public endpoints and no API key is required. The US board's Alpha Vantage
mirror uses the provider's public `demo` key; when it rate-limits, the board reports it as a
rejected payload and carries on, which is the intended behaviour.

## Configuration

Each board exposes props: `refreshSeconds` (5–120), `maxConcurrency` (1–6), `showFaultControls`,
and on the Euro board `homeCountry`, which selects the highlighted economy in the World Bank
comparison table.

## Sources

- [ECB Data Portal SDMX API](https://data.ecb.europa.eu/help/api/overview)
- [World Bank Indicators API](https://datahelpdesk.worldbank.org/knowledgebase/articles/889392)
- [SingStat Table Builder](https://tablebuilder.singstat.gov.sg/)
- [US Treasury Daily Yield Curve](https://home.treasury.gov/interest-rates-data-csv-archive)
- [BLS Public Data API](https://www.bls.gov/developers/)
- [Alpha Vantage](https://www.alphavantage.co/documentation/)
