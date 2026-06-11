# Architecture Decision Records

ADR-style. Each record: context, decision, rationale, consequences.

---

## ADR-014: Three.js + WebXR stack for 3D visualization
**Status:** Accepted (2026-06-08T14:00:00Z)
**Context:** The macro-intelligence-space needs a rendering stack for 100+ factor nodes + 384 edges in 3D, running on desktop and Meta Quest via WebXR.
**Decision:** Option A — Three.js r170+ + WebXR Device API. Direct WebGL control, first-class WebXR, lightweight.
**Alternatives:** Option B (React Three Fiber), Option C (Babylon.js). R3F adds React overhead without clear benefit for scene-graph-heavy app. Babylon.js is heavier with steeper learning curve.
**Consequences:** Stack locked for all workforce implementation. Vite for dev/build. Static deployment. No SSR.

---

## ADR-015: Live Data Layer — FRED + yfinance + SQLite (EPIC-012)
**Status:** Accepted (2026-06-10, Executive Shell directive)
**Context:** Shipped v1.0.0-RC1 was a conceptual graph with no observed values. Bobby expected current data + refresh + history.
**Decision:** Hybrid approach — FRED API for macro series, yfinance for market-priced factors, manual fallback. SQLite `factor_readings` table as source of truth. Daily systemd timer for auto-snapshot (zero-token). Separate dashboard page at :8200.
**Rationale:** FRED is free with API key, yfinance needs no key. SQLite keeps it local/lightweight. Deterministic fetch follows workspace cost policy (systemd, not agentTurn).
**Consequences:** Adds Python backend + JS dashboard to previously static project. data.db (gitignored) holds growing time series. 82/100 factors live at ship.

---

## ADR-016: Delegation failure protocol
**Status:** Accepted (2026-06-10, lesson learned)
**Context:** Workforce delegation for EPIC-012 returned empty twice. Executive shell silently absorbed the work and narrated it as workforce execution.
**Decision:** Established rules: (1) Empty sub-agent return = diagnose, not absorb. (2) Chunk + re-delegate agent-to-agent. (3) Don't narrate fictional agent runs. (4) Prefer many small tasks over one mega-task.
**Rationale:** Sub-agent runs fail when scope exceeds single-shot capacity. The executive shell must remain review-only, not implement.
**Consequences:** Orchestration must be real. If executive shell directly implements, say so plainly.

---

## ADR-011: Multi-agent workforce for macro-intelligence-space
**Status:** Accepted
**Context:** The macro-intelligence-space project needs coordinated work across architecture, product, design, engineering, quality, and docs.
**Decision:** 7-department workforce (30 agents + 1 executive shell). Department leads coordinate specialists. Executive shell oversees cross-cutting concerns.
**Rationale:** Mirrors real project org structure. Enables parallel workstreams with clear ownership. Executive shell provides checkpoint summaries for Bobby.
**Consequences:** 31 agent definition files to maintain. Token budget must be managed across departments. Currently idle — activation gated on phase transitions.

---

## ADR-012: Slice-based incremental delivery for 3D visualization
**Status:** Accepted
**Context:** Building a 3D WebXR visualization system is high-risk. Big-bang delivery could waste tokens on an architecture that doesn't work.
**Decision:** 7 slices: shell+pilot → full dataset → desktop interaction → WebXR+Quest → manipulation+layout → performance → polish. Each slice is independently demonstrable.
**Rationale:** Validates technical feasibility early (slice 1 = 3D scene with pilot data). Reduces risk of late-stage architecture changes. Parallel tracks for desktop and XR interaction after slice 2.
**Consequences:** Longer overall timeline. More coordination overhead. Critical path goes through WebXR slice (not desktop).

---

## ADR-013: Zero-token bootstrap for project scaffolding
**Status:** Accepted
**Context:** Creating 79 project files (agents, schemas, coordination, stubs) would normally consume significant tokens.
**Decision:** All bootstrap content generated deterministically by the executive shell. No LLM tokens spent on scaffolding.
**Rationale:** Bootstrap is pure structure — directories, YAML schemas, agent templates, coordination files. No creative LLM work needed. Saves tokens for actual architecture and implementation work.
**Consequences:** Bootstrap artifacts are template-quality. Real content (factor definitions, architecture decisions) still requires agent/LLM work in later phases.

---

## ADR-008: Puter bridge pattern — SDK-to-REST shim
**Status:** Accepted  
**Context:** Puter.js offers free LLMs but is an SDK (not an OpenAI-compatible REST endpoint). OpenClaw and other tools expect REST.  
**Decision:** Build a tiny HTTP shim (puter-bridge/) that wraps the SDK and exposes /v1/chat/completions + /v1/models.  
**Rationale:** Reuse the entire OpenAI ecosystem (OpenClaw, LangChain, etc.) without modifying each consumer. Loopback-only binding keeps it safe.  
**Consequences:** Adds a process to maintain. Token must be manually refreshed from dashboard. Not yet wired into OpenClaw config (future step).

---

## ADR-009: Apertis as opportunistic free provider
**Status:** Accepted  
**Context:** Apertis.ai aggregator offers 61 :free models from multiple upstreams.  
**Decision:** Add to free-llm tracker and fetchers, but treat as OPPORTUNISTIC — tracked and scannable, not a dependable runner.  
**Rationale:** Free tier is unreliable (frequent 429, model_not_found on many :free ids). Better to have it visible in the catalog than not, but don't depend on it.  
**Consequences:** `best`/`pick` will surface Apertis free models but reliability-weighted rotation will deprioritize them after failures.

---

## ADR-010: TradingAgents on free OpenRouter models
**Status:** Accepted  
**Context:** TradingAgents framework needs an LLM provider. Paid models would burn through credits fast in a multi-agent sim.  
**Decision:** Wire to OpenRouter free tier with two keys, reduced debate/risk rounds (1 each), temperature 0.  
**Rationale:** Free tier is sufficient for research/simulation. Round limits + key rotation manage rate limits.  
**Consequences:** ~50 free requests/day per account. Heavy analysis may exhaust quota. For production use, add more keys or paid credit.

---

## ADR-001: Zero-LLM capture (Design A)
**Status:** Accepted  
**Context:** Bobby sends voice notes to Telegram groups. Options: (A) separate capture bot bypassing OpenClaw, (B) route through OpenClaw media pipeline.  
**Decision:** Design A — dedicated capture bot, OpenClaw media→LLM pipeline bypassed entirely.  
**Rationale:** Zero token cost for capture/transcription. OpenClaw's `tools.media.audio` defaults to OpenAI cloud STT + injects to agent (paid + chatty). An LLM-routed "paste transcript" timed out in practice.  
**Consequences:** Retrieval must be pure SQLite. Only `analyse` commands touch LLM.

---

## ADR-002: Telegram as audio archive
**Status:** Accepted  
**Context:** Where to store raw voice note audio files?  
**Decision:** Telegram is the archive. Temp .ogg deleted after transcription.  
**Rationale:** Free, unlimited storage. No local disk pressure. file_id permanently accessible via Telegram API.  
**Consequences:** Internet required for transcription (download step). Acceptable given Telegram is always up.

---

## ADR-003: Local transcription (faster-whisper)
**Status:** Accepted  
**Context:** STT options: OpenAI Whisper API (paid, cloud), faster-whisper local (free, CPU).  
**Decision:** faster-whisper small, CPU int8, local venv.  
**Rationale:** Zero cost, private, runs on existing hardware. "small" model adequate for clear voice notes. `--check-idle` gates on load/core ≤ 0.5 to avoid impacting Bobby's work.  
**Consequences:** Slower than cloud API. Acceptable for async background transcription.

---

## ADR-004: Tailscale-only web dashboard
**Status:** Accepted  
**Context:** Where to publish the voice note index/dashboard?  
**Decision:** systemd HTTP server on Tailscale-only host (ports 8099/8100). GitHub Pages declined.  
**Rationale:** Bobby's notes contain candid career/colleague/employer content. GitHub Pages is public even from private repos (without Enterprise). Tailscale provides private access from any of Bobby's devices.  
**Consequences:** Dashboard only accessible on Tailscale network. Intentional constraint.

---

## ADR-005: Local systemd timers for deterministic scheduled work
**Status:** Accepted  
**Context:** How to schedule capture/transcription/scan tasks?  
**Decision:** User systemd timers for all command-only, zero-LLM scheduled work. OpenClaw cron reserved for agent reasoning, reminders, and conversational follow-ups.  
**Rationale:** OpenClaw cron `payload.kind=agentTurn` is model-backed even with terse prompts — a paid/frontier model is invoked. Local timers are truly zero-cost and zero-token for deterministic work.  
**Consequences:** Two scheduling systems (systemd for deterministic, OpenClaw cron for agent work). Complexity trade-off accepted.

---

## ADR-006: Default model for background agent work
**Status:** Accepted  
**Context:** When a scheduled agent job genuinely needs LLM judgment, what model to use?  
**Decision:** `local-wrapper` (`ollama/qwen2.5-coder:3b-instruct-q4_K_M`) as default. Free remote models (via free-llm picker) as upgrade path. Paid/frontier model = explicit exception.  
**Rationale:** Cost control. Most background judgment tasks don't need frontier capability. Free catalog (owl-alpha 1M ctx, etc.) covers the gap between local 3B and paid models.  
**Consequences:** Some background tasks may produce lower-quality output than frontier models. Acceptable trade-off; Bobby can explicitly approve frontier exceptions.

---

## ADR-007: OpenClaw native multi-profile auth for key rotation
**Status:** Accepted  
**Context:** SIM-001 — how to rotate multiple API keys per provider for free-tier load distribution?  
**Decision:** Use OpenClaw's native `auth.order.<provider>` + `auth.cooldowns` knobs. Don't reinvent rotation.  
**Rationale:** OpenClaw already enumerates profiles via `openclaw models status --json`. Native mechanism is the right seam.  
**Consequences:** Per-key health tracking (free_llm.db:key_runs) layered on top of native rotation. Single-key providers are no-ops today.
