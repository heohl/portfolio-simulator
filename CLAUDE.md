# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Static three-page web app (no build step, no framework, no backend). German-language UI. Deployed via GitHub Pages at `https://heohl.github.io/portfolio-simulator/`.

- **index.html** — Monte Carlo Investment Planner (accumulation phase)
- **fire.html** — FIRE Rechner (Financial Independence, Retire Early — two-phase simulation)
- **entnahme.html** — Entnahmerechner / Withdrawal Calculator (decumulation phase)

Only external dependency: Chart.js 4.4.0 via jsDelivr CDN.

## Simulation Model

All pages use **geometric Brownian motion** with monthly steps:

- `drift = (μ - 0.5σ²) × (1/12)`
- `sigSqrtDt = σ × √(1/12)`
- Monthly step: `v = (v + savings) × exp(drift + sigSqrtDt × randNorm())` (accumulation)
- Monthly step: `v = max(0, v × exp(...) - withdrawal)` (withdrawal — returns first, then subtract)
- Random normals via Box-Muller transform

Paths are stored as `Float64Array[]` in the global `lastPaths`.

## Rendering Architecture

Two overlaid canvases inside `.canvas-wrap` (fixed 420px height):

1. **`bgCanvas`** — absolutely positioned, `pointer-events:none`. Draws spaghetti paths manually in `drawSpaghettiCanvas()`. Must be resized to `wrap.clientWidth/Height` before drawing. Max 300 paths rendered; alpha scales dynamically.
   - entnahme.html: ruined paths (end value = 0) drawn red
   - fire.html: three layers — grey (never reached FIRE), red (reached FIRE then bankrupt), gold (reached FIRE and survived). Also draws vertical dashed lines at P25/P50/P75 of the FIRE month distribution.
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

| Variable | Purpose | Pages |
|---|---|---|
| `P` | Current parameter object | all |
| `lastPaths` | All simulation results as `Float64Array[]` | all |
| `chart` | Chart.js singleton, created once and reused | all |
| `chartYearStep` | Shared with Chart.js tick callback — must be global | all |
| `transferValue` | Selected end-value for cross-page transfer | index.html only |
| `lastFireStat` | Computed FIRE statistics, kept between redraws | fire.html only |

`P` keys differ per page:
- **index.html**: `start`, `savings`, `ret`, `vol`, `years`, `sims`
- **entnahme.html**: `start`, `withdraw`, `ret`, `vol`, `years`, `sims`
- **fire.html**: `start`, `savings`, `expenses`, `swr`, `retAcc`, `volAcc`, `retWd`, `volWd`, `years`, `sims`

## FIRE Calculator (fire.html)

### Two-Phase Simulation
Each path simulates two phases sequentially within a single `simulate(params, fireNum)` call:

1. **Accumulation**: `v = (v + savings) × exp(driftAcc + sigSqrtAcc × randNorm())` — uses `retAcc`/`volAcc`
2. **Withdrawal**: triggered the month `v >= fireNum`; `v = max(0, v × exp(driftWd + sigSqrtWd × randNorm()) - expenses)` — uses separate `retWd`/`volWd`

`simulate()` returns `{ paths, months, fireMonths }` where `fireMonths[s]` is the month path `s` crossed the FIRE number (`Infinity` if never).

### FIRE Number
```
fireNum = expenses × 12 / (swr / 100)
```
Displayed live in the sidebar as the user adjusts `expenses` or `swr`.

### FIRE Stats (`computeFireStats`)
Takes `(paths, months, fireMonths)` — note `fireMonths` comes from `simulate()`, not re-computed. Returns:
- `pFire` — share of paths that reached FIRE
- `pRuin` — share of all paths ending at 0
- `pRuinGivenFire` — share of FIRE-reaching paths that then went bankrupt
- `pSuccess = pFire × (1 - pRuinGivenFire)` — reached FIRE and survived
- `p25`, `p50`, `p75` — FIRE month percentiles (over paths that achieved FIRE)

### Invested Capital Line
`investedCapital(start, savings, months, stopMonth)` flattens at `stopMonth` (passed as `lastFireStat.p50`) — savings stop at the median FIRE month. If no path reached FIRE (`p50 = Infinity`), the line grows the full horizon.

**Important**: `computeFireStats` must be called before `investedCapital` in `runSim()` so that `p50` is available.

## Cross-Page Data Transfer

index.html → entnahme.html via URL params:
```
entnahme.html?start=<median|mean|p95>&ret=<percent>&vol=<percent>
```
entnahme.html reads params on `load`, updates `P`, and shows the import banner. Only `start`, `ret`, and `vol` are transferred (not `years`/`sims`).

fire.html is currently standalone — no incoming or outgoing URL transfers.

## Scenario Persistence (localStorage)

- `mc_planner_scenarios` — index.html scenarios
- `mc_fire_scenarios` — fire.html scenarios
- `mc_entnahme_scenarios` — entnahme.html scenarios

Each entry: `{ id: Date.now().toString(), name, params: {...P}, savedAt: "DD.MM.YYYY" }`. Stored as JSON array, newest first. `applyScenario(id)` syncs all UI controls and calls `runSim()`.

## CSS Design Tokens

Defined in `:root` on each page (all files are self-contained):
- `--accent: #6c63ff` (purple) — index.html primary color
- `--fire: #f59e0b` (amber/gold) — fire.html primary color
- `--orange: #f97316` — entnahme.html primary color
- `--accent2: #00d4aa` (cyan) — mean line and positive highlights on all pages
- Layout: fixed 310px sidebar + flex-1 main, header 57px, `height: calc(100vh - 57px)` on `.layout`

## Deployment

```bash
git add index.html fire.html entnahme.html
git commit -m "Description"
git push
```
GitHub Pages updates automatically within ~1 minute. No build step needed.
