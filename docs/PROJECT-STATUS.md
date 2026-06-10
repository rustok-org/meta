# Project Status — Rustok Org

> Updated: 2026-06-10  
> Read this at session start to understand where we are.  
> Phase numbering follows `core/docs/CORE-MCP-ROADMAP.md` (source of truth for the MVP roadmap).

---

## Current Phase: MVP Roadmap Phases 0–4.5 ✅ COMPLETE → Phase 5 (Production Hardening) is next

**Roadmap goal:** Functional Core + Gateway + MCP Server = minimum viable product.

| Phase | Scope | Status |
|-------|-------|--------|
| 0 — Core Hardening | PrivateKey newtype, gas oracle, docs | ✅ #24, #26, #27 |
| 1 — Core as gRPC Service | Tonic server, Dockerfile, compose | ✅ #28/#29, #30 |
| 2 — Gateway (Axum) | scaffold, Core bridge, rate limit | ✅ #31, #32, #33 |
| 3 — MCP Server (Python) | scaffold, protocol, capabilities, Gateway client | ✅ mcp #17, #18, #19, #20; read path (PR-3.5) ✅ mcp #22 + core #44 |
| 4 — Event Bus + Audit | Redis Streams publisher, audit consumer | ✅ #34, #36 |
| 4.5 — Core Real Integration | gRPC stub → real wallet/router/sign/provider | ✅ #38, #40, #41, #42, #43 (plan #39) |
| 5 — Production Hardening | docker-compose-full, observability, postgres | 🔄 started — full compose w/o TLS ✅ meta #11; Traefik/SSL, observability, postgres pending |

**Core workspace (12 crates):** `types`, `crypto`, `keyring`, `provider`, `router`, `txguard`, `sign`, `audit`, `events`, `wallet`, `gateway`, `grpc` (~190 `#[test]` functions; all gates green at PR #43 merge, 2026-06-03).

**End-to-end chain works with real data:** MCP → Gateway → Core returns real address, balances, `tx_hash`, EIP-191 signatures (Phase 4.5 gate passed). All 5 MCP tools are real since 2026-06-10 (read path: Gateway `GET /api/v1/wallet/context` + `/balance`, core #44; MCP wiring, mcp #22 — verified end-to-end via stdio). Full stack runs via `meta/docker-compose.yml` (meta #11).

---

## In Progress 🔄

| Task | Repo | Blocker |
|------|------|---------|
| — (no active PRs; Phase 5 not started) | | |

---

## Next Immediate Steps

### Option A: Finish Phase 5 — Production Hardening (roadmap default)
1. **PR-5.1 remainder** — Traefik reverse proxy + SSL (Let's Encrypt) on top of `meta/docker-compose.yml` (full compose w/o TLS merged: meta #11)
2. **PR-5.2** `feat/observability` — Grafana + Loki + Tempo + Prometheus; OTel tracing Gateway → Core
3. **PR-5.3** `feat/postgres-migration` (optional)

### Backlog (from review round 2026-06-10, below single-PR threshold)
- Gateway positive-path tests: needs an in-process tonic test server (`CoreClient` is concrete, not a trait) — verify JSON serialization against proto changes
- `wallet_context` TTL cache on Gateway — **rejected for now** (tools are called on demand, stale balance risk); revisit with Phase 5 metrics

### Option B: Phase 5 security debt (deferred from 4.5)
1. Runtime/RPC unlock (today: startup-unlock only via `UnlockMethod`)
2. Policy + BudgetTracker (only execute budget-lock landed in 4.5.3)
3. Keystore file perms `0600`; receipt wait in `router::execute`; parallel balance fetch

### Option C: Begin LLM Brain (separate product line)
1. Resolve stack decision: Rust (Rig) vs Python (FastAPI + LangChain) — see Deferred Decisions
2. Define Intent JSON schema; scaffold in `rustok-org/llm`

---

## Blockers 🚧

| Blocker | Impact | Resolution |
|---------|--------|------------|
| GitHub Free — no branch protection for private repos (`core`) | Can accidentally push to `main` | **Mitigated** — pre-push hook + process discipline. |
| `uniffi` FFI bridge does not exist (no crate) | Mobile cannot call core | Low priority until Mobile phase. Needs `uniffi-bindgen-react-native` scaffold. |
| `simulateAssetChanges` not implemented in provider | Cannot preview swap/stake asset changes | Documented in `meta/docs/ALCHEMY-INTEGRATION.md`. Deferred to provider v2. |

---

## Deferred Decisions

| Decision | Options | Status |
|----------|---------|--------|
| `llm` stack | Rust (Rig) vs Python (FastAPI + LangChain) | **Under review.** `meta/docs/LLM-ARCHITECTURE.md` recommends Rust (Rig) with gRPC isolation fallback; reconcile at LLM kickoff. |
| `llm` repo visibility | AGENTS.md says private; repo is **public** on GitHub | Verify intent — flip visibility or fix docs. |
| ClawHub/Smithery legacy skill (`temrjan/rustok-wallet`, ~323 downloads) | Keep / republish on Python MCP | **Parked.** Do NOT delete until new MCP is republished. |
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

---

## Repo Snapshot

| Repo | Visibility | Stack | State |
|------|-----------|-------|-------|
| `core` | Private | Rust 2024 (12 crates) | Phases 0–4.5 done; real gRPC + Axum Gateway |
| `mcp` | Public | Python 3.12 + FastAPI + uv | Complete: protocol/SSE/stdio/capabilities + all 5 tools real, Docker image |
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

---

## How to Update This File

At session end:
1. Move completed items to **Completed**
2. Update **In Progress**
3. Rewrite **Next Immediate Steps**
4. Add new blockers or mark resolved
5. Update date at top
