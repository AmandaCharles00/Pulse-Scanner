# Metric tooltips for Pulse Scanner column headers

**Date:** 2026-05-23
**Status:** Design approved, ready for implementation plan
**Target PR:** `autumnalcity/Pulse-Scanner` → `AmandaCharles00/Pulse-Scanner` (main)

## Motivation

The current column labels conflate two unrelated metrics under a common "VOL" prefix:

- **`VOL ×`** measures **volume** (today's shares traded vs. yesterday's).
- **`VOL %`** measures **volatility** (ATR as % of price).

A user reading the table intuitively assumes `VOL ×` is a volatility multiple of `VOL %`. For Ford (F) the displayed values are `VOL × = 1.5x`, `VOL % = 4.2%`, `Today = +9.00%` — the "today's move ÷ typical volatility" ratio (~2.14×) is not what `VOL ×` represents, but the label suggests it might be. Other columns (`ATR`, `ATR/50MA`, `RS`) have non-obvious formulas as well.

The fix: hover/focus tooltips on the column headers whose math isn't self-evident from the label, defining each metric in plain language alongside its literal formula.

## Scope

**In scope** — tooltips for five header columns:

- `VOL ×` (ticker views)
- `ATR` (ticker views)
- `VOL %` (ticker views) / `ATR%` (theme + sector views) — same metric, same tooltip content
- `ATR/50MA` (ticker views)
- `RS` (theme + sector views)

**Out of scope:**

- Tooltips on self-evident column headers (`Ticker`, `Price`, `Today`, `1W`, `1M`, `3M`, `YTD`, `#`).
- Per-cell tooltips on color-coded chips (`volChip`, `ATR/50MA` arrows).
- A dedicated glossary/help panel.
- Renaming the inconsistent `ATR%` (Sectors) vs. `Vol %` (Themes) label — flagged as a separate visible change Amanda should sign off on independently.

## Architecture

One helper module added near the existing UI utilities (around `index.html:497`), preserving the single-file vanilla-JS aesthetic.

```text
TOOLTIPS = {
  vol_x:    { def: "...", formula: "..." },
  atr:      { def: "...", formula: "..." },
  vol_pct:  { def: "...", formula: "..." },
  atr_ma:   { def: "...", formula: "..." },
  rs:       { def: "...", formula: "..." },
}

setTT(el, key)             // attach tooltip to one element
wireTooltips(rootEl)       // walk root.querySelectorAll('[data-tt]'), idempotent
showTooltip(el, content)   // position ttPop above/below el, populate, show
hideTooltip()              // hide ttPop
```

- **`ttPop`** — a single hidden `<div class="tt">` appended once to `<body>` with `position: fixed`. Two children: `.tt-def` (sans, regular weight) and `.tt-formula` (`var(--mono)`, dimmer color, small top margin).
- **`setTT(el, key)`** — sets `tabindex="0"` and `data-tt="<key>"` on the element, attaches `mouseenter` / `mouseleave` / `focus` / `blur` / `click` listeners. The `click` listener dismisses the tooltip so it doesn't fight the sort-indicator update.
- **`wireTooltips(rootEl)`** — finds all `[data-tt]` elements within `rootEl` and calls `setTT` on each. Skips elements already marked `data-tt-wired="1"`. Used for sites that build headers via `innerHTML` string assignment.
- **`showTooltip`** positions `ttPop` using `getBoundingClientRect()`. Default position: directly below the header, horizontally centered. If the popover would clip off the viewport bottom, flip above. Clamp horizontal position so it never bleeds off left or right edge.

### Styling

Two CSS rules added to the existing `<style>` block, using existing CSS custom properties:

```css
.tt {
  position: fixed;
  background: var(--bg3);
  border: 1px solid var(--border2);
  border-radius: 4px;
  padding: 8px 10px;
  max-width: 320px;
  font-size: 12px;
  color: var(--text2);
  z-index: 9999;
  pointer-events: none;
  box-shadow: 0 4px 12px rgba(0,0,0,0.4);
  display: none;
}
.tt-formula {
  margin-top: 6px;
  font-family: var(--mono);
  font-size: 11px;
  color: var(--text3);
}
```

### Accessibility

- Each tooltipped header is keyboard-focusable via `tabindex="0"`.
- Tooltip appears on `focus` as well as `mouseenter`.
- Touch / mobile is graceful no-op — the app is already desktop-only.

## Content

Each tooltip has a definition line (sans, normal text color) and a formula line (monospace, dimmer color).

### `VOL ×` *(ticker views)*

> Today's share volume vs. yesterday's. 1.5× = 50% more shares traded today than yesterday.
>
> `vol × = today's volume ÷ yesterday's volume`

### `ATR` *(ticker views)*

> Average True Range over the last 14 days, in dollars — how much a share typically moves in a day. $0.63 means typical daily swings of ~$0.63.
>
> `ATR = 14-day Wilder smoothing of daily true range`

### `VOL %` (ticker) / `ATR%` (theme + sector) — shared content

> Average daily price range (ATR) as a % of price — measures typical volatility. 4% means typical daily swings around ~4% of the stock's price.
>
> `vol % = ATR ÷ price × 100`

### `ATR/50MA` *(ticker views)*

> How far current price has stretched from its 50-day moving average, measured in ATR units. +3.0 means price is 3 ATRs above the 50MA (extended); −2.0 means 2 ATRs below.
>
> `ATR/50MA = (price − 50-day MA) ÷ ATR`

### `RS` *(theme + sector views)*

> Relative Strength rank, 1–98 — how strong this theme's performance is across 5 timeframes vs. all other themes. 50 = median, 80+ = strong, 20− = weak.
>
> `RS = weighted rank of [today×2, 1W×2, 1M×1.5, 3M×1, YTD×0.5], normalized 1–98`

## Coverage — render sites to modify

Seven sites in `index.html`. Line numbers reflect the current state of `main` and should be re-verified at implementation time.

| # | Line | View | Render style | Headers affected | Strategy |
|---|---|---|---|---|---|
| 1 | 388 | Sectors list | imperative `createElement` | `ATR%`, `RS` | One-line `setTT(hvol2, 'vol_pct'); setTT(hrs, 'rs');` after the element creations |
| 2 | 751 | Themes list | imperative `createElement` | `Vol %`, `RS` | Same pattern as #1 |
| 3 | 838 | Drill view | `mkH()` helper | `Vol ×`, `ATR`, `Vol %`, `ATR/50MA` | Add optional 3rd argument to `mkH`, e.g. `mkH('Vol ×', 'vol', 'vol_x')`; `mkH` calls `setTT` on the created `<th>` when the arg is present |
| 4 | 1138–1141 | 50MA Scan | `innerHTML` string | `ATR`, `Vol %`, `ATR/50MA`, `Vol ×` | Add `data-tt="..."` attributes inline in the HTML string, call `wireTooltips(body)` after assignment |
| 5 | 1348–1349 | Positions | column config objects | `ATR/50MA`, `Vol ×` | Add a `tt: 'atr_ma'` field to column objects, propagate to rendered `<th>` as `data-tt`, call `wireTooltips` on the table |
| 6 | 1777–1778 | Morning Gaps | `innerHTML` string | `ATR/50MA`, `Vol ×` | Same pattern as #4 |
| 7 | 2010–2013 | Extension scan | `innerHTML` string | `ATR/50MA`, `Vol %`, `Vol ×` | Same pattern as #4 |

Idempotency: `wireTooltips` marks each wired element with `data-tt-wired="1"` and skips already-wired ones, so re-renders don't stack listeners.

## Edge cases

- **Viewport clipping**: tooltip flips above the header if showing below would clip off the bottom edge. Horizontal position is clamped to the viewport.
- **Sort interaction**: clicking a tooltipped header (which sorts the column) immediately hides the tooltip; sort behavior is unchanged.
- **Re-renders**: `wireTooltips` is idempotent via `data-tt-wired="1"` guard, so the same header re-rendered after a sort doesn't end up with duplicate listeners.
- **Scroll**: while the tooltip is visible, a scroll event hides it (the fixed-position popover would otherwise float in a wrong location relative to the scrolling header).
- **Missing data**: tooltip content is static — it does not depend on cached data, so it works even before sync completes.

## Verification (manual, in browser)

1. Open `index.html` directly in Chrome/Edge.
2. Enter Polygon API key, click Sync, wait for completion.
3. **Themes tab (default)**: hover `Vol %`, `RS` headers — popover appears with correct content.
4. **Sectors tab**: hover `ATR%`, `RS`.
5. **Drill view** (click any theme row to drill in): hover `Vol ×`, `ATR`, `Vol %`, `ATR/50MA`.
6. **50MA Scan tab**: hover `ATR`, `Vol ×`, `Vol %`, `ATR/50MA`.
7. **Positions tab**: hover `Vol ×`, `ATR/50MA`.
8. **Morning Gaps tab**: hover `Vol ×`, `ATR/50MA`.
9. **Extension tab**: hover `Vol ×`, `Vol %`, `ATR/50MA`.
10. **Keyboard**: Tab through the page — popover shows on focus, hides on blur.
11. **Viewport edge**: resize window so a header sits near the bottom — popover flips above.
12. **Sort interaction**: click a header — popover dismisses, sort still works.
13. **Non-tooltipped columns** (`Ticker`, `Price`, `Today`, `1W`, `1M`, `3M`, `YTD`, `#`): hover shows no popover.
14. **Repeated drilling/sorting**: drill into a theme, sort, drill back, drill in again — no duplicate listeners (each header still shows the popover exactly once on hover).

## Estimated diff size

- ~50 lines of JS added (`TOOLTIPS` map, `setTT`, `wireTooltips`, `showTooltip`, `hideTooltip`, the `ttPop` element creation).
- ~15 lines of CSS added (`.tt`, `.tt-formula`).
- ~1 line change at each of 7 render sites (a `setTT(...)` call, a `wireTooltips(...)` call, or a one-arg addition to `mkH`).

Total: well under 100 lines added, minimal modifications to existing code. Self-contained in `index.html` — no new files, no build step changes.
