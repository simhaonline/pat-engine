# Pending Tasks — Windows Agent / MT4-MT5 / Health Page

> Handoff for continuation. Created after the `pat-engine` extraction to `/srv/pat-engine`.
> IMPORTANT: the code to fix for the items below lives in the **monorepo**
> `/srv/predictatrade/xauusd` (`windows-agent/` + `mql/`), NOT in this `pat-engine`
> project. Point the next prompt at `/srv/predictatrade/xauusd` (reference `windows-agent/`).

## Current status
- License: **ACTIVE** (dev/test key from `TESTING_LICENSE.md` in the monorepo root).
- MT5 Client EA: **CONNECTED**.
- MT4 Client EA: **OFFLINE** (unresolved).
- `127.0.0.1:9000` health page: **flapping** — intermittently shows both terminals
  OFFLINE and `LICENSE NO TERMINAL`.

## TASK 1 — MT4 OFFLINE
**Symptom:** MT4 terminal never registers; MT5 works, so the agent side is fine.
**Hypothesis:** the MT4 EA is not delivering ticks to the shared `PAT_ticks.txt`
(the agent only sends ticks when `g_connection=="CONNECTED"`, i.e. it sees the
agent heartbeat file; tick-send is gated at `mql/mt4/PredictATrade_MT4.mq4` ~line 795).
**Still needed from the user:**
- MT4 Terminal `Experts` tab output (does the EA print `Windows Agent detected`
  and tick lines, or nothing?).
- Agent log tail: `C:\PredictATrade\Client\agent.log` — look for
  `Terminal auto-registered from tick: MT4 ...` (absence = EA not sending) or any `panic`.
**Likely causes:** EA not attached / Algo Trading off / terminal logged out / stale
`.ex4` (recompile `PredictATrade_MT4.mq4`) / license gating stuck at non-ACTIVE.
**Files:** `mql/mt4/PredictATrade_MT4.mq4`, `windows-agent/internal/pipe.go`.

## TASK 2 — `:9000` health page flicker (intermittent OFFLINE + NO TERMINAL)
**Symptom:** page blinks to "both offline / LICENSE NO TERMINAL".
**Leading hypothesis:** the **agent is restarting** (panic → NSSM restarts → blank
state), OR state is wiped on each WS reconnect. Strong suspect in
`windows-agent/internal/pipe.go`: the readLoop **clears `PAT_ticks.txt` after every
read** (line ~454) while MT4 AND MT5 append to that ONE shared file, and license/
terminal state resets on reconnect.
**Fix direction:** do NOT clear the shared tick file (truncate only consumed lines);
persist last-known terminal + license state so the page stays stable; ensure JSON
parse errors recover instead of exiting.
**Files:** `windows-agent/internal/pipe.go`, `windows-agent/internal/health_endpoint.go`,
`windows-agent/internal/agent.go`.

## Already done (for reference)
- Control-plane `/licensing/validate` confirmed working: a valid key returns
  `{"valid":true,"status":"ACTIVE"}` with NO device-activation prerequisite.
- `windows-agent` agent change (commit `6666f7d`) propagates the EA-provided license
  key into device activation; needs rebuild+reinstall to take effect but may be moot
  since validation is key-based.
- `pat-engine` extracted here as its own repo; monorepo updated + pushed
  (`a95a5d8`). Docker stack unaffected (uses `./realtime`, not `./pat-engine`).

## Next session must have
1. Agent log tail (`C:\PredictATrade\Client\agent.log`).
2. MT4 `Experts` tab contents.
3. Confirm whether the agent is crashing (look for `panic`/`restart` in agent log).
