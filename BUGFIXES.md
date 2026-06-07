# Bug-fix log

## Round 7 — robustness audit (edge-case fuzzing)
Fuzzed the recently-added calibration/lookback paths and analysis modules with
boundary inputs. Bugs found & fixed:

1. **🔴 `tetherDom` crash on bad data.** It checked `usdtSeries.length >= 8` but
   not the *values*; a single `undefined`/`NaN`/zero in the series threw
   `Cannot read properties of undefined (reading 'toFixed')`, killing the whole
   signal. **Fix:** coerce to number and guard `isFinite` / non-zero baseline →
   returns a neutral "Data unavailable" instead of crashing. (Also fixed an
   inconsistent label.)

2. **🟠 Calibration could run O(n²) if caller omitted `lookback`.** The bounded-
   lookback optimisation only applied when the caller passed `lookback`; calling
   `calibrate` with just `maxCalibBars` (or nothing) re-introduced the O(n²)
   slowdown. **Fix:** `calibrate` now auto-defaults a safe `lookback`
   (≥ 200×htf, covering EMA200) for every internal backtest. Verified identical
   results to an explicit equal lookback.

3. **🟠 `maxCalibBars` below `warmup` produced an empty sweep.** A tiny
   `maxCalibBars` (e.g. 100 < warmup 210) left the parameter sweep with no
   tradable bars while the reported stats silently came from the full run.
   **Fix:** floor `maxCalibBars` at `warmup + 300` so the sweep always sees a
   real trade sample.

Verified non-bugs: live full-history signal == lookback-window signal (identical
side/net/levels), 44k-bar `buildSignal` stays finite & fast (0.45s), all swing/
FVG/OB indices stay in-bounds, app.js loads with no top-level errors.

## Round 6 — 5-year depth on intraday/scalp + O(n²) performance bug
Extending 5-year history to the 4H and 1H profiles surfaced a critical
performance bug and required data-fetch + calibration changes.

1. **🔴 O(n²) backtest loop.** The walk-forward did `daily.slice(0, i+1)` and
   re-`resample()`/re-analysed the *entire* history at every bar. Fine for ~1,800
   daily candles, but a 4H/5-year set (~11k candles) took **23s per backtest**
   and 1H (~44k) was unusable. **Fix:** a bounded **rolling-lookback window**
   (`opts.lookback`, ≥ 200×htf) makes each iteration O(1) → O(n) overall. Proven
   **loss-less** (identical trades/equity vs full history). 4H single backtest:
   23s → ~3s.

2. **🔴 Delta 4000-candle cap (silent truncation).** Like the earlier Binance
   bug, Delta returns only the most-recent ~4000 candles when over-requested, so
   intraday windows were silently truncated. **Fix:** the Delta fetcher now
   **paginates** forward in ≤4000-candle pages (verified: 10,950 4h / 43,800 1h
   candles, both ends correct, back to 2021).

3. **🟠 Calibration too slow for large sets.** Even at O(n), a 216-cell grid over
   44k candles would take many minutes. **Fix:** calibration auto-selects a
   **coarse 36-cell grid** for >4000-bar datasets and sweeps a capped recent
   window (`maxCalibBars`, default 6000) for speed, while the **final reported
   backtest + out-of-sample check still cover the full 5 years**. 4H/1H
   calibration now completes in ~1.5 min.

Note: out-of-sample validation now honestly shows the price-action strategy
**does not generalise to 4H/1H** with current defaults (OOS PF < 1), whereas it
holds up on the daily timeframe. That transparency is by design.


## Round 5 — 5-year backtest audit
Extended the swing backtest window to ~5 years (1830 daily candles) and audited
the analysis + backtest engine against the longer history.

1. **🔴 Silent data truncation (Binance, >1000 bars).** Binance.US caps each
   klines call at 1000 candles. The old single-call fetcher requested a
   5-year-ago `startTime` and received only the *oldest* 1000 candles
   (2021→2024), silently dropping the most recent ~2 years. **Fix:** the Binance
   fetcher now paginates forward page-by-page until it reaches "now" (verified:
   1830/1830 candles, both ends correct).

2. **🔴 Macro EMA filter disabled for ~half the backtest.** `trendEMA` used a
   fixed EMA200, which is `null` on the HTF (resampled) series until ~daily bar
   1000 — so the "50 vs 200 EMA" macro trend filter was silently off for the
   first ~2.7 years. **Fix:** the long-EMA period now adapts to available
   history (200 with full data, ≥80 on short/resampled series) so the filter
   always contributes.

3. **🟠 Calibration overfitting (no out-of-sample test).** Calibration picked
   the single best in-sample parameter set with no forward validation, hiding
   curve-fitting. **Fix:** walk-forward validation — optimise on the first 70%
   of history, then re-score the winner on the unseen last 30%. Both windows are
   reported in the UI so degradation is visible (in-sample PF 1.64 → OOS 1.45 on
   the 5-yr Delta run).

4. **🟠 Same-bar stop+TP always counted as a loss.** When one bar's range
   engulfed both the stop and a take-profit, the old code always assumed the
   stop hit first (pessimistic bias). **Fix:** if both are in-range, the level
   nearer the bar's open is assumed first; a guard prevents any double-close.

5. **🟡 Reaffirmed:** USDT-dominance history is CoinGecko-capped at 365 days, so
   over a 5-year backtest the tether factor is padded for older bars. Left as-is
   (documented) since it's a low-weight confirming factor; calibration can lower
   its weight (it chose tether 0.5 on the 5-yr run).


## Round 4 — pre-release audit
Systematic audit (syntax, DOM-reference cross-check, edge-case fuzzing,
browser-load simulation, full test suite). Bugs found & fixed:

1. **Division-by-zero → `NaN` signal.** If a parameter combination zeroed every
   module weight (e.g. `paOnly` + a calibrated set with `priceAction:0`), the
   confluence divisor became 0 and `net` was `NaN`, producing fragile/unstable
   output. **Fix:** guard all weight divisors (`tfDiv`, `tfWsum`, `blendDiv`)
   with a `|| 1` / positive fallback. All-zero weights now yield a clean
   `net = 0 → NEUTRAL`.

2. **Zero-quantity TP fills logged.** A `tpSplit` of `[0,0,0]` (or partial
   zeros) produced phantom `TP1:0.000` fills in the trade log. **Fix:** an
   empty/zero split now defaults to closing the full position at TP1, and the
   TP loop only records non-zero partial fills.

3. **Negative dollars rendered as `$-1,234.50`.** The `usd()` formatter put the
   minus sign after the `$`. **Fix:** sign now precedes the symbol
   (`-$1,234.50`); verified no double-sign with the existing `+`/`-` prefixes.

4. **Timeframe dropdown labels didn't match the engine.** The HTML showed
   `Intraday (1D·4H·1H)` / `Scalp (4H·1H·15m)` but the engine actually resamples
   to `1D·8H·4H` and `4H·2H·1H`. **Fix:** labels synced to the real profile
   config.

### Verified clean (no fix needed)
- No DOM IDs referenced in JS are missing from HTML (`cprog` is created
  dynamically during calibration).
- No top-level ReferenceErrors when `app.js` loads (browser-scope simulation).
- 9 fuzz edge cases (zero capital, 200% risk, zero fees, tiny datasets, flat
  candles, `paOnly`, tiny calibrate) — no crashes, all outputs finite.
- Engine data contracts (FVG/order-block timestamps, range fields) intact.
- Test suite: **109/109 passing** (94 offline unit + 15 live integration).
