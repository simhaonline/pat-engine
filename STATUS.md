# pat-engine — Status (as of 2026-08-28)

Extracted as a standalone project from the `predictatrade` monorepo to `/srv/pat-engine`
(its own git repo). The monorepo no longer contains or references `pat-engine`.

Decision: build a clean standalone engine, reference only, no existing service/DB,
current repo untouched. Delivered and tested:

- **Engine (zero-dep Go):** 4 strategy products (`ULTRA_SCALPING`,
  `STANDARD_SCALPING`, `STANDARD_SWING`, `TREND_SWING`), single-source SL/TP/RR config,
  shared confluence scoring + trade geometry, broker-policy gating, hard risk gates
  (R:R floor + broker stop/freeze).
- **Broker scalping policy is first-class:** `AllowsScalping=false` excludes both
  scalpers (`BROKER_SCALPING_NOT_ALLOWED`); only swing/trend remain eligible. Verified
  by tests + backtest harness.
- **Backtest harness** runs the *exact same* strategy code/config as live; synthetic
  demo yields PF (ULTRA ~3.1, SWING ~7.7 on trend-friendly synthetic data) and a real
  `BARS_CSV` loader is wired for KAGGLE/MT5 data.
- **Live gateway (stdlib HTTP):** ingests bars (`POST /bar`), runs the pipeline, writes
  `SIGNAL|<json>` to `PAT_signals.txt` in the exact format the existing MQL EA parses.
- **End-to-end proven:** reference agent streams bars → gateway → signal file →
  EA-file simulator "executes". `go test ./...` green (executability, R:R floor,
  broker exclusion, high-spread block).
- **MQL reference EAs** (`mql/`) read `PAT_signals.txt` unchanged.

Deferred (per plan, backend+MQL first): frontend/Command Center, control-plane
licensing WS, rebuilt Windows Agent binary (reuse `windows-agent`, point at `/bar`).

Next: plug real 2025 KAGGLE/MT5 bars into `cmd/backtest` to lock honest v1 stats,
then wire the live agent + EA on a no-scalping-broker-eligible package (swing/trend).

## Open debugging tasks (live MT4/MT5 setup — lives in the monorepo, not here)
See `PENDING_TASKS.md`: MT4 OFFLINE + `:9000` health-page flicker. The agent/EA code
to fix is in `/srv/predictatrade/xauusd` (`windows-agent/` + `mql/`).
