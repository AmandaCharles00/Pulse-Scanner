# Metric tooltips Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add hover/focus tooltips with definition + formula to the five non-obvious column headers (`VOL ×`, `ATR`, `VOL %` / `ATR%`, `ATR/50MA`, `RS`) across all seven views where those headers appear.

**Architecture:** A single tooltip-helper module (~50 lines JS + ~15 lines CSS) added inline to `index.html`. One shared `position: fixed` popover element appended to `<body>`. Helper exposes `setTT(el, key)` for imperative DOM sites and `wireTooltips(root)` for `innerHTML`-built sites. Idempotent via `data-tt-wired="1"` guard. Vanilla JS, no deps, fits the existing single-file aesthetic.

**Tech Stack:** Vanilla JavaScript, inline CSS, native DOM APIs. No build step. No test framework (verification is manual in browser, matching the existing codebase convention).

**Spec:** [`docs/superpowers/specs/2026-05-23-metric-tooltips-design.md`](../specs/2026-05-23-metric-tooltips-design.md)

**Notes:**
- `index.html` has no test framework. Each task's verification is **manual in the browser**. The plan calls this out at each verification step.
- Line numbers below reflect `main` at plan-write time. If Amanda has pushed an upload between now and execution, re-verify line numbers by grep before editing.
- Each task ends with a commit. Tim has approved this workflow by accepting the brainstorming → plan path. If you want to batch commits at the end instead, ask before deviating.

---

## File structure

Single file modified: `index.html`. Two logical regions added, no new files created.

**Insertion points (verify before editing):**

- **CSS block** (`.tt`, `.tt-formula`, `[data-tt]` cursor): inside the existing `<style>...</style>` block at the top of the file, immediately before `</style>` (currently around line 118).
- **JS helper** (`TOOLTIPS` map, `showTooltip`, `hideTooltip`, `setTT`, `wireTooltips`, global scroll/resize listeners): after the existing visualization utility functions (`volChip`, `miniBar`, `atrBar`, `heatBg`) and before `function computeRS()` (currently around line 506–512).

**Render sites modified** (no new responsibilities — just adding tooltip wiring at each):

| Site | Current line | View | Headers |
|---|---|---|---|
| A | 388 | Sectors list | `ATR%`, `RS` |
| B | 751 | Themes list | `Vol %`, `RS` |
| C | 837 (`mkH` defn) + 838 (calls) | Drill view | `Vol ×`, `ATR`, `Vol %`, `ATR/50MA` |
| D | 1132 (innerHTML) | 50MA Scan | `ATR`, `Vol %`, `ATR/50MA`, `Vol ×` |
| E | 1343 (config) + 1352–1369 (render loop) | Positions | `ATR/50MA`, `Vol ×` |
| F | 1772 (innerHTML) | Morning Gaps | `ATR/50MA`, `Vol ×` |
| G | 2007 (innerHTML) | Extension scan | `ATR/50MA`, `Vol %`, `Vol ×` |

---

### Task 1: Create feature branch

**Files:**
- No file changes — git branch only.

- [ ] **Step 1: Verify clean working tree on `main`**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" status
```
Expected: `On branch main`, working tree clean (the spec doc was written to `docs/` but is untracked; that's fine — we'll commit it in this task).

- [ ] **Step 2: Create and check out feature branch from `main`**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" checkout -b feature/metric-tooltips
```
Expected: `Switched to a new branch 'feature/metric-tooltips'`.

- [ ] **Step 3: Stage and commit the spec + plan docs**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" add docs/superpowers/specs/2026-05-23-metric-tooltips-design.md docs/superpowers/plans/2026-05-23-metric-tooltips.md
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" commit -m "docs: spec and plan for metric column tooltips"
```
Expected: one new commit on `feature/metric-tooltips`.

---

### Task 2: Add CSS rules for the tooltip popover

**Files:**
- Modify: `index.html` — insert inside the existing `<style>` block, immediately before `</style>` (around line 118).

- [ ] **Step 1: Locate the closing `</style>` tag**

Run:
```
grep -n "</style>" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
```
Expected: one line number, near 118–120.

- [ ] **Step 2: Insert the CSS block immediately before `</style>`**

Insert this exact block:

```css
.tt{position:fixed;background:var(--bg3);border:1px solid var(--border2);border-radius:4px;padding:8px 10px;max-width:320px;font-size:12px;color:var(--text2);z-index:9999;pointer-events:none;box-shadow:0 4px 12px rgba(0,0,0,.4);display:none;line-height:1.4}
.tt-formula{margin-top:6px;font-family:var(--mono);font-size:11px;color:var(--text3)}
[data-tt]{cursor:help}
[data-tt]:focus{outline:1px dotted var(--text3);outline-offset:2px}
```

Notes on the additions beyond what the spec literally specifies:
- `[data-tt]{cursor:help}` — small affordance so users get a visual cue on hover before the popover even shows.
- `[data-tt]:focus{outline:...}` — visible keyboard-focus indicator (`tabindex="0"` headers would otherwise look identical when focused vs not).

These are minor accessibility/affordance additions consistent with the spec's accessibility intent.

- [ ] **Step 3: Verify the CSS exists in the file**

Run:
```
grep -n "\.tt{position:fixed" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
```
Expected: one match, near the previous `</style>` line.

- [ ] **Step 4: Open `index.html` in browser, confirm no visual regression**

Open `C:/Users/Tim Parker/Projects/Pulse Scanner/index.html` in Chrome/Edge. Verify the existing app still loads and looks identical (no new visible elements — `.tt` is `display:none` until JS shows it).

- [ ] **Step 5: Commit**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" add index.html
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" commit -m "tooltips: add CSS for header tooltip popover"
```

---

### Task 3: Add JS tooltip infrastructure

**Files:**
- Modify: `index.html` — insert after the visualization utility functions (`volChip`, `miniBar`, `atrBar`, `heatBg`) and before `function computeRS()` (currently around line 506–512).

- [ ] **Step 1: Locate the exact insertion point**

Run:
```
grep -n "^function computeRS" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
```
Expected: one line number (currently 513).

Insert the new block on the blank line immediately above `function computeRS`.

- [ ] **Step 2: Insert the JS helper block**

Insert this exact block:

```javascript
// ── TOOLTIPS ─────────────────────────────────────────────────────────────────
const TOOLTIPS={
  vol_x:{def:"Today's share volume vs. yesterday's. 1.5× = 50% more shares traded today than yesterday.",formula:"vol × = today's volume ÷ yesterday's volume"},
  atr:{def:"Average True Range over the last 14 days, in dollars — how much a share typically moves in a day. $0.63 means typical daily swings of ~$0.63.",formula:"ATR = 14-day Wilder smoothing of daily true range"},
  vol_pct:{def:"Average daily price range (ATR) as a % of price — measures typical volatility. 4% means typical daily swings around ~4% of the stock's price.",formula:"vol % = ATR ÷ price × 100"},
  atr_ma:{def:"How far current price has stretched from its 50-day moving average, measured in ATR units. +3.0 means price is 3 ATRs above the 50MA (extended); −2.0 means 2 ATRs below.",formula:"ATR/50MA = (price − 50-day MA) ÷ ATR"},
  rs:{def:"Relative Strength rank, 1–98 — how strong this theme's performance is across 5 timeframes vs. all other themes. 50 = median, 80+ = strong, 20− = weak.",formula:"RS = weighted rank of [today×2, 1W×2, 1M×1.5, 3M×1, YTD×0.5], normalized 1–98"}
};

let ttPop=null;
function showTooltip(target,key){
  const c=TOOLTIPS[key]; if(!c||!target) return;
  if(!ttPop){
    ttPop=document.createElement('div');
    ttPop.className='tt';
    ttPop.innerHTML='<div class="tt-def"></div><div class="tt-formula"></div>';
    document.body.appendChild(ttPop);
  }
  ttPop.querySelector('.tt-def').textContent=c.def;
  ttPop.querySelector('.tt-formula').textContent=c.formula;
  ttPop.style.display='block';
  const r=target.getBoundingClientRect();
  const pr=ttPop.getBoundingClientRect();
  const vh=window.innerHeight, vw=window.innerWidth;
  let top=r.bottom+6;
  if(top+pr.height>vh-8) top=Math.max(8,r.top-pr.height-6);
  let left=r.left+(r.width-pr.width)/2;
  if(left<8) left=8;
  if(left+pr.width>vw-8) left=vw-pr.width-8;
  ttPop.style.top=top+'px';
  ttPop.style.left=left+'px';
}
function hideTooltip(){ if(ttPop) ttPop.style.display='none'; }
window.addEventListener('scroll',hideTooltip,{passive:true,capture:true});
window.addEventListener('resize',hideTooltip,{passive:true});

function setTT(el,key){
  if(!el||!TOOLTIPS[key]||el.dataset.ttWired==='1') return;
  el.dataset.tt=key;
  el.dataset.ttWired='1';
  el.tabIndex=0;
  el.addEventListener('mouseenter',()=>showTooltip(el,key));
  el.addEventListener('focus',()=>showTooltip(el,key));
  el.addEventListener('mouseleave',hideTooltip);
  el.addEventListener('blur',hideTooltip);
  el.addEventListener('click',hideTooltip);
}

function wireTooltips(root){
  (root||document).querySelectorAll('[data-tt]:not([data-tt-wired])').forEach(el=>setTT(el,el.dataset.tt));
}
```

- [ ] **Step 3: Verify the helper exists in the file**

Run:
```
grep -n "const TOOLTIPS=" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
grep -n "function setTT" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
grep -n "function wireTooltips" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
```
Expected: one match each, all in the 506–600 area (above `computeRS`).

- [ ] **Step 4: Open `index.html` in browser, check for JS errors**

Open in Chrome/Edge, open DevTools console (F12). Verify no red errors. The app should load identically to before — the helpers are defined but nothing calls them yet. Also confirm the `.tt` element is NOT yet in the DOM (it's lazy-created on first `showTooltip` call) — search the Elements panel for "class=\"tt\"" — should find nothing.

- [ ] **Step 5: Commit**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" add index.html
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" commit -m "tooltips: add helper module (TOOLTIPS map, setTT, wireTooltips)"
```

---

### Task 4: Wire drill view via `mkH` helper

**Files:**
- Modify: `index.html:837` (`mkH` definition) and `index.html:838` (the four `mkH` calls for metric columns).

- [ ] **Step 1: Locate the `mkH` definition**

Run:
```
grep -n "const mkH=(label,col)" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
```
Expected: one match, currently line 837.

- [ ] **Step 2: Modify `mkH` to accept an optional tooltip key**

Replace this exact line at 837:

```javascript
  const mkH=(label,col)=>{const h=document.createElement('th');h.textContent=label;if(drillSort.col===col)h.className=drillSort.dir;h.style.textAlign=col==='ticker'?'left':'right';h.addEventListener('click',()=>{drillSort.dir=drillSort.col===col&&drillSort.dir==='desc'?'asc':'desc';drillSort.col=col;renderDrill(filter);});return h;};
```

with:

```javascript
  const mkH=(label,col,tt)=>{const h=document.createElement('th');h.textContent=label;if(drillSort.col===col)h.className=drillSort.dir;h.style.textAlign=col==='ticker'?'left':'right';h.addEventListener('click',()=>{drillSort.dir=drillSort.col===col&&drillSort.dir==='desc'?'asc':'desc';drillSort.col=col;renderDrill(filter);});if(tt)setTT(h,tt);return h;};
```

- [ ] **Step 3: Update the four metric `mkH` calls at line 838**

Replace this exact line:

```javascript
  hrow.append(mkH('Ticker','ticker'),mkH('Price','price'),mkH('Vol ×','vol'),mkH('ATR','atr'),mkH('Vol %','atrPct'),mkH('ATR/50MA','atrMA'));
```

with:

```javascript
  hrow.append(mkH('Ticker','ticker'),mkH('Price','price'),mkH('Vol ×','vol','vol_x'),mkH('ATR','atr','atr'),mkH('Vol %','atrPct','vol_pct'),mkH('ATR/50MA','atrMA','atr_ma'));
```

- [ ] **Step 4: Browser-verify the drill view tooltips**

Open `index.html` in Chrome/Edge, enter Polygon API key, click Sync, wait for completion. Click any theme row in the Themes tab to enter drill view.

- Hover `Vol ×` header → popover appears with the volume tooltip; mouse out → hides.
- Hover `ATR` header → popover with ATR tooltip.
- Hover `Vol %` header → volatility tooltip.
- Hover `ATR/50MA` header → deviation tooltip.
- Click `Vol ×` header → table re-sorts, popover dismisses immediately.
- Tab to a tooltipped header (keyboard) → popover appears on focus, focus outline visible.
- Hover `Ticker` and `Price` headers → no popover (we didn't wire them).

If any of these fail, stop and debug before continuing.

- [ ] **Step 5: Commit**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" add index.html
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" commit -m "tooltips: wire drill view headers via mkH helper"
```

---

### Task 5: Wire Sectors list header (`ATR%`, `RS`)

**Files:**
- Modify: `index.html:388–391` (Sectors header creation in `renderSectors()`).

- [ ] **Step 1: Locate the header creation lines**

Run:
```
grep -n "hvol2.textContent='ATR%'" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
```
Expected: one match, currently line 388.

- [ ] **Step 2: Add `setTT` calls immediately after `hrs` is created**

Find the exact existing block (currently lines 388–391):

```javascript
  const hvol2=document.createElement('div');hvol2.className='lhdr-tf';hvol2.style.cssText='width:52px;text-align:right;font-size:9px;color:#ccc;letter-spacing:.06em;text-transform:uppercase;font-family:var(--mono);flex-shrink:0';hvol2.textContent='ATR%';
  const hrs=document.createElement('div');hrs.className='lhdr-rs';hrs.textContent='RS';
  const hcnt=document.createElement('div');hcnt.className='lhdr-cnt';hcnt.textContent='#';
  hdr.append(hvol2,hrs,hcnt);
```

Replace with:

```javascript
  const hvol2=document.createElement('div');hvol2.className='lhdr-tf';hvol2.style.cssText='width:52px;text-align:right;font-size:9px;color:#ccc;letter-spacing:.06em;text-transform:uppercase;font-family:var(--mono);flex-shrink:0';hvol2.textContent='ATR%';
  const hrs=document.createElement('div');hrs.className='lhdr-rs';hrs.textContent='RS';
  const hcnt=document.createElement('div');hcnt.className='lhdr-cnt';hcnt.textContent='#';
  setTT(hvol2,'vol_pct'); setTT(hrs,'rs');
  hdr.append(hvol2,hrs,hcnt);
```

- [ ] **Step 3: Browser-verify Sectors tab**

Reload `index.html`. Click the **Sectors** tab.
- Hover `ATR%` → volatility tooltip.
- Hover `RS` → relative-strength tooltip.

- [ ] **Step 4: Commit**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" add index.html
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" commit -m "tooltips: wire Sectors list ATR% and RS headers"
```

---

### Task 6: Wire Themes list header (`Vol %`, `RS`)

**Files:**
- Modify: `index.html:751–754` (Themes list header creation in `renderList()`).

- [ ] **Step 1: Locate the header creation lines**

Run:
```
grep -n "hvol.textContent='Vol %'" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
```
Expected: one match, currently line 751.

- [ ] **Step 2: Add `setTT` calls immediately after `hrs` is created**

Find the exact existing block (currently lines 751–754):

```javascript
  const hvol=document.createElement('div');hvol.className='lhdr-tf';hvol.style.cssText='width:52px;text-align:right;font-size:9px;color:#ccc;letter-spacing:.06em;text-transform:uppercase;font-family:var(--mono);flex-shrink:0';hvol.textContent='Vol %';
  const hrs=document.createElement('div');hrs.className='lhdr-rs';hrs.textContent='RS';
  const hcnt=document.createElement('div');hcnt.className='lhdr-cnt';hcnt.textContent='#';
  hdr.append(hvol,hrs,hcnt);
```

Replace with:

```javascript
  const hvol=document.createElement('div');hvol.className='lhdr-tf';hvol.style.cssText='width:52px;text-align:right;font-size:9px;color:#ccc;letter-spacing:.06em;text-transform:uppercase;font-family:var(--mono);flex-shrink:0';hvol.textContent='Vol %';
  const hrs=document.createElement('div');hrs.className='lhdr-rs';hrs.textContent='RS';
  const hcnt=document.createElement('div');hcnt.className='lhdr-cnt';hcnt.textContent='#';
  setTT(hvol,'vol_pct'); setTT(hrs,'rs');
  hdr.append(hvol,hrs,hcnt);
```

- [ ] **Step 3: Browser-verify Themes tab**

Reload `index.html`. The **Themes** tab is the default; if you've switched away, click it.
- Hover `Vol %` → volatility tooltip.
- Hover `RS` → relative-strength tooltip.

- [ ] **Step 4: Commit**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" add index.html
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" commit -m "tooltips: wire Themes list Vol % and RS headers"
```

---

### Task 7: Wire 50MA Scan headers (`ATR`, `Vol %`, `ATR/50MA`, `Vol ×`)

**Files:**
- Modify: `index.html:1132–1144` (50MA Scan header `innerHTML` block in `renderScan()`).

- [ ] **Step 1: Locate the `innerHTML` block**

Run:
```
grep -n "width:65px;text-align:right;flex-shrink:0\">ATR<" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
```
Expected: one match in the 50MA Scan area (currently line 1138).

- [ ] **Step 2: Add `data-tt` attributes inline in the innerHTML strings**

Find the existing block (currently lines 1132–1143):

```javascript
  hdr.innerHTML=
    '<div style="width:60px;flex-shrink:0">Ticker</div>'+
    '<div style="flex:1">Theme</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">Price</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">50MA</div>'+
    '<div style="width:90px;text-align:right;flex-shrink:0">% from MA</div>'+
    '<div style="width:65px;text-align:right;flex-shrink:0">ATR</div>'+
    '<div style="width:65px;text-align:right;flex-shrink:0">Vol %</div>'+
    '<div style="width:90px;text-align:right;flex-shrink:0">ATR/50MA</div>'+
    '<div style="width:70px;text-align:right;flex-shrink:0">Vol ×</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">AvgVol $M</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">Today</div>';
  body.appendChild(hdr);
```

Replace with:

```javascript
  hdr.innerHTML=
    '<div style="width:60px;flex-shrink:0">Ticker</div>'+
    '<div style="flex:1">Theme</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">Price</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">50MA</div>'+
    '<div style="width:90px;text-align:right;flex-shrink:0">% from MA</div>'+
    '<div data-tt="atr" style="width:65px;text-align:right;flex-shrink:0">ATR</div>'+
    '<div data-tt="vol_pct" style="width:65px;text-align:right;flex-shrink:0">Vol %</div>'+
    '<div data-tt="atr_ma" style="width:90px;text-align:right;flex-shrink:0">ATR/50MA</div>'+
    '<div data-tt="vol_x" style="width:70px;text-align:right;flex-shrink:0">Vol ×</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">AvgVol $M</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">Today</div>';
  body.appendChild(hdr);
  wireTooltips(hdr);
```

- [ ] **Step 3: Browser-verify 50MA Scan tab**

Reload `index.html`. Click the **50MA Scan** tab.
- Hover `ATR` → ATR tooltip.
- Hover `Vol %` → volatility tooltip.
- Hover `ATR/50MA` → deviation tooltip.
- Hover `Vol ×` → volume tooltip.

- [ ] **Step 4: Commit**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" add index.html
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" commit -m "tooltips: wire 50MA Scan headers"
```

---

### Task 8: Wire Positions table (`ATR/50MA`, `Vol ×`)

**Files:**
- Modify: `index.html:1343–1351` (Positions header config) and `index.html:1352–1369` (the header render loop) and add a `wireTooltips` call after `tbl.appendChild(thead);` (currently line 1370).

- [ ] **Step 1: Locate the Positions header config**

Run:
```
grep -n "label:'ATR/50MA',col:'atrMA'" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
```
Expected: one match (currently line 1348).

- [ ] **Step 2: Add `tt` field to the two relevant header config entries**

Find lines 1348–1349:

```javascript
      {label:'% from Entry',col:'pctFromEntry'},{label:'ATR/50MA',col:'atrMA'},
      {label:'Vol ×',col:'volX'},...TFS.map(t=>({label:t==='today'?'Today':t,col:t})),
```

Replace with:

```javascript
      {label:'% from Entry',col:'pctFromEntry'},{label:'ATR/50MA',col:'atrMA',tt:'atr_ma'},
      {label:'Vol ×',col:'volX',tt:'vol_x'},...TFS.map(t=>({label:t==='today'?'Today':t,col:t})),
```

- [ ] **Step 3: Propagate `tt` to the rendered `<th>` in the loop**

Find the existing loop body (currently lines 1352–1369):

```javascript
    hdrs.forEach((h,i)=>{
      const th=document.createElement('th');
      th.style.textAlign=(i===0||i===1||i===2||i===3||i===hdrs.length-1)?'left':'right';
      if(h.col){
        th.style.cursor='pointer';
        const isActive=posSort.col===h.col;
        th.innerHTML=h.label+(isActive?(posSort.dir==='desc'?' ▼':' ▲'):'');
        th.style.color=isActive?'var(--blue)':'';
        th.addEventListener('click',()=>{
          posSort.dir=posSort.col===h.col&&posSort.dir==='desc'?'asc':'desc';
          posSort.col=h.col;
          renderPositions();
        });
      } else {
        th.textContent=h.label;
      }
      hrow.appendChild(th);
    });
```

Replace with:

```javascript
    hdrs.forEach((h,i)=>{
      const th=document.createElement('th');
      th.style.textAlign=(i===0||i===1||i===2||i===3||i===hdrs.length-1)?'left':'right';
      if(h.col){
        th.style.cursor='pointer';
        const isActive=posSort.col===h.col;
        th.innerHTML=h.label+(isActive?(posSort.dir==='desc'?' ▼':' ▲'):'');
        th.style.color=isActive?'var(--blue)':'';
        th.addEventListener('click',()=>{
          posSort.dir=posSort.col===h.col&&posSort.dir==='desc'?'asc':'desc';
          posSort.col=h.col;
          renderPositions();
        });
      } else {
        th.textContent=h.label;
      }
      if(h.tt) setTT(th,h.tt);
      hrow.appendChild(th);
    });
```

Note: `setTT` sets `th.tabIndex=0`, which is fine for these headers — they're already keyboard-clickable via the `addEventListener('click', ...)`, so making them focusable is consistent. The `cursor:pointer` from sort + `cursor:help` from `[data-tt]` will compete; the inline `cursor:pointer` set in the `if(h.col)` branch will win (inline styles beat selector rules unless `!important` is used). That's intentional — the column is primarily clickable-to-sort; the tooltip is secondary.

- [ ] **Step 4: Browser-verify Positions tab**

Reload `index.html`. Click the **Positions** tab. If you don't have positions, the table won't render — in that case, add a test position via the "Add Position" form (any ticker, e.g. AAPL, any entry price and shares). Then:

- Hover `ATR/50MA` header → deviation tooltip.
- Hover `Vol ×` header → volume tooltip.
- Click `ATR/50MA` header → table sorts by that column, tooltip dismisses.
- Hover `Ticker` or `Theme` headers → no tooltip.

- [ ] **Step 5: Commit**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" add index.html
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" commit -m "tooltips: wire Positions table headers"
```

---

### Task 9: Wire Morning Gaps headers (`ATR/50MA`, `Vol ×`)

**Files:**
- Modify: `index.html:1772–1782` (Morning Gaps header `innerHTML` block in `renderGaps()`).

- [ ] **Step 1: Locate the `innerHTML` block**

Run:
```
grep -n "width:90px;text-align:right;flex-shrink:0\">Gap %<" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
```
Expected: one match (currently line 1775).

- [ ] **Step 2: Add `data-tt` attributes and `wireTooltips` call**

Find the existing block (currently lines 1772–1782):

```javascript
  hdr.innerHTML=
    '<div style="width:60px;flex-shrink:0">Ticker</div>'+
    '<div style="flex:1">Theme</div>'+
    '<div style="width:90px;text-align:right;flex-shrink:0">Gap %</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">Price</div>'+
    '<div style="width:80px;text-align:right;flex-shrink:0">ATR/50MA</div>'+
    '<div style="width:70px;text-align:right;flex-shrink:0">Vol ×</div>'+
    '<div style="width:80px;text-align:right;flex-shrink:0">AvgVol $M</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">1W</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">1M</div>';
  body.appendChild(hdr);
```

Replace with:

```javascript
  hdr.innerHTML=
    '<div style="width:60px;flex-shrink:0">Ticker</div>'+
    '<div style="flex:1">Theme</div>'+
    '<div style="width:90px;text-align:right;flex-shrink:0">Gap %</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">Price</div>'+
    '<div data-tt="atr_ma" style="width:80px;text-align:right;flex-shrink:0">ATR/50MA</div>'+
    '<div data-tt="vol_x" style="width:70px;text-align:right;flex-shrink:0">Vol ×</div>'+
    '<div style="width:80px;text-align:right;flex-shrink:0">AvgVol $M</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">1W</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">1M</div>';
  body.appendChild(hdr);
  wireTooltips(hdr);
```

- [ ] **Step 3: Browser-verify Morning Gaps tab**

Reload `index.html`. Click the **Morning Gaps** tab. (If the section is empty for the current session, the header may not render — adjust the "Min gap %" threshold in the controls to a low value like 1% so at least the header bar shows.)

- Hover `ATR/50MA` → deviation tooltip.
- Hover `Vol ×` → volume tooltip.

- [ ] **Step 4: Commit**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" add index.html
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" commit -m "tooltips: wire Morning Gaps headers"
```

---

### Task 10: Wire Extension scan headers (`ATR/50MA`, `Vol %`, `Vol ×`)

**Files:**
- Modify: `index.html:2007–2018` (Extension header `innerHTML` block in `buildExtension()` / `renderExtension()`).

- [ ] **Step 1: Locate the `innerHTML` block**

Run:
```
grep -n "width:100px;text-align:right;flex-shrink:0\">ATR/50MA<" "C:/Users/Tim Parker/Projects/Pulse Scanner/index.html"
```
Expected: one match (currently line 2010).

- [ ] **Step 2: Add `data-tt` attributes and `wireTooltips` call**

Find the existing block (currently lines 2007–2018):

```javascript
  hdr.innerHTML=
    '<div style="width:60px;flex-shrink:0">Ticker</div>'+
    '<div style="flex:1">Theme</div>'+
    '<div style="width:100px;text-align:right;flex-shrink:0">ATR/50MA</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">Price</div>'+
    '<div style="width:70px;text-align:right;flex-shrink:0">Vol %</div>'+
    '<div style="width:70px;text-align:right;flex-shrink:0">Vol ×</div>'+
    '<div style="width:80px;text-align:right;flex-shrink:0">AvgVol $M</div>'+
    '<div style="width:80px;text-align:right;flex-shrink:0">Today</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">1W</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">1M</div>';
  body.appendChild(hdr);
```

Replace with:

```javascript
  hdr.innerHTML=
    '<div style="width:60px;flex-shrink:0">Ticker</div>'+
    '<div style="flex:1">Theme</div>'+
    '<div data-tt="atr_ma" style="width:100px;text-align:right;flex-shrink:0">ATR/50MA</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">Price</div>'+
    '<div data-tt="vol_pct" style="width:70px;text-align:right;flex-shrink:0">Vol %</div>'+
    '<div data-tt="vol_x" style="width:70px;text-align:right;flex-shrink:0">Vol ×</div>'+
    '<div style="width:80px;text-align:right;flex-shrink:0">AvgVol $M</div>'+
    '<div style="width:80px;text-align:right;flex-shrink:0">Today</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">1W</div>'+
    '<div style="width:75px;text-align:right;flex-shrink:0">1M</div>';
  body.appendChild(hdr);
  wireTooltips(hdr);
```

- [ ] **Step 3: Browser-verify Extension tab**

Reload `index.html`. Click the **Extension** tab.

- Hover `ATR/50MA` → deviation tooltip.
- Hover `Vol %` → volatility tooltip.
- Hover `Vol ×` → volume tooltip.

- [ ] **Step 4: Commit**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" add index.html
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" commit -m "tooltips: wire Extension scan headers"
```

---

### Task 11: Full end-to-end browser verification

**Files:** None modified — verification only.

- [ ] **Step 1: Hard-reload `index.html` in Chrome/Edge**

Cmd+Shift+R (Mac) or Ctrl+Shift+R (Win) to bypass cache. Enter Polygon key, click Sync, wait for completion.

- [ ] **Step 2: Walk every tooltipped header in every view**

Verify each of the following shows the correct popover on hover and dismisses on mouse-out:

- **Themes tab (default):** `Vol %`, `RS`.
- **Sectors tab:** `ATR%`, `RS`.
- **Drill view** (click any theme): `Vol ×`, `ATR`, `Vol %`, `ATR/50MA`.
- **50MA Scan tab:** `ATR`, `Vol %`, `ATR/50MA`, `Vol ×`.
- **Positions tab:** `ATR/50MA`, `Vol ×` (add a test position if needed).
- **Morning Gaps tab:** `ATR/50MA`, `Vol ×`.
- **Extension tab:** `ATR/50MA`, `Vol %`, `Vol ×`.

- [ ] **Step 3: Keyboard accessibility check**

Click somewhere outside any header, then press Tab repeatedly. When focus lands on a tooltipped header, the popover should appear and the header should show a dotted focus outline. Press Tab again — popover hides.

- [ ] **Step 4: Viewport-edge clipping check**

Resize the browser window so a tooltipped header sits near the bottom edge. Hover it — the popover should flip to appear above the header instead of below.

Then resize so a header sits near the left edge of the viewport. Hover — popover should be clamped so it doesn't bleed off the screen.

- [ ] **Step 5: Sort interaction check**

In the drill view, hover `Vol ×` → popover shows. Click the header → table sorts by Vol × AND popover dismisses immediately. Hover again → popover reappears. Repeat for Positions table.

- [ ] **Step 6: Re-render idempotency check**

Drill into a theme, sort by Vol ×, click "Back", drill into a different theme, hover Vol ×. Popover should appear exactly once (not twice or stacked). This confirms `data-tt-wired` is preventing duplicate listener registration on re-render.

- [ ] **Step 7: Negative check — non-tooltipped columns are untouched**

Hover `Ticker`, `Price`, `Today`, `1W`, `1M`, `3M`, `YTD`, `#` in any view. No popover should appear. Cursor should remain `default` (not `help`).

- [ ] **Step 8: Console-error check**

DevTools console (F12) → Console tab. Refresh. Click through all tabs. Verify no red errors logged.

- [ ] **Step 9: If everything passes — proceed to Task 12. If anything fails — fix before moving on.**

Common failure modes:
- Tooltip never shows → check `setTT`/`wireTooltips` was actually called for the failing site; check `TOOLTIPS[key]` exists.
- Tooltip shows wrong text → check the `data-tt` value or third `setTT` arg matches a `TOOLTIPS` map key.
- Tooltip in wrong position → likely the target element has unusual transform or z-index; `position:fixed` + `getBoundingClientRect` should usually be robust, but a parent with `transform` can break `position:fixed`. Investigate the parent chain.
- Duplicate popovers → `data-tt-wired` guard failed; verify `setTT` sets it.

---

### Task 12: Push fork and open the PR

**Files:** None modified — git/GitHub only.

- [ ] **Step 1: Confirm clean working tree on `feature/metric-tooltips`**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" status
```
Expected: `On branch feature/metric-tooltips`, working tree clean. If not clean, commit or discard before continuing.

- [ ] **Step 2: Push the feature branch to the fork**

Run:
```
git -C "C:/Users/Tim Parker/Projects/Pulse Scanner" push -u origin feature/metric-tooltips
```
Expected: branch published to `autumnalcity/Pulse-Scanner`. `origin` was already set to the fork during onboarding.

- [ ] **Step 3: Open the PR against upstream main**

Run:
```
gh pr create --repo AmandaCharles00/Pulse-Scanner --base main --head autumnalcity:feature/metric-tooltips --title "Add metric column tooltips" --body "$(cat <<'EOF'
## Summary

Adds hover/focus tooltips on the column headers whose math isn't obvious from the label, defining each metric in plain language alongside its literal formula.

Five metrics get tooltips:

- `VOL ×` — today's volume ÷ yesterday's volume
- `ATR` — 14-day Wilder smoothing of daily true range, in dollars
- `VOL %` / `ATR%` — ATR as a percent of price (typical volatility)
- `ATR/50MA` — (price − 50-day MA) ÷ ATR (extension from the 50-day MA, in ATR units)
- `RS` — relative strength rank, 1–98, across 5 timeframes

The motivating issue: `VOL ×` and `VOL %` both start with "VOL" but measure different things (volume vs. volatility). New users intuitively read `VOL × = 1.5x` as a volatility multiple of `VOL %`, which it isn't. Tooltips disambiguate without renaming any columns.

## Implementation

Single-file change to `index.html`. ~50 lines of vanilla JS + ~15 lines of CSS added, plus a one-line tooltip wiring at each of seven render sites. No new files, no build step change, no dependencies. The app still runs by opening `index.html` directly in a browser.

The helper exposes `setTT(el, key)` for imperative DOM sites and `wireTooltips(root)` for `innerHTML`-built sites. It's idempotent via a `data-tt-wired` guard so re-renders don't stack listeners.

## Test plan

- [ ] Open `index.html` in Chrome/Edge, enter Polygon key, click Sync.
- [ ] Hover each tooltipped header in every view:
  - Themes tab: `Vol %`, `RS`
  - Sectors tab: `ATR%`, `RS`
  - Drill view: `Vol ×`, `ATR`, `Vol %`, `ATR/50MA`
  - 50MA Scan: `ATR`, `Vol %`, `ATR/50MA`, `Vol ×`
  - Positions: `ATR/50MA`, `Vol ×`
  - Morning Gaps: `ATR/50MA`, `Vol ×`
  - Extension: `ATR/50MA`, `Vol %`, `Vol ×`
- [ ] Tab through with the keyboard — popover appears on focus.
- [ ] Resize window so a header sits near the bottom edge — popover flips above.
- [ ] Click a tooltipped header — table sorts AND popover dismisses immediately.
- [ ] Hover a non-tooltipped column (Ticker, Price, Today, etc.) — no popover.

Spec: [`docs/superpowers/specs/2026-05-23-metric-tooltips-design.md`](docs/superpowers/specs/2026-05-23-metric-tooltips-design.md)
Plan: [`docs/superpowers/plans/2026-05-23-metric-tooltips.md`](docs/superpowers/plans/2026-05-23-metric-tooltips.md)
EOF
)"
```

Expected: `gh` prints a PR URL. Capture it for Task 13.

- [ ] **Step 4: Verify the PR opened correctly**

Run:
```
gh pr view --repo AmandaCharles00/Pulse-Scanner <PR-number>
```
Expected: PR exists, points from `autumnalcity:feature/metric-tooltips` → `AmandaCharles00:main`, title and body look right.

---

### Task 13: Generate handoff prompt for Amanda's Claude Code

**Files:** None modified — output is a single message Tim DMs to Amanda.

- [ ] **Step 1: Produce the handoff prompt as a single fenced block**

Print the following block to the user. It is self-contained — Amanda pastes it into her Claude Code session and her CC takes it from there. Substitute the actual PR number and URL from Task 12.

```
Hey Amanda! Tim sent you a Pull Request on Pulse-Scanner. Can you help me review and decide whether to merge it?

PR: https://github.com/AmandaCharles00/Pulse-Scanner/pull/<PR_NUMBER>
Title: Add metric column tooltips
Author: autumnalcity (Tim)

What I'd like you to do, step by step:

1. Tell me in 1-2 sentences what this PR actually changes — the user-facing behavior, not the technical detail. (Read the PR body and the diff to summarize.)

2. Pull the PR down so I can test it locally. Run:
     gh pr checkout <PR_NUMBER> --repo AmandaCharles00/Pulse-Scanner
   That switches my working copy to Tim's branch.

3. Open index.html in my browser (Chrome or Edge — same as I normally do). Tell me what to look for: hover each of the column headers Tim added tooltips to, and confirm the little popover appears with a definition and a formula. Walk me through each one — Themes tab, Sectors tab, drill into a theme, then 50MA Scan, Positions, Morning Gaps, Extension. Show me the list of headers to hover in each.

4. After I've tested it, ask me how it looks. If I say "looks good", merge the PR with:
     gh pr merge <PR_NUMBER> --repo AmandaCharles00/Pulse-Scanner --merge
   Then switch back to my normal main branch:
     gh repo set-default AmandaCharles00/Pulse-Scanner
     git checkout main
     git pull

5. If I say "something's off", help me write a comment on the PR describing what I saw, using:
     gh pr comment <PR_NUMBER> --repo AmandaCharles00/Pulse-Scanner --body "..."

6. Don't merge until I explicitly say to. Don't delete the branch. Don't change anything else in the repo.
```

- [ ] **Step 2: Deliver the prompt to Tim**

Print it in chat in a single fenced code block so he can copy it to Amanda in one paste. Confirm the PR number and URL are filled in. Mark Task 13 complete.

---

## Self-review checklist

After implementation completes, verify:

- [ ] Every header listed in the spec coverage table actually has a tooltip in the running app.
- [ ] No header outside that list has a tooltip (no over-application).
- [ ] DevTools console clean (no errors) on every tab.
- [ ] Diff against `upstream/main` is ~100 lines, single file, no new files.
- [ ] Spec doc and plan doc are committed and referenced from the PR body.
