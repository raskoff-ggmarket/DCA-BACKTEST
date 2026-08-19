# CLAUDE.md

Guidance for AI assistants (and humans) working in this repository.

## What this project is

**DCA-BACKTEST** is a client-side, single-file **backtesting simulator** for
crypto trading bots, delivered as an installable PWA. It reproduces the behavior
of 3Commas-style **DCA (Dollar-Cost Averaging) bots** and **Grid bots** against
historical 1-minute candle data, entirely in the browser — there is no server,
no backend, and no build step.

- **UI language is Turkish.** All labels, comments, toasts, and commit messages
  are in Turkish. Match this convention: write new UI strings and code comments
  in Turkish, keep number/date formatting locale-aware (`tr-TR` for display,
  `en-US` for money).
- **Deployed on GitHub Pages** under the path `/DCA-BACKTEST/` (see
  `manifest.json` `start_url`/`scope`). Pushing to `main` publishes.
- **Audience:** mobile-first (max-width 480px layout, portrait), with a
  responsive desktop mode (multi-column at ≥820px / ≥1140px).

## Repository layout

```
index.html              The entire application — HTML + CSS + JS in one file (~3500 lines)
dca_strategy.pine       TradingView Pine v5 replica of the DCA engine (for on-chart verification)
manifest.json           PWA manifest (name, icons, theme colors, scope)
.github/workflows/
  version-bump.yml       CI: auto-increments the build number on every push to main
bitcoin_1m_*_part{1..6}of6.json   Bundled sample data: BTC 1m candles 2022→2026, exported in ≤10MB parts
android-chrome-*.png, icon.png    PWA / favicon icons
.gitignore              Only ignores node_modules/ (there is no node project; vestigial)
```

There is **no `package.json`, no bundler, no test runner, no framework.** The app
is plain ES5-style vanilla JavaScript (`var`, `function`, no modules) written to
run directly in the browser. Do not introduce a build toolchain unless explicitly
asked — the zero-dependency, single-file nature is the core design constraint.

## How to run / develop

- **Run locally:** serve the directory over HTTP (e.g. `python3 -m http.server`)
  and open `index.html`. A plain `file://` open mostly works but OPFS/persistent
  storage and the manifest behave better over HTTP.
- **No install, no compile.** Edit `index.html` and refresh.
- **Do not run `playwright install`** if you use the pre-installed Chromium for
  manual verification (see environment notes).

## Architecture of `index.html`

The file is organized top-to-bottom as: `<head>`/CSS → HTML markup (splash,
header, panels) → one big `<script>` (starts ~line 641) → init calls at the very
bottom. Key regions in the script:

- **Constants:** `COINS` (coin list + data `src` per coin), `PERS` (period
  presets), `RSI_TFS`, `INSTANT_BAR`.
- **State:** `DEFAULTS` → `S` (the live settings object). `loadState()` /
  `saveState()` persist `S` to `localStorage` under key **`dca_v2`** and include
  backward-compat migrations (e.g. old `smaFilter` → `entryMaMode`, single
  `commission` → `makerFee`/`takerFee`). When you add a setting, add it to
  `DEFAULTS`, add a stepper config to `STEPPERS` if it's numeric, and preserve
  backward compat in `loadState()`.
- **Candle storage (three tiers):**
  - `localStorage` — small flags (`histDone_*`) and settings.
  - **IndexedDB** database `dca_cache`, store `c`, keyPath `["k","ts"]` — the
    primary candle cache (`openDB`, `cGet`, `cPut`, `cPutBatched`, `cClearAll`).
  - **OPFS** (Origin Private File System) — durable fallback / seed
    (`opfsPut`, `opfsGet`, `opfsRestore`). On startup `opfsRestore()` repopulates
    IndexedDB from OPFS if the DB is empty.
- **Data fetching:** `fetchOKXCandles`, `_fetchHistorical`, `_fetchNew`,
  `loadCandleRange`, `fetchCandlesRun`. Two live sources, chosen per-coin by
  `COINS[].src`:
  - **Bybit** spot: `https://api.bybit.com/v5/market/kline?category=spot...`
  - **OKX** history: `https://www.okx.com/api/v5/market/history-candles...`
  Raw candles are normalized by `parseRawCandles` into
  `{ts, price(close), high, low, date, time}`, array index `[0]=ts,1=o,2=h,3=l,4=c`.
- **Import/Export:** `exportCandles` splits cached candles into ≤10MB JSON parts
  named `<key>_<from>_<to>_part{N}of{M}.json` (this is exactly how the bundled
  `bitcoin_1m_*` files were produced). `importCandles` accepts `.json` and `.csv`.
- **DCA engine:** `backtest(prices, rsiArr)` — the core simulator. Models base
  order + martingale safety orders (`priceStep`/`stepMult` for deviation,
  `safeAmt`/`amtMult` for volume), TP, trailing TP (TTP), stop-loss modes,
  leverage + liquidation (`updateLiqPrice`), MA entry/SO filters
  (`_entryOk`/`_soOk` via `calcDailySMA`), RSI entry (`calcRSI`), profit mode
  (`quote` vs `base`), and profit reinvest (`applyReinvest`). Fees: **maker** rate
  for limit fills (safety orders, TP), **taker** rate for market fills (base order,
  TTP, SL). `closeDealAt(price,date,why)` handles TP/TTP/SL/LIQ exits.
- **Grid engine:** `gridBacktest` / `gridBacktestWithParams`, `_gridLevels`
  (arith vs geometric spacing), `gridAutoFillRange`. Toggled via `S.botType`
  (`"dca"` | `"grid"`).
- **Optimizer (Monte Carlo):** `_sampleParams` / `runOptimizer` for DCA and
  `_sampleGridParams` / `runGridOptimizer` for Grid. Randomly samples parameter
  sets, scores each with `_optScore` / `_optScoreGrid`, and surfaces named
  profiles (e.g. "Sık İşlem"). `backtestWithParams` runs the engine with a
  temporary override of `S`.
- **Rendering / navigation:** `render`, `renderGrid`, `run`, panel switches
  (`goResult`, `goSettings`, `goTab`, `setTab`, `setBotType`), and many
  `build*`/`refresh*` UI builders. Navigation is panel-swap within the single
  page; `currentTab` tracks the active view.
- **Init:** the bottom of the script (after `closeSplash`) calls `loadState()`,
  builds all UI, requests persistent storage, and restores caches.

## `dca_strategy.pine`

A Pine Script v5 `strategy()` that mirrors the DCA engine so results can be
cross-checked on a TradingView chart. **If you change DCA math in `index.html`
(order sizing, TP/TTP, reinvest, fees), keep this file in sync** — its header and
input defaults are meant to match the app's `DEFAULTS`.

## Bundled sample data

`bitcoin_1m_2022-01-01_2026-06-23_part{1..6}of6.json` are BTC 1-minute candles
committed so the app has data without a network fetch. They are **not referenced
directly by `index.html`** — they are import files (produced by `exportCandles`)
that a user loads via the "İçe Aktar" (Import) button, or seed data for OPFS.
They are large (~10MB each); avoid rewriting or re-committing them unnecessarily.

## Build numbering & CI

`.github/workflows/version-bump.yml` runs on every push to `main`. It reads the
`data-build="N"` attribute on the `#appVer` span in `index.html`, increments it,
updates both the attribute and the visible `build N` text, and commits
`chore: build N [version-bump]`. Notes:

- **Do not manually bump `data-build`** — CI owns it. The `[version-bump]` tag in
  the message prevents the workflow from recursing on its own commit.
- Expect an automatic follow-up commit after any push to `main`; this is normal.
- The many `chore: build NN [version-bump]` / `Add files via upload` commits in
  history come from this flow and from GitHub web uploads.

## Conventions & guidance for changes

- **Keep it a single self-contained file.** No external runtime dependencies
  except the Google Fonts stylesheet already linked. No frameworks, no modules,
  no npm.
- **Style:** compact ES5 (`var`, function declarations, short helper names like
  `f`, `fu`, `fp`, `upd`). Match the terse, dense style of surrounding code and
  its Turkish comments (section banners use `═══` rules).
- **Money vs display formatting:** `f`/`fu`/`fp` format numbers in `en-US`; dates
  render in `tr-TR`. Keep this split.
- **Storage keys are contractual:** `localStorage["dca_v2"]`, IndexedDB
  `dca_cache`/store `c`, OPFS `<key>.json`. Changing them silently drops user
  data — migrate instead.
- **Fees semantics matter:** maker for limit orders, taker for market orders —
  preserve this when touching `backtest`.
- **When adding a coin:** append to `COINS` with the correct `src`
  (`"bybit"` or the OKX branch) and matching symbol; verify the source actually
  lists it.
- **Verify manually in a browser** for UI/engine changes — there is no test
  suite. Consider adding a quick Node/console reproduction only as scratch work,
  not committed unless asked.

## Git workflow

- Development branch for this line of work: `claude/claude-md-docs-hk5q80`.
- Push with `git push -u origin <branch>`; do **not** push to `main` directly
  unless explicitly told — pushing to `main` triggers the version-bump workflow.
- Do not open a PR unless explicitly asked.
- Do not include model identifiers in commits, comments, or any pushed artifact.
