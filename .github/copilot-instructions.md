# Copilot Instructions

## Project Overview

Pure static HTML/JS Taiwan Stock day-trading P&L calculator. No build system, no framework, no package manager. Deployed on Vercel as static files.

## File Structure & Routing

- `index.html` — Immediately redirects to `mobile-ios.html` via `location.replace()`. Contains the original desktop implementation with full business logic (kept as reference/fallback).
- `mobile-ios.html` — **Primary file.** iOS-first phone-shell UI. All active feature development goes here.
- `taiwan-stock-tick-calculator-ios.html` — Archived duplicate of the iOS version; do not modify.

## Architecture

Both HTML files are fully self-contained (CSS + JS inline in `<script>`/`<style>` tags). There is no bundler, no imports, and no external JS other than Tailwind CSS via CDN.

### Business Logic Engines (inside each HTML file's `<script>` block)

**Tick Engine** — Taiwan Stock Exchange price-tick rules by zone:

| Price range    | Tick size |
|---------------|-----------|
| < 10          | 0.01      |
| 10 – < 50     | 0.05      |
| 50 – < 100    | 0.1       |
| 100 – < 500   | 0.5       |
| 500 – < 1000  | 1.0       |
| ≥ 1000        | 5.0       |

`tickDown()` has boundary-crossing logic: when stepping down crosses a price-zone boundary, use the smaller tick of the lower zone.

**Fee Engine**
- Base rate: `0.1425%` per leg (buy and sell charged separately)
- Optional minimum fee: 20 TWD
- Rounding: configurable `Math.floor` (default, most brokers) or `Math.round`
- `parseDiscount(v)` normalises three input formats: `2.5` (折) → `0.25`, `25` (%) → `0.25`, `0.25` (ratio) → `0.25`

**P&L Engine**
- Tax (證交稅) is charged on the **sell leg only** (TW law)
- Long: `netPnL = (sellAmt − sellFee − tax) − (buyAmt + buyFee)`; return base = `totalBuyCost`
- Short: same formula, but entry is the sell leg; return base = `sellAmt` (entry notional)

**Tax rates**

| Type              | Rate   |
|-------------------|--------|
| 當沖股票 (day trade) | 0.15%  |
| 一般股票 (normal)   | 0.30%  |
| 股票 ETF           | 0.10%  |
| 債券 ETF           | 0%     |

### Key Divergence: `index.html` vs `mobile-ios.html`

| Aspect              | `index.html`                           | `mobile-ios.html`                          |
|---------------------|----------------------------------------|--------------------------------------------|
| Floating point      | `prec(n, d=8)` helper (float × 10^d)  | Integer cents arithmetic (price × 100)    |
| State model         | Direct DOM reads on every `calc()`    | `state` object + `saveState()` to localStorage |
| Tax rounding        | `prec(sellAmt * taxRate)`             | `Math.floor(sellAmt * taxRate)`            |
| Tick function names | `getTickSize(p)` (float prices)       | `getTickSizeCents(priceCents)` (integers) |

When porting logic between files, account for these differences.

## Key Conventions

- **Table row ordering**: rows are sorted so the most-favourable tick offsets appear first. Long → descending (+N … −N); Short → ascending (−N … +N). This ensures profitable rows are always at the top.
- **Breakeven detection**: the breakeven row is the *last* row in the leading profitable block (netPnL ≥ 0), explicitly excluding the entry row (offset 0).
- **Quantity modes**: `lots` (1 lot = 1,000 shares) and `shares` (raw count). The active mode is toggled by `setQtyMode()`.
- **CSS class naming for rows**: `row-entry`, `row-be` (breakeven), `row-profit`, `row-loss` — these drive row background colours.
- **`mobile-ios.html` uses CSS custom properties** with `oklch()` colour values and `color-mix(in oklab, …)` — keep all colours in this system, not hex/rgb.
- **No shared JS files**: if you add a helper function used by both HTML files, copy it into both `<script>` blocks.

## Deployment

Deployed to Vercel (project ID in `.vercel/project.json`). Push to main branch triggers automatic deploy. No build step required — Vercel serves the HTML files directly.
