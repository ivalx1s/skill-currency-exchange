# currency-exchange

Project skill for the Exchange CLI — a currency exchange calculator.

## What it covers

- Architecture and project structure (SwiftPM, ExchangeLib, CLI entry point)
- Rate providers: Google scraping (WebKit), rate.am scraping (WebKit), open.er-api.com (REST)
- WebKitEngine pattern (headless WKWebView, setupScript/javaScript lifecycle)
- rate.am DOM structure and extraction strategy (exchange offices page, stale detection, IQR filtering)
- Buy/sell/avg rate semantics
- Output formatting (tax, rates, totals in both currencies)
- Common pitfalls and gotchas

## Supported currencies

Currently: **AMD** (Armenian Dram) and **RUB** (Russian Ruble) only.

The `Currency` enum is designed for extension — adding new currencies is straightforward, but rate providers and CLI logic will need updates (e.g. `--to`/`--from` options instead of hardcoded pair).

## Files

- `SKILL.md` — full technical reference
