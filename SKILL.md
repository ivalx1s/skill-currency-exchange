---
name: currency-exchange
description: >
  Currency exchange calculator CLI workflow for AMD/RUB conversions using
  real cash rates and the local `exchange` command.
triggers:
  - currency exchange
  - exchange cli
  - amd rub
  - rate.am
  - поменять драмы
  - поменять рубли
---

# Exchange CLI — Project Skill

> Currency exchange calculator CLI: AMD <-> RUB with real cash rates from Armenian exchange offices.

## When Triggered (MANDATORY)

When the user asks to exchange/convert currencies (AMD, RUB, драмы, рубли, rate.am, etc.):

1. **ALWAYS use the `exchange` CLI tool** — NEVER scrape rate sources manually, NEVER use WebFetch.
2. Map the user's request to CLI flags:
   - "поменять драмов на рубли" / "give AMD, get RUB" → `--give amd=<amount>`
   - "поменять рублей на драмы" / "give RUB, get AMD" → `--give rub=<amount>`
   - "хочу получить X рублей" → `--get rub=<amount>`
   - "по курсу раттам" / "rate.am" → `--ratam`
   - "раттам среднему" / "mid-market rate.am" → `--ratam --avg`
   - "по гуглу" / "google rate" → `--google`
   - "среднему по гуглу" → `--google --avg`
   - "комиссия 0.3%" → `--tax=-0.003`
   - "бонус 0.3%" → `--tax=0.003`
   - "курс 4.82" / "manual rate" → `--rate 4.82`
3. Run the command via Bash and show the output.

### Examples

| User says | CLI command |
|-----------|-------------|
| "поменять 25к рублей на драмы по раттам среднему, комиссия 0.3%" | `exchange --give rub=25000 --ratam --avg --tax=-0.003` |
| "хочу получить 10к рублей, курс гугл" | `exchange --get rub=10000 --google` |
| "50к драмов в рубли по курсу 0.205" | `exchange --give amd=50000 --rate 0.205` |
| "25к рублей по раттам кеш" | `exchange --give rub=25000 --ratam` |

## Project Overview

Swift CLI tool (`exchange`) for currency conversion between AMD (Armenian Dram) and RUB (Russian Ruble). Fetches real cash exchange rates from rate.am or mid-market rates from Google, supports manual rate input. macOS native binary using WebKit for headless page scraping.

## Architecture

```
Sources/
  Exchange/Exchange.swift           # @main CLI entry (ArgumentParser)
  ExchangeLib/
    Currency.swift                  # Enum: amd, rub (extensible)
    RateProvider.swift              # Protocol + RateError enum
    RateResolver.swift              # Wraps provider, optional --avg (both directions averaging)
    WebKitEngine.swift              # Headless WKWebView: load URL -> setupScript -> delay -> JS -> result
    GoogleScrapingProvider.swift    # Scrapes Google Search for mid-market rate
    RateAmProvider.swift            # Scrapes rate.am exchange offices (cash rates, IQR-trimmed)
    OpenErApiProvider.swift         # open.er-api.com REST API (fallback, no WebKit)
    IQRFilter.swift                 # Interquartile range outlier trimming
    Formatter.swift                 # Output formatting (rate, tax, total in both currencies)
Tests/ExchangeTests/                # Swift Testing framework (NOT XCTest)
```

## Tech Stack

- **Swift 5.9+**, SwiftPM, macOS 13+
- **ArgumentParser** for CLI options
- **WebKit (WKWebView)** for headless page scraping (Google, rate.am)
- **URLSession** for REST API calls (OpenErApi)
- **Swift Testing** for tests

## CLI Options

```
--give <currency=amount>   I give this, how much do I get?
--get <currency=amount>    I want to get this, how much do I give?
--google                   Fetch mid-market rate (via WebKit scraping)
--ratam                    Cash rate from rate.am (buy or sell based on direction)
--ratam --avg              Mid-market from rate.am (IQR(buys) + IQR(sells)) / 2
--google --avg             Average both directions for true mid-market
--rate <decimal>           Manual exchange rate
--tax <decimal>            Tax/commission on output. +0.003 = +0.3% bonus, -0.003 = -0.3% commission
```

Constraints: `--give`/`--get` mutually exclusive. `--google`/`--ratam`/`--rate` mutually exclusive. `--avg` requires `--google` or `--ratam`.

## Rate Provider Pattern

All providers implement `RateProvider`:
```swift
public protocol RateProvider {
    func rate(from: Currency, to: Currency) async throws -> Double
}
```

### WebKitEngine

Reusable headless browser engine:
1. Creates `WKWebView` off-screen (1280x800)
2. Loads URL
3. On `didFinish`: runs optional `setupScript` (e.g. click a button)
4. Waits `renderDelay` (default 2s) for JS frameworks to render
5. Evaluates `javaScript` extraction script
6. Returns result string (or nil)
7. Has timeout safety (default 15s)

Key: `setupScript` runs BEFORE delay (for clicking buttons/tabs that trigger re-render). Main `javaScript` runs AFTER delay (for extracting rendered data).

### RateAmProvider — rate.am Cash Rates

**URL:** `https://www.rate.am/ru/armenian-dram-exchange-rates/exchange-points`

**DOM structure (as of 2026-02):**
- Next.js RSC app, no HTML tables — flat divs
- Exchange offices page (NOT banks): no CASH/CLEARING toggle needed
- Rate container: `.overflow-auto` div
- Each exchange office row: `div[class*="h-12"][class*="items-center"]`
- Stale rows (>1 hour old): have `text-N60` CSS class (grayed out text)
- Fresh rows: no `text-N60` class (normal black text `rgb(31,31,31)`)

**Rate data layout per exchange office:**
- 6 numbers in innerText: USD buy, USD sell, EUR buy, EUR sell, RUR buy, RUR sell
- Some offices skip RUR (only 4 numbers: USD + EUR)
- No "Среднее" (average) row on exchange-points page (unlike banks page)
- Rates are AMD per 1 unit of foreign currency

**JS extraction strategy:**
1. Count fresh rows (no `text-N60`)
2. Parse all numbers from container `innerText`
3. Find start of rate data (first number in USD range 300-500)
4. Group numbers by 6 (USD b/s, EUR b/s, RUR b/s)
5. Skip groups with only 4 (no RUR)
6. Take only first `freshCount` entries
7. Return `[[buy, sell], ...]` JSON

**Rate semantics (rate.am perspective):**
- **Buy** = exchange office buys RUR from you = you get AMD → **RUB->AMD direction**
- **Sell** = exchange office sells RUR to you = you pay AMD → **AMD->RUB direction**
- Both expressed as AMD per 1 RUR
- For AMD->RUB, return `1/sell` (not raw sell value!)

**Averaging modes:**
- `--ratam` (no avg): IQR-trimmed average of buy OR sell depending on direction
- `--ratam --avg`: `(IQR(buys) + IQR(sells)) / 2` — mid-market of cash rates
- Provider handles avg internally, RateResolver always `average: false` for ratam

### IQR Filter

Standard Interquartile Range outlier removal:
- Sort values, compute Q1 (median of lower half), Q3 (median of upper half)
- IQR = Q3 - Q1
- Keep values in [Q1 - 1.5*IQR, Q3 + 1.5*IQR]
- Return average of survivors
- For < 4 values: just return simple average (IQR not meaningful)

### GoogleScrapingProvider

Scrapes `google.com/search?q=1+{FROM}+to+{TO}` via WebKit. Tries multiple DOM selectors:
1. `[data-value]` attributes (currency converter widget)
2. `[data-last-price]` (Google Finance)
3. Known CSS classes (`.DFlfde`, `.SwHCTb`, `.pclqee`, `.b4mUmd`)
4. Input text fields with converted amounts

### RateResolver

Wraps any provider. With `--avg`:
1. Fetch forward rate (A->B)
2. Fetch reverse rate (B->A)
3. Invert reverse: 1/reverse
4. Return `(forward + 1/reverse) / 2`

Used for `--google --avg`. NOT used for `--ratam --avg` (provider handles that internally).

## Output Format

### Without tax:
```
25,000.00 RUB = 120,500.00 AMD
Rate: 1 RUB = 4.820000 AMD
      1 AMD = 0.207469 RUB
```

### With tax (order: conversion, tax, rate, total):
```
25,000.00 RUB = 120,138.50 AMD
Tax (-0.30%): -361.50 AMD / -75.00 RUB
Rate: 1 RUB = 4.820000 AMD
      1 AMD = 0.207469 RUB
Total: 120,138.50 AMD / 24,925.00 RUB
```

Key: amounts formatted with commas + 2 decimal places (en_US locale). Rates with 6 decimal places. Tax and Total shown in both currencies.

## Build & Run

```bash
swift build                              # debug
swift build -c release                   # release
swift test                               # run tests
swift run exchange --give amd=50000 --google
bash scripts/setup.sh                    # build + install to ~/.local/bin/
```

## Common Pitfalls

- **DOM changes**: rate.am is Next.js RSC, DOM structure may change. If scraping breaks, dump DOM first (see `.temp/dump-*.swift` debug scripts).
- **Stale detection**: `text-N60` class marks old entries. If they change the styling, stale filtering will break.
- **Number parsing heuristics**: JS extraction uses value ranges (USD 300-500, RUR 1-20) to distinguish currencies. If exchange rates shift dramatically, ranges need updating.
- **AMD->RUB direction**: Returns `1/sell`, not raw sell. Easy to forget and return wrong direction.
- **IQR with few values**: < 4 data points = simple average, no outlier removal.
- **WebKit MainActor**: `WebKitEngine` is `@MainActor`. All WKWebView operations must be on main thread.
- **setupScript vs javaScript**: setupScript runs immediately after load (before delay), javaScript runs after delay. Don't confuse them.
