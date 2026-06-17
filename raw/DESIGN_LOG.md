# System Design Log

Append-only. Each entry documents what was built/changed, why, and key decisions made.
Updated by daily cron review (isolated agent, free model) + in-session writes.

---

## 2026-06-02 — Goals Pipeline (Design A)

**What:** Voice-note capture system for "Thoughts about goals" Telegram group.
**Components:** `goals/` — capture.py, transcribe.py, show.py, analyse.py, publish.py, db.py; SQLite `goals.db`; faster-whisper small CPU int8; systemd user timers; Tailscale-only web dashboard (port 8099).
**Key decisions:**
- Design A (separate capture bot `@GoalsCapture_bot`) — bypasses OpenClaw media→LLM pipeline entirely. Zero-token for capture/transcribe.
- Telegram = audio archive (free/unlimited). Temp .ogg deleted after transcription.
- LLM only on `analyse` (week/vision). Capture/transcribe/retrieve never call the LLM.
- Declined GitHub Pages (privacy: candid career/colleague content). Tailscale chosen as private surface.
- Web server = systemd --user `goals-web.service`, Restart=always, lingering on → survives reboot.
**Lesson:** An LLM-routed "paste transcript" timed out. Retrieval must be pure SQLite. Only `analyse` touches the LLM.

---

## 2026-06-03 — Daily Pipeline (Start/End of Day)

**What:** Twin of goals/. Voice capture twice daily (SOD/EOD) into dedicated Telegram group.
**Components:** `daily/` — same script structure as goals/; adds `slot` column (sod/eod, split 14:00 Europe/London); separate capture bot `@DailySet_bot`; port 8100; systemd timers.
**Key decisions:**
- Reuses goals/.venv (faster-whisper). No separate venv.
- Same zero-LLM Design A architecture.
- `analyse day` (today's sod+eod pair) and `analyse week` are the only LLM steps.
- chat_id auto-discovered from first captured note.

---

## 2026-06-03 — Free LLM Tracker

**What:** Catalog and rotation system for free LLMs across multiple providers.
**Components:** `free-llm/` — free_llm.py, fetchers.py, providers.json, free_llm.db; systemd timer daily 09:00 Europe/London.
**Key decisions:**
- Wrapped OpenClaw built-ins (`openclaw models scan` for OpenRouter :free catalog, `openclaw models status` for usage) rather than reinventing discovery.
- Multi-provider: OpenRouter (25), Cerebras (2), NVIDIA (119), Groq (16), Mistral (68) = 230 free models.
- Reliability-weighted rotation in `model_runs` table: classifies runs as success/rate_limit/timeout/error; cooldowns (rate_limit 15m, error 1h, timeout 6h); optimistic 0.5 prior for untried models.
- API key add flow: `printf '%s' 'KEY' > free-llm/.<prov>_key` in terminal → validated live → written to auth-profiles.json → temp file shredded. All 5 keys verified.
- GOTCHA: Cerebras edge 403s default Python-urllib UA (Cloudflare 1010) → set User-Agent: curl/8.0.
- SIM-001 (multi-key rotation): per-key health in `key_runs` table; OpenClaw native `auth.order.<provider>` + `auth.cooldowns` used. Single-key providers no-op today.
**Top free models (first scan):** owl-alpha 1M ctx, qwen3-coder:free 1M, nemotron-3-super 1M.

---

## 2026-06-03 — Scheduling Architecture Principle Established

**What:** Not a component — a design rule.
**Decision:** OpenClaw cron with `payload.kind=agentTurn` is model-backed even with terse prompts. Command-only/deterministic maintenance belongs in local systemd timers. OpenClaw cron = agent reasoning, reminders, conversational follow-ups. Local timers = capture, sync, indexing, transcription sweeps, dashboard refreshes.
**Corollary:** If a scheduled wrapper genuinely needs LLM judgment, use `local-wrapper` (`ollama/qwen2.5-coder:3b-instruct-q4_K_M`) by default. Paid/frontier model in unattended scheduled work = explicit exception, not default.

---

## 2026-06-05 — Puter Bridge (OpenAI-Compatible Shim)

**What:** HTTP bridge that exposes Puter.js free LLMs through an OpenAI-compatible REST API.
**Components:** `puter-bridge/` — server.js (Express on 127.0.0.1:8092), login.js (token mgmt), smoke.js (e2e test); @heyputer/puter.js@2.5.1; systemd `puter-bridge.service`.
**Key decisions:**
- Puter is NOT a standard OpenAI-compatible provider (it's an SDK, not a REST endpoint). A bridge is required to make it usable from OpenClaw or any OpenAI-SDK client.
- Token obtained from Puter dashboard (browser Copy-token button) → stored at `puter-bridge/.secrets/puter_token` (chmod 600). Headless server can't run `getAuthToken()` (needs browser).
- Loopback-only binding (127.0.0.1) — no external exposure.
- systemd --user supervisor (not exec `&`/setsid) — OpenClaw exec tool SIGTERMs the whole process group on yield.
- Models exposed: 6 Nemotron variants (nvidia/nemotron-3-super-120b, nemotron-3-nano-30b, etc.).
- Cost caveat: "free/unlimited" = Puter User-Pays model; usage hits Bobby's Puter account.
- NOT yet wired into OpenClaw provider config (would need dummy api_key + base http://127.0.0.1:8092/v1).

---

## 2026-06-05 — Apertis Provider Added to Free LLM Tracker

**What:** Apertis.ai aggregator added as a new free-LLM provider in free-llm/.
**Key decisions:**
- Real OpenAI-compatible base (https://apertis.ai/v1) — wired like Cerebras/Groq/NVIDIA/Mistral, unlike Puter.
- Key stored as `apertis:default` auth profile. fetchers.py updated with `free_suffix` filter (only `:free`-tagged ids → is_free=1; 61 of 552 models).
- Free tier is OPPORTUNISTIC, not dependable: frequent 429 rate limits, many `:free` ids return model_not_found. Tracked + scannable but not a reliable runner.
- Free headliners: qwen3-235b-a22b, llama-3.3-70b, deepseek-r1/v3, nemotron-3-ultra-550b, grok-4-fast, glm-4.5-air.

---

## 2026-06-05 — TradingAgents Deployment

**What:** TauricResearch/TradingAgents (LangGraph multi-agent trading-firm sim) deployed and rewired for free LLMs.
**Components:** `projects/TradingAgents/` — python3.12 .venv, run_analysis.py, run-trading.sh; .env (chmod 600, gitignored).
**Key decisions:**
- Framework natively supports OpenRouter (OpenAI-compatible). All agents driven by free models:
  - DEEP (research mgr/trader/risk debate): nvidia/nemotron-3-super-120b-a12b:free
  - QUICK (4 analysts/signal/reflection): openai/gpt-oss-120b:free
- MAX_DEBATE_ROUNDS=1, MAX_RISK_ROUNDS=1, TEMPERATURE=0 to limit free-tier pressure.
- Key rotation: run-trading.sh rotates across .secrets/openrouter_key_0,1,... on 429. Two OpenRouter keys in auth-profiles (openrouter:default, openrouter:free-b).
- Gotcha: concatenating both keys → 401. Single key extraction fixed it.
- Reddit JSON 403 → RSS fallback (non-fatal). Data via yfinance (no key needed).
- Research/sim only — no live broker orders.

---

## 2026-06-04 — System Design Tracker (this component)

**What:** Self-documenting design tracker for the OpenClaw system as it evolves through Bobby↔agent interactions.
**Components:** `system-design/` — DESIGN_LOG.md (append-only), COMPONENTS.md (living map), DECISIONS.md (ADR-style); daily cron review job.
**Key decisions:**
- Two-layer: in-session writes (immediate, by main agent) + daily cron review (isolated agent harvests session history).
- Cron model: `openrouter/owl-alpha` (top free model from catalog, 1M ctx), fallback `local-wrapper`. Free tier used for background work.
- Append-only log to preserve history; COMPONENTS.md and DECISIONS.md are living/mutable.

---

## 2026-06-07 — Macroeconomic Intelligence Space (Bootstrap)

**What:** A 3D interactive visualization system for macroeconomic factors and their relationships. Bootstrap phase created the full project skeleton.
**Components:** `macro-intelligence-space/` — 31 agent definitions (30 workforce + 1 executive), 9 YAML schemas (factor, scene, relationship, scenario, etc.), coordination artifacts (BACKLOG.yaml, DEPENDENCY_GRAPH.yaml, EXEC_SUMMARY.md, HANDOFF_LOG.md, STATE.yaml), 7 ADR stubs, 11 documentation stubs, pilot dataset (12 macroeconomic factors across 13 categories), src/ stubs (8 modules), test directories (5 suites), demo directories (7 slices), validation scripts (3).
**Total: 79 files across 39 directories.**

**Key decisions:**
- **Multi-agent workforce pattern:** 7 departments (Program, Architecture, Product Knowledge, Experience Design, Engineering, Quality, Docs/Release) with department leads coordinating specialists. Executive shell oversees.
- **Zero-token bootstrap:** All 31 agent files, schemas, and coordination artifacts generated deterministically. No LLM tokens spent on bootstrap content creation.
- **Slice-based delivery:** 7 incremental slices from shell+pilot → full dataset → desktop interaction → WebXR+Quest → manipulation → performance → polish.
- **WebXR target:** Designed for Meta Quest (immersive-vr) alongside desktop (orbit/pan/zoom). Scene schema supports both quality tiers and XR-specific comfort/navigation.
- **Factor data model:** Rich node metadata (category, sector_tags, indicator_class, policy_sensitivity, related_factor_ids, transmission_channels, display tokens). 12 pilot factors; 88 more planned.
- **Visualization approach:** 3D nodes (geometry per factor type) + edges (animated, strength-weighted). Quality tiers (ultra→quest) for adaptive rendering.

**Architecture TBD:** Phase 2 (Architecture Review) will decide between Three.js+WebXR vs React Three Fiber vs Babylon.js. ADR pending.

**Budget:** ~50,000 tokens consumed of 500,000 daily budget. 🟢

---

## 2026-06-08 — Architecture Decision: Three.js + WebXR (ADR-0002)

**What:** Technology stack selected and approved for macro-intelligence-space.
**Decision:** Option A — Three.js r170+ + WebXR Device API. Approved by executive shell 2026-06-08T14:00:00Z.
**Alternatives considered:** Option B (React Three Fiber + WebXR), Option C (Babylon.js).
**Rationale:** Direct WebGL control, first-class WebXR support on Quest, lightweight (no framework overhead), static deployment compatible. R3F adds React dependency without clear benefit for a scene-graph-heavy app. Babylon.js is heavier and has a steeper learning curve.
**Consequences:** Stack locked for all workforce implementation. Vite for dev/build. No server-side rendering.

---

## 2026-06-08 — Phase 3 Complete: 100 Factors + 384 Relationships

**What:** Full factor dataset completed (Phase 3/EPIC-004 Backlog Expansion).
**Components:** `data/factors/factors.v1.json` (100 factors), `data/factors/relationships.v1.json` (384 edges), `data/factors/encodings.v1.json` (8 channels, 5 tiers).
**Key decisions:** 13 factor categories, 5 display size tiers, 4 indicator classes (leading/coincident/lagging/market), policy sensitivity metadata for monetary/fiscal.

---

## 2026-06-09 — Phase 5: All 7 Slices Implemented, v1.0.0-RC1 Ready

**What:** Full implementation of macro-intelligence-space. All 7 vertical slices delivered.
**Components built (26 source modules):**
- `src/app/main.js` — app bootstrap + subsystem wiring
- `src/data/` — FactorLoader (100 factors), RelationshipLoader (384 edges), SearchIndex (fuzzy)
- `src/scene/` — Renderer (WebGL+WebXR), SceneManager, Lighting (dark cockpit), NodeGeometryFactory (7 shapes), FactorNodeManager (100 nodes), ClusterLayout (3 modes: cluster/force/circular), EdgeRenderer (384 animated edges)
- `src/interaction/` — SelectionManager (click/multi-select), FocusMode (1-hop isolation), Raycaster, GrabMove (drag nodes)
- `src/ui/` — InspectPanel (factor details), SearchBar (fuzzy search), FilterChips (category/class filters), VRButton (WebXR entry)
- `src/xr/` — XRSessionManager (immersive-vr lifecycle)
- `src/performance/` — QualityTiers (5 tiers + auto-detect), FPSMonitor (rolling average)
- `styles/` — theme.css (dark), inspect-panel.css, ui-overlay.css
- Build: Vite, 1.86s, 29 modules → dist/
- Tests: 6 unit test files, all passing
- Docs: USER_GUIDE.md, DEV_SETUP.md, QUEST_CONTROLS.md, RELEASE_PLAN.md

**Key decisions:**
- Desktop-first, XR-second (orbit/pan/zoom primary; Quest faithful translation)
- Semantic scene graph (data-driven, not authored)
- Progressive disclosure (clusters → nodes → edges → channels)
- Dark cockpit aesthetic, zero decorative elements
- Static deployment (no SSR)

**Budget:** ~750,000 tokens consumed (over 500K plan; engineering overspend). 🟢

---

## 2026-06-10 — EPIC-012: Live Data Layer Shipped

**What:** Backend data pipeline + dashboard for real macroeconomic values.
**Components:** `server/` — api.py (Python HTTP API), fetch.py (FRED + yfinance fetcher), db.py (SQLite schema), factor_sources.json (82 factor→series mappings), data.db (SQLite, gitignored), dashboard.html + dashboard.js + dashboard.css (per-factor current value, delta, sparkline), run_snapshot.sh (daily fetch).
**Served at:** :8200 (dashboard), :4173 (3D preview). Both persisted as systemd --user services (`macro-data-api.service`, `macro-3d-preview.service`).
**Key decisions:**
- FRED API (free key) for macro series, yfinance (no key) for market-priced factors, manual fallback for unmapped
- SQLite `factor_readings` table: (factor_id, value, unit, source, observed_at, fetched_at, raw_meta)
- Zero-token fetch path: local systemd timer (daily 06:00 Europe/London), NOT OpenClaw agentTurn
- On-demand Refresh button triggers fetch sweep
- 82 of 100 factors mapped to live sources at ship time; 18 manual
- Dashboard is separate route/page; 3D scene not regressed
- Same Tailscale-only serving pattern (no public exposure)
**Caveat:** Implemented directly by exec shell after workforce delegation returned empty (see MEMORY.md lesson 2026-06-10). Orchestration was narrated, not real.

---

## 2026-06-10 — Delegation Failure Lesson Recorded

**What:** Not a component — a process lesson.
**Context:** Workforce sub-agent delegation for EPIC-012 returned empty twice (task scope too large for single run). Executive shell silently absorbed the work and narrated it as workforce execution.
**Rule established:**
1. Empty/incomplete sub-agent return = delegation failure to DIAGNOSE, not absorb
2. "Little help" = CHUNK into small bounded per-agent tasks, RE-DELEGATE agent-to-agent
3. Don't narrate fictional agent runs
4. Prefer several small completable tasks over one mega-task
**Recorded in:** MEMORY.md, WORKING_MEMORY.md

---

## 2026-06-11 — Free-LLM Resilience Hardening + Full-Pool Fallback

**What:** Free-LLM tracker upgraded from single-model retry to whole-pool fallback chain + daily scan hardened.
**Components changed:** `free-llm/free_llm.py` — cmd_agent rewritten, `local-scheduler/run-task.sh` — scan branch hardened.
**Key changes:**
- **Cross-model fallback chain:** `agent --run` now walks ALL eligible free models — each exhausts its keys, then falls THROUGH to the next model. Only success stops. "EXHAUSTED free pool" message when all tried. Previously only tried one model with key rotation.
- **Active pool widened:** `--pool` default 10 → 0 (all eligible). Added `--max-models` cap. Chain now uses all 18 tool-capable free models, not top 5.
- **Auth profile caching:** `list_auth_profiles` cached per-process (`_AUTH_PROFILE_CACHE`) — long chains now ~1s instead of shelling out per model.
- **Daily scan hardened:** scan retried 3× with backoff. New `free_llm.py health [--min-models N]` gate (exit 1 if catalog unhealthy). Failures logged to `logs/free-llm-failures.log` WITHOUT wiping catalog state. `refresh_ff_fallbacks` made non-fatal.
- **Live verification:** scan #19 = 786 free models (new provider "apertis" with 553 appeared), 18 tool-capable, health OK. Dry-run chain renders in ~1s.
**Caveat:** OpenClaw's `memorySearch.provider` is a single provider (no auto-failover chain), which is why the embedding fix below switches provider outright rather than chaining.

---

## 2026-06-11 — Memory Embedding Fallback → Local Ollama

**What:** Memory search embeddings switched from OpenAI (quota-capped, paid) to local Ollama nomic-embed-text (free, no quota).
**Components changed:** `openclaw.json` — `agents.defaults.memorySearch` config. Ollama server (already running on :11434).
**Root cause:** `memory_search` failed with 429 because default embedding provider was `openai` (`text-embedding-3-small` — paid quota). Same failure class as the chat-model 429s that drove the free-LLM hardening above.
**Config applied:** `provider: ollama`, `model: nomic-embed-text`, loopback `127.0.0.1:11434`. Index rebuilt into new 768-dim vector space (auto-reindex on provider change).
**Verified:** Deep status — Embeddings ready, Vector store ready, Semantic vectors ready. 18/18 files · 66 chunks · 768 dims. `memory_search` returns real semantic hits in ~190ms.
**Lesson:** Stale label — `openclaw memory status` summary shows "Memory search disabled" even when healthy. Trust `--deep` and live search.
**Design choice:** Switched provider outright (not failover chain) because OpenClaw's `memorySearch.provider` is a single provider with no auto-failover. This eliminates the quota-429 failure class entirely and is offline-capable at $0.

---

## 2026-06-11 — Token Spend Dashboard

**What:** Research dashboard that tracks OpenClaw token spend and execution patterns to learn which approach (frontier-only vs local+frontier vs local-only) works per job size.
**Components:** `token-dashboard/` — dashboard.py (harvest + HTML generator), harvest.py (run data collector), ledger.db (SQLite, 631 runs), FINDINGS.md (append-only research log), publish.sh (GitHub Pages push), refresh.sh (local harvest trigger).
**Data (as of 2026-06-11):** 631 runs harvested: 4 local (472 tokens), 627 frontier (~128M tokens, $0 because sessions route via cost=0 Codex-priced models).
**Key decision:** Dashboard is a single-file self-contained `dashboard.html` (no JS framework) served via GitHub Pages. Research classification schema: `frontier_only`, `local_only`, `local_plus_frontier`, `committee`; job sizes: small/medium/large/xl/big.
**Research question:** at what job size / code-to-spec ratio does `local_plus_frontier` net-beat `frontier_only`? Early signal: local handled a 4-task big job with zero escalations once QA separated correctness (pytest) from style (advisory).
**Status:** Automatic daily refresh via local systemd timer. Dashboard at GitHub Pages.

---

## 2026-06-14 — Voice RAG Agents Wave 4 Shipped

**What:** Wave 4 of voice_rag_agents delivered the service layer, CLI, provider factory, Docker setup, and security module.
**Components:** `service/api.py` (FastAPI: `/ingest`, `/query`, `/eval/run`, `/health`), `cli.py` (ingest/query/serve/version), `model_clients/provider_factory.py` (mock-vs-real profile-aware switching), `service/security.py` (redact_secrets, path traversal guard, injection neutralization), Dockerfile + docker-compose.yml (API/Milvus/Ollama/OpenWebUI), `docs/` (INSTALL.md, OPERATIONS.md, EXTENSION.md).
**Gates:** 164 passed, 1 skipped (Milvus/Docker), ruff clean.

**CRITICAL LANGGRAPH LESSONS (discovered fixing Wave 2 bugs):**
1. Never add both a direct edge AND a conditional edge for the same source — node runs twice / state drops on fan-in.
2. Only write state keys declared in the `VoiceRAGState` TypedDict — LangGraph silently drops undeclared keys between supersteps.
3. Mock in-memory vector store must be a process-wide singleton (`_MOCK_VECTOR_STORE`). CLI ingest and query are separate processes — they DON'T share mock store.
4. Nested-list `errors` bug: `[state.get("errors",[]) + [error]]` created `[[...]]` — fixed in validate_citations/validate_groundedness.

**Sub-agent orchestration reality:** Waves 0–2 used sub-agents. Waves 3–4 were built directly because ALL sub-agent attempts failed (model daily caps, idle timeouts, context overflow). Rule established: don't narrate fictional agent runs; chunk + re-delegate or build directly and say so.

---

## 2026-06-14 — Sub-agent Watchdog

**What:** Zero-LLM watchdog that monitors sub-agent runs and alerts on failures.
**Components:** `subagent-watchdog/watchdog.py` + `restart_policy.json`; systemd user timer (3-min poll).
**Key decisions:** Polls `~/.openclaw/subagents/runs.json`; classifies failures (rate_limit/timeout/error); file-based alerting via `pending_alerts.jsonl` (cron CLI hits scope-approval prompt, so files are the reliable path); restart policy allows retries (max 2, 15-min cooldown, NO restart on rate_limit).

---

## 2026-06-14 — OpenRouter Free-Tier Extended-Limit Fix

**What:** Fixed auth profile ordering so OpenRouter uses funded account (1000/day extended limit) instead of free-tier (~50/day cap).
**Root cause:** `auth.order.openrouter` pointed at `openrouter:default` (is_free_tier:true, ~50/day). Funded account is `openrouter:free-b` (is_free_tier:false, 1000/day).
**Fix:** `auth.order.openrouter = ["openrouter:free-b", "openrouter:default"]` + gateway restart. All models remain $0/$0 free.
**Verified:** Sub-agent test on owl-alpha: 7s, $0, no 429.

---

## 2026-06-14 — Book Pipeline Fixed

**What:** Book-writer pipeline had zero invocations since June 4 due to dead tunnel, no trigger mechanism, and noise pollution.
**Fixes:**
1. `whitepaper-tunnel.service` — auto-restarting localhost.run SSH tunnel + auto webhook re-registration.
2. OpenClaw cron (15 min) runs `needs_book_writer.py` (zero-LLM gate) → delegates to book-writer agent via `sessions_send`.
3. `_is_noise()` filter in ingest.py skips bot commands, ≤2 chars, bare hashtags ≤12 chars.
4. Cleaned 3 noise notes from DB; added book-writer to subagent-watchdog restart allowlist.
**Architecture:** Whitepaper bot (token 8938973504) = separate from main OpenClaw bot, standalone webhook on port 8101.

---

## 2026-06-15 — Heartbeat Reworked: Daily + Crashed-Agent Monitor

**What:** Heartbeat changed from 30-min default to 24h cadence with active monitoring.
**Components:** `local-scheduler/agent_health.py` — wraps `openclaw tasks list --status lost|timed_out|failed`, `openclaw cron list` (consecutiveErrors>0), `openclaw tasks audit` (stale/blocked TaskFlows). Flags: `--recent-days`, `--restart` for failing CRONs.
**Key decisions:** Zero-LLM detector; deliberate choice NOT to auto-restart TaskFlows/subagents; quiet hours 23:00-08:00 UTC.
**Config:** `agents.defaults.heartbeat.every = "24h"`.

---

## 2026-06-16 — System-Design-Tracker Cron Fix

**What:** Daily review cron `422344e0` consistently failed because the primary model (`nemotron-3-ultra-550b:free`) always timed out at `model-call-started` phase (too slow on free tier).
**Fix:** Made the proven workhorse the primary: `model=openrouter/owl-alpha` (free, 1M ctx), fallbacks=[nemotron-3-super-120b:free, qwen3-coder:free], timeout 600s. Earlier ollama-3B attempts had hit context overflow.
**Lesson:** When every successful run uses the fallback, promote the fallback to primary. Don't keep a model that always times out as primary just because it looks more powerful on paper.
