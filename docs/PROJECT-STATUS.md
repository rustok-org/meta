# Project Status — Rustok Org

> Updated: 2026-06-14  
> Read this at session start to understand where we are.  
> Phase numbering follows `core/docs/CORE-MCP-ROADMAP.md` (source of truth for the MVP roadmap).

---

## Current Phase: Phases 0–6.1 ✅ + production cutover done → Phase 7 (Self-Custody Distribution) in progress

**Roadmap goal:** Functional Core + Gateway + MCP Server = minimum viable product.

| Phase | Scope | Status |
|-------|-------|--------|
| 0 — Core Hardening | PrivateKey newtype, gas oracle, docs | ✅ #24, #26, #27 |
| 1 — Core as gRPC Service | Tonic server, Dockerfile, compose | ✅ #28/#29, #30 |
| 2 — Gateway (Axum) | scaffold, Core bridge, rate limit | ✅ #31, #32, #33 |
| 3 — MCP Server (Python) | scaffold, protocol, capabilities, Gateway client | ✅ mcp #17, #18, #19, #20; read path (PR-3.5) ✅ mcp #22 + core #44 |
| 4 — Event Bus + Audit | Redis Streams publisher, audit consumer | ✅ #34, #36 |
| 4.5 — Core Real Integration | gRPC stub → real wallet/router/sign/provider | ✅ #38, #40, #41, #42, #43 (plan #39) |
| 5 — Production Hardening | docker-compose-full, observability, TLS | ✅ full compose meta #11; MCP inbound auth mcp #24; Caddy TLS meta #15; **observability COMPLETE** (logs+traces+metrics, PR-5.2a/b/c/d); postgres (5.3) still optional/pending |
| 6.1 — DeFi positions | Aave v3 + ERC-4626 `get_positions` | ✅ core #52, mcp #27 (clean-room port of v1 agent-dapps) |
| 7 — Self-Custody Distribution | onboarding, public image, all-in-one MCP image, ClawHub republish | 🔄 onboarding ✅ core #53 (PR-7.3); GHCR publish workflow ✅ core #54 (PR-7.2, **no `v*` tag pushed yet**); de-v1-ify ✅ mcp #28; all-in-one stdio image ✅ mcp #29 (PR-7.1b); **ClawHub web republish pending** |
| Production cutover | new stack on `api.rustokwallet.com` | ✅ done 2026-06-14; audit-consumer fixes #55/#56 built & deployed; monolith stopped (retained for rollback) |

**Core workspace (13 crates):** `types`, `crypto`, `keyring`, `provider`, `router`, `txguard`, `sign`, `audit`, `events`, `wallet`, `gateway`, `grpc`, `positions` (PR-6.1) (~190 `#[test]` functions as of PR #43; positions added more).

**End-to-end chain works with real data:** MCP → Gateway → Core returns real address, balances, `tx_hash`, EIP-191 signatures (Phase 4.5 gate passed). All 5 original MCP tools are real since 2026-06-10 (read path: Gateway `GET /api/v1/wallet/context` + `/balance`, core #44; MCP wiring, mcp #22 — verified end-to-end via stdio). Full stack runs via `meta/docker-compose.yml` (meta #11).

**DeFi positions shipped (PR-6.1, 2026-06-14):** a 6th MCP tool `get_positions` (Aave v3 + ERC-4626) — new core `crates/positions` + `GetPositions` gRPC + gateway `GET /api/v1/wallet/positions` (core #52), mcp tool gated by READ_WALLET (mcp #27). Clean-room port of v1 `agent-dapps`; best-effort, no price oracle. Verified live full-chain (MCP → Gateway → Core) against a real mainnet Aave v3 position. **The org MCP is now at parity with the deployed ClawHub wallet (`temrjan/rustok-wallet`) and ready to replace it.**

---

## In Progress 🔄

| Task | Repo | Blocker |
|------|------|---------|
| Phase 7 finish: tag `v*` → publish public Core image; then republish ClawHub skill | core / mcp | needs version tag + Captain ClawHub web publish |
| Fix gateway↔core startup race (eager connect, no retry) | core (gateway) / meta | see Blockers |

---

## Next Immediate Steps

### Primary: Finish Phase 7 (self-custody distribution)
1. **Tag a `v*` release on `core`** → triggers #54 to publish
   `ghcr.io/<owner>/rustok-core`; the mcp all-in-one image (#29) depends on it.
2. **Captain republishes the skill on ClawHub** (web UI) once the image is live.
3. **Fix the gateway↔core startup race** (eager connect, no retry) — reconnecting
   tonic channel or `depends_on: service_healthy` in compose.
4. **Reproducibility:** commit deploy-v2 compose + `Caddyfile-v2` into `meta`.
5. **Remove the old monolith** after 24–48h prod stability.

### Done — Phase 5 Production Hardening (PR-5.3 Postgres is the only optional remainder)
1. ~~**PR-5.1 remainder** — reverse proxy + SSL~~ ✅ **done** — Caddy TLS overlay
   (`docker-compose.prod.yml`), inbound auth required in prod (mcp #24). Real
   Let's Encrypt issuance + host-level rate limiting are deploy-time steps
   (README deploy checklist).
2. **PR-5.2** `feat/observability` (decomposed):
   - **5.2a** ✅ — observability backend in `meta` (Alloy + Tempo +
     Prometheus + Loki + Grafana overlay) + host-level edge rate-limit docs (#9).
   - **5.2b** ✅ — `core` instrumentation: JSON logs + EnvFilter,
     OTLP/HTTP trace export to `alloy:4318`, `#[instrument]` spans on gRPC
     handlers, W3C traceparent propagation gateway→core, `trace_id` in logs.
     Adds the `telemetry` network in `meta`.
   - **5.2c** ✅ — `mcp` instrumentation (Python OTel auto-instrumentation:
     FastAPI server span + httpx traceparent) + Gateway inbound HTTP traceparent
     extract (core) + mcp on the `telemetry` network (meta). Full
     **MCP→Gateway→Core** trace in Tempo, `trace_id` in Loki.
   - **5.2d** ✅ — metrics via **OTLP push** to Alloy (→ Prometheus
     remote_write; no `/metrics` scrape): gateway HTTP-duration middleware, core
     gRPC per-handler duration, mcp auto HTTP metrics. All three services.
   - **Phase 5 observability COMPLETE** — logs + traces + metrics across mcp,
     gateway, core. e2e trace-in-Grafana gate MET (5.2c).
3. **PR-5.3** `feat/postgres-migration` — **optional, not started** (the only remaining Phase 5 item)

### Backlog (from review round 2026-06-10, below single-PR threshold)
- Gateway positive-path tests: needs an in-process tonic test server (`CoreClient` is concrete, not a trait) — verify JSON serialization against proto changes
- `wallet_context` TTL cache on Gateway — **rejected for now** (tools are called on demand, stale balance risk); revisit with Phase 5 metrics
- chore (mcp): make `mypy --strict` clean on the pre-existing `tests/` (Gate-2 reviewer note on PR-6.1 PR-B; `mypy src` is already clean — this is test-only debt, unrelated to feature code)

### Deferred: security debt (from Phase 4.5)
1. Runtime/RPC unlock (today: startup-unlock only via `UnlockMethod`)
2. Keystore file perms `0600`; receipt wait in `router::execute`; parallel balance fetch

> Hard spend-limit/budget/policy is **deliberately dropped** (risk is the user's
> to accept; opt-in limits may come later). `txguard` is kept. Do **not** re-add a
> forced BudgetTracker.

### Deferred: LLM Brain (separate product line)
1. Resolve stack decision: Rust (Rig) vs Python (FastAPI + LangChain) — see Deferred Decisions
2. Define Intent JSON schema; scaffold in `rustok-org/llm`

---

## Blockers 🚧

| Blocker | Impact | Resolution |
|---------|--------|------------|
| Gateway↔core startup race (eager connect, no retry) | Recreating core+gateway together → gateway serves `core_unavailable` until restarted | Workaround: restart `gateway` after core is listening. Fix: reconnecting tonic channel / compose `depends_on: service_healthy`. |
| Public Core image not published to GHCR | mcp self-custody all-in-one image (#29) can't pull Core binaries | Tag a `v*` release on `core` → #54 publishes the image. |
| GitHub Free — no branch protection for private repos (`core`) | Can accidentally push to `main` | **Mitigated** — pre-push hook + process discipline. |
| `uniffi` FFI bridge does not exist (no crate) | Mobile cannot call core | Low priority until Mobile phase. Needs `uniffi-bindgen-react-native` scaffold. |
| `simulateAssetChanges` not implemented in provider | Cannot preview swap/stake asset changes | Documented in `meta/docs/ALCHEMY-INTEGRATION.md`. Deferred to provider v2. |

---

## Deferred Decisions

| Decision | Options | Status |
|----------|---------|--------|
| `llm` stack | Rust (Rig) vs Python (FastAPI + LangChain) | **Under review.** `meta/docs/LLM-ARCHITECTURE.md` recommends Rust (Rig) with gRPC isolation fallback; reconcile at LLM kickoff. |
| `llm` repo visibility | AGENTS.md says private; repo is **public** on GitHub | Verify intent — flip visibility or fix docs. |
| ClawHub/Smithery legacy skill (`temrjan/rustok-wallet`, ~323 downloads) | Keep / republish on Python MCP | **Parity reached (PR-6.1).** The new MCP now matches ClawHub (incl. `get_positions`). Still parked — do NOT delete until the new MCP is republished/deployed. |
| React Native version | 0.76 (current) vs latest | **Resolved:** Stay on 0.76 until Mobile phase. |
| Hardware signers | Ledger, Keystone, air-gapped | After core v1.0 |
| Swap integration | 1inch, 0x, CoW | After txguard v2 + `simulateAssetChanges` |

---

## Completed ✅ (recent — full trail in git history)

| Date | Task |
|------|------|
| 2026-05-27 | Org + 5 repos created, CI/CD hardened, `crypto` + `keyring` crates |
| 2026-05-28 | Security hardening PR #6/#7, audit Steps 1–5, `provider` + `txguard` |
| 2026-05-29 | `router` + `sign`, EIP-1559, edition 2024, MSRV 1.91 |
| 2026-05-30 | `audit` crate (#21, #22), PrivateKey newtype (#24), gas auto-fetch (#26), gRPC scaffold (#28), core-docker (#30) |
| 2026-05-31 | **mcp:** Python FastAPI rewrite — scaffold (#17), protocol + SSE + stdio + tool registry (#18) |
| 2026-06-01 | **mcp:** capability-based permissions (#19), Gateway HTTP client PR-3.4 (#20), CI bump (#21) |
| 2026-06-01–02 | **core:** Gateway (Axum) Phase 2 (#31–#33), Event Bus + Audit Phase 4 (#34, #36) |
| 2026-06-02 | **core:** `crates/wallet` WalletCore (#38), Phase 4.5 plan v2 (#39) |
| 2026-06-03 | **core:** Phase 4.5 complete — gRPC wired to real WalletCore/router/sign: #40, #41, #42, #43 |
| 2026-06-10 | **PR-3.5 closed:** Gateway wallet read endpoints (core #44) + MCP real read tools & Python Dockerfile (mcp #22); full-stack compose w/o TLS (meta #11); docs sync (core #45, meta #12); `deny.toml`: ignore RUSTSEC-2026-0173; mcp smoke test rewritten for Python image |
| 2026-06-14 | **PR-6.1 DeFi positions:** clean-room port of v1 `agent-dapps` — core `crates/positions` (Aave v3 + ERC-4626) + `GetPositions` gRPC + gateway `/api/v1/wallet/positions` (core #52), mcp `get_positions` tool gated READ_WALLET (mcp #27). Live full-chain e2e vs a real mainnet Aave position. **MCP at parity with ClawHub.** |
| 2026-06-14 | **Phase 7 self-custody distribution:** onboarding `create_wallet`+recovery mnemonic (core #53, PR-7.3); public Core image → GHCR on `v*` tags (core #54, PR-7.2); mcp de-v1-ify (#28) + all-in-one stdio wallet image (#29, PR-7.1b). |
| 2026-06-14 | **Audit consumer P1 fixed + deployed:** resilient consumer (core #55) + block-compatible `response_timeout` (core #56); prod core rebuilt from `main` & redeployed (core+gateway), verified `audit stream read failed`=0, agent wallet `0x0C58…` intact. Found + worked around the gateway↔core startup race. |

---

## Repo Snapshot

| Repo | Visibility | Stack | State |
|------|-----------|-------|-------|
| `core` | Private | Rust 2024 (13 crates) | Phases 0–6.1 done + prod-deployed; real gRPC + Axum Gateway |
| `mcp` | Public | Python 3.12 + FastAPI + uv | Complete: protocol/SSE/stdio/capabilities + 6 tools real (incl. `get_positions`, PR-6.1), Docker image |
| `mobile` | Public | React Native 0.76 + TS | Scaffold only (placeholder App.tsx) |
| `llm` | Public (docs say private!) | TBD | Scaffold only; stack undecided |
| `meta` | Public | Docs / Docker | This file + specs |

---

## Session History (last 10)

| Session | What Was Done |
|---------|---------------|
| 2026-05-27 | Org creation, repo setup, CI/CD hardening, AGENTS.md, `crypto` + `keyring` complete |
| 2026-05-28 | Security hardening PR #6 + #7, audit Steps 1–5, `provider` + `txguard` scaffold, audit log |
| 2026-05-29 | `router` + `sign` scaffold, EIP-1559 integration, edition bump 2021→2024, MSRV 1.91 |
| 2026-05-30 | `audit` crate (PR #21), audit-router integration (PR #22), docs: CORE-API-SPEC + LLM-ARCHITECTURE |
| 2026-05-30 | gRPC server scaffold (PR #28), core-docker (PR #30) |
| 2026-05-31–06-01 | **mcp Python rewrite:** scaffold, protocol, SSE/stdio, capabilities, Gateway client (mcp #17–#20) |
| 2026-06-01–02 | **Gateway (Axum) Phase 2** (#31–#33), **Event Bus + Audit Phase 4** (#34, #36) |
| 2026-06-02–03 | **Phase 4.5 Core Real Integration:** WalletCore crate + full gRPC wiring (#38–#43) |
| 2026-06-10 | Workstation re-clone; docs sync (core #45, meta #12); **PR-3.5 closed** (core #44 + mcp #22, e2e verified via stdio); full-stack compose (meta #11); deny.toml RUSTSEC-2026-0173; mcp smoke test fix |
| 2026-06-14 | **PR-6.1 + Phase 7 + prod deploy:** DeFi positions (core #52, mcp #27); self-custody onboarding (core #53), GHCR publish workflow (core #54), mcp de-v1-ify (#28) + all-in-one image (#29); audit-consumer #55/#56 built & redeployed to prod (verified); docs sync (meta #21 overview + this status). |

---

## How to Update This File

At session end:
1. Move completed items to **Completed**
2. Update **In Progress**
3. Rewrite **Next Immediate Steps**
4. Add new blockers or mark resolved
5. Update date at top
