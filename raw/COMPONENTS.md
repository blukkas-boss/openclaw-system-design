# Components Map

Living document. Updated by daily cron review + in-session writes.
Last updated: 2026-06-11

---

## goals/ — Voice Goals Pipeline
- **Purpose:** Capture Bobby's voice notes from Telegram "Thoughts about goals" group; transcribe locally; analyse on demand.
- **Capture bot:** @GoalsCapture_bot (id 8879160447), admin in group
- **DB:** goals/goals.db (SQLite) — tables: notes, meta
- **Transcription:** faster-whisper small, CPU int8, goals/.venv
- **Web:** http://ubuntu.tailefc181.ts.net:8099 (systemd goals-web.service, port 8099)
- **Timers:** openclaw-local-goals-capture.timer (10min), openclaw-local-goals-transcribe-idle.timer (30min), openclaw-local-goals-transcribe-nightly.timer (03:00 UTC)
- **LLM touch:** `analyse week` / `analyse vision` only

## daily/ — Start & End of Day Pipeline
- **Purpose:** Capture twice-daily SOD/EOD voice notes from dedicated Telegram group.
- **Capture bot:** @DailySet_bot (id 8678513817), admin in group
- **DB:** daily/daily.db — adds `slot` column (sod/eod, split 14:00 Europe/London)
- **Venv:** reuses goals/.venv
- **Web:** http://ubuntu.tailefc181.ts.net:8100 (systemd daily-web.service, port 8100)
- **Timers:** openclaw-local-daily-capture.timer (10min), openclaw-local-daily-transcribe-idle.timer (30min), openclaw-local-daily-transcribe-nightly.timer (03:30 UTC)
- **LLM touch:** `analyse day` / `analyse week` only

## free-llm/ — Free LLM Tracker & Rotation
- **Purpose:** Catalog free LLMs across providers; reliability-weighted model rotation; multi-key auth rotation.
- **DB:** free-llm/free_llm.db — tables: models, model_runs, key_runs
- **Providers:** OpenRouter (25), Cerebras (2), NVIDIA (119), Groq (16), Mistral (68) = 230 free models
- **Key scripts:** free_llm.py (CLI), fetchers.py (per-provider catalogs), providers.json (registry)
- **Timer:** openclaw-local-free-llm-scan.timer (daily 09:00 Europe/London)
- **Top models now:** owl-alpha (1M ctx), qwen3-coder:free (1M), nemotron-3-super (1M)
- **SIM-001:** Multi-key provider rotation (per-key health tracking, cooldowns, LRU selection)

## local-scheduler/
- **Purpose:** Runner for deterministic local tasks. run-task.sh dispatches to named task scripts.
- **Tasks wired:** free-llm-scan, goals-capture, goals-transcribe, daily-capture, daily-transcribe

## puter-bridge/ — Puter.js OpenAI-Compatible Shim
- **Purpose:** Expose Puter.js free LLMs through an OpenAI-compatible REST API for use from OpenClaw or any OpenAI-SDK client.
- **Endpoint:** http://127.0.0.1:8092/v1 (loopback-only)
- **Server:** puter-bridge/server.js (Express), @heyputer/puter.js@2.5.1
- **Auth:** Puter account token at puter-bridge/.secrets/puter_token (chmod 600); obtained from Puter dashboard
- **Models:** 6 Nemotron variants (nvidia/nemotron-3-super-120b, nemotron-3-nano-30b, etc.)
- **Supervisor:** systemd --user `puter-bridge.service` (enabled, Restart=always, linger=yes)
- **Status:** Fully working, end-to-end tested. NOT yet wired into OpenClaw provider config.
- **Cost:** "Free/unlimited" = Puter User-Pays model; usage hits Bobby's Puter account (watch usd_cents).

## projects/TradingAgents/ — Multi-Agent Trading Simulation
- **Purpose:** LangGraph multi-agent trading-firm sim (analysts → bull/bear researchers → research manager → trader → risk team → portfolio manager).
- **Source:** TauricResearch/TradingAgents, python3.12 .venv
- **Models:** All free via OpenRouter — DEEP: nemotron-3-super-120b:free, QUICK: gpt-oss-120b:free
- **Key rotation:** run-trading.sh rotates OpenRouter keys on 429 (two keys: openrouter:default, openrouter:free-b)
- **Limits:** MAX_DEBATE_ROUNDS=1, MAX_RISK_ROUNDS=1, TEMPERATURE=0
- **Data:** yfinance (no key), Reddit RSS fallback. Research/sim only — no live orders.

## macro-intelligence-space/ — 3D Macroeconomic Visualization
- **Purpose:** Interactive 3D visualization of macroeconomic factors and their relationships. Desktop + WebXR (Meta Quest).
- **Status:** Bootstrap complete (Phase 0). 79 files across 39 directories.
- **Agents:** 31 agent definitions (30 workforce + 1 executive) across 7 departments. Currently idle, awaiting Phase 1 activation.
- **Schemas:** 9 YAML schemas — factor, scene, relationship, scenario, agent, sprint, handoff, report, state.
- **Coordination:** BACKLOG.yaml (11 epics), DEPENDENCY_GRAPH.yaml, EXEC_SUMMARY.md, HANDOFF_LOG.md, STATE.yaml.
- **Data:** Pilot dataset of 12 macroeconomic factors (GDP, inflation, unemployment, interest rates, etc.) with rich metadata. 88 more planned.
- **Delivery:** 7 slices — shell+pilot → full dataset → desktop interaction → WebXR+Quest → manipulation → performance → polish.
- **Architecture TBD:** Phase 2 will decide Three.js+WebXR vs React Three Fiber vs Babylon.js.
- **Project root:** `/home/bobby/.openclaw/workspace/macro-intelligence-space/`

## token-dashboard/ — Token Spend Tracker
- **Purpose:** Track token spend + execution patterns to learn which approach (frontier-only vs local+frontier vs local-only) works best per job size.
- **Components:** dashboard.py (harvest + HTML generator), harvest.py (run data collector), ledger.db (SQLite), FINDINGS.md (append-only research log), publish.sh (GitHub Pages push), refresh.sh (local harvest trigger).
- **Dashboard:** Single-file `dashboard.html` (self-contained, no JS framework) served via GitHub Pages.
- **Data:** 631 runs harvested as of 2026-06-11: 4 local (472 tokens), 627 frontier (~128M tokens).
- **Key finding:** Local model handled a 4-task big job with zero escalations once QA separated correctness (pytest) from style (advisory). Research question: at what job size does `local_plus_frontier` net-beat `frontier_only`?
- **Location:** `/home/bobby/.openclaw/workspace/token-dashboard/`

## macro-intelligence-space/ — 3D Macroeconomic Visualization (v1.0.0-RC1 + Live Data)
- **Purpose:** Interactive 3D visualization of macroeconomic factors and their relationships. Desktop + WebXR (Meta Quest). Now includes live data dashboard.
- **Status:** v1.0.0-RC1 complete. EPIC-012 (Live Data Layer) shipped.
- **3D App:** Three.js r170+ + WebXR Device API. 26 source modules, Vite build → dist/. 100 factor nodes, 384 edges, 7 geometry types, 3 layout modes, 5 quality tiers. Dark cockpit aesthetic.
- **Live Data Backend:** `server/` — Python HTTP API (api.py), FRED+yfinance fetcher (fetch.py), SQLite (data.db). 82/100 factors mapped to live sources.
- **Dashboard:** HTML/JS dashboard at :8200 (per-factor current value, delta vs previous, sparkline history). Refresh button for on-demand fetch.
- **3D Preview:** Served at :4173.
- **Systemd services:** `macro-data-api.service` (:8200), `macro-3d-preview.service` (:4173). Both enabled, Restart=always, linger=yes.
- **Data snapshot:** Daily systemd timer (06:00 Europe/London), zero-token deterministic fetch.
- **Tests:** 6 unit test files, all passing.
- **Agents:** 30 workforce agents across 7 departments. All slices delivered; agents now idle.
- **Architecture:** ADR-0002 — Three.js + WebXR (Option A) approved 2026-06-08.
- **Project root:** `/home/bobby/.openclaw/workspace/macro-intelligence-space/`

## system-design/ — Design Tracker (this component)
- **Purpose:** Self-documenting log of OpenClaw system design decisions as they emerge from Bobby↔agent interactions.
- **Files:** DESIGN_LOG.md (append-only), COMPONENTS.md (this file), DECISIONS.md (ADRs)
- **Review cron:** Daily isolated agentTurn, model=owl-alpha (free), fallback=local-wrapper

---

## Infrastructure
- **Host:** ubuntu (Linux 6.8.0-124-generic x64)
- **Tailscale:** ubuntu.tailefc181.ts.net (100.120.64.13)
- **Workspace:** /home/bobby/.openclaw/workspace
- **Node:** v24.15.0
- **Shell:** bash
