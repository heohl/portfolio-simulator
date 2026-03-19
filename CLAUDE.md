# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Static two-page web app (no build step, no framework, no backend). German-language UI. Deployed via GitHub Pages at `https://heohl.github.io/portfolio-simulator/`.

- **index.html** — Monte Carlo Investment Planner (accumulation phase)
- **entnahme.html** — Entnahmerechner / Withdrawal Calculator (decumulation phase)

Only external dependency: Chart.js 4.4.0 via jsDelivr CDN.

## Simulation Model

Both pages use **geometric Brownian motion** with monthly steps:

- `drift = (μ - 0.5σ²) × (1/12)`
- `sigSqrtDt = σ × √(1/12)`
- Monthly step: `v = (v + savings) × exp(drift + sigSqrtDt × randNorm())` (accumulation)
- Monthly step: `v = max(0, v × exp(...) - withdrawal)` (withdrawal — returns first, then subtract)
- Random normals via Box-Muller transform

Paths are stored as `Float64Array[]` in the global `lastPaths`.

## Rendering Architecture

Two overlaid canvases inside `.canvas-wrap` (fixed 420px height):

1. **`bgCanvas`** — absolutely positioned, `pointer-events:none`. Draws spaghetti paths manually in `drawSpaghettiCanvas()`. Must be resized to `wrap.clientWidth/Height` before drawing. Max 300 paths rendered; alpha scales dynamically. In entnahme.html, ruined paths (end value = 0) are drawn red.
2. **`mainChart`** — Chart.js instance (z-index: 1). Renders mean, percentile band, and reference lines. Uses `animation: false` and `update('none')` everywhere — animations are handled separately.

**Reveal animation**: on new simulations, `clip-path: inset(0 100% 0 0)` → `inset(0 0% 0 0)` over 1.5s is applied to `canvas-wrap` after both canvases are rendered.

Spaghetti is drawn in a `setTimeout(..., 50)` after `chart.update()` to ensure `chart.chartArea` is populated.

## Chart.js Configuration Gotchas

**X-axis ticks**: `autoSkip: false` was causing browser freezes (600+ grid lines for 50-year periods). Fixed by filtering in `afterBuildTicks`:
```js
afterBuildTicks(axis) {
  const step = chartYearStep * 12;
  axis.ticks = axis.ticks.filter(t => t.value % step === 0);
}
```
`chartYearStep` is a **global variable** (not a closure) so the callback — which is only created once at chart instantiation — always reads the current value on subsequent simulations.

Year step logic: ≤10y → 1, ≤25y → 2, ≤40y → 5, >40y → 10.

## Key Global State

| Variable | Purpose |
|---|---|
| `P` | Current parameter object (`start`, `savings`/`withdraw`, `ret`, `vol`, `years`, `sims`) |
| `lastPaths` | All simulation results as `Float64Array[]` |
| `chart` | Chart.js singleton, created once and reused |
| `chartYearStep` | Shared with Chart.js tick callback — must be global |
| `transferValue` | Selected end-value for cross-page transfer (index.html only) |

## Cross-Page Data Transfer

index.html → entnahme.html via URL params:
```
entnahme.html?start=<median|mean|p95>&ret=<percent>&vol=<percent>
```
entnahme.html reads params on `load`, updates `P`, and shows the import banner. Only `start`, `ret`, and `vol` are transferred (not `years`/`sims`).

## Scenario Persistence (localStorage)

- `mc_planner_scenarios` — index.html scenarios
- `mc_entnahme_scenarios` — entnahme.html scenarios

Each entry: `{ id: Date.now().toString(), name, params: {...P}, savedAt: "DD.MM.YYYY" }`. Stored as JSON array, newest first. `applyScenario(id)` syncs all UI controls and calls `runSim()`.

## CSS Design Tokens

Defined in `:root` on both pages (both files are self-contained):
- `--accent: #6c63ff` (purple) — index.html primary color
- `--orange: #f97316` — entnahme.html primary color (replaces accent for buttons, sliders, borders)
- `--accent2: #00d4aa` (cyan) — mean line and positive highlights on both pages
- Layout: fixed 310px sidebar + flex-1 main, header 57px, `height: calc(100vh - 57px)` on `.layout`

## Deployment

```bash
git add index.html entnahme.html
git commit -m "Description"
git push
```
GitHub Pages updates automatically within ~1 minute. No build step needed.
