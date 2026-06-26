# Project Status — Rustok Org

> Updated: 2026-06-26  
> Read this at session start to understand where we are.  
> Phase numbering follows `core/docs/CORE-MCP-ROADMAP.md` (source of truth for the MVP roadmap).

---

> ⚠️ **ПИВОТ 2026-06-25 → non-custodial device-signing.** Продукт развёрнут на on-device подпись
> (ключ + аппрув на телефоне, Android-first; MCP = proposer, **keyless**). Источник истины:
> ADR `core/.claude/decisions/2026-06-26-pivot-noncustodial-device-signing.md` + MASTER PLAN
> `core/.claude/specs/2026-06-25-epic-noncustodial-device-signing-PLAN.md`.
> Статус ниже (Phases 0–7, custodial-стек на `api.rustokwallet.com`, cutover 2026-06-14) — теперь
> **исторический контекст custodial-эпохи**: custodial-прод = **F8-кандидат на ретайр** (ОТЛОЖЕНО,
> «решить позже»; при ретайре — миграция ClawHub-юзеров). Сервер → pure-proposer keyless на Этапе 3.
> **Эпик-прогресс: Этап 0 ЗАКРЫТ** (core PR #74, main `23bf736`: ADR + скелет `mobile-bindings`).

---

## Current Phase: Phases 0–7 ✅ — self-custody distribution SHIPPED (ClawHub skill 0.3.0, public image); prod current — **исторический/custodial; см. ПИВОТ-врезку выше**

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
| 7 — Self-Custody Distribution | onboarding, public image, all-in-one MCP image, ClawHub republish | ✅ **SHIPPED 2026-06-15** — onboarding core #53; capability-grant fix mcp #30; de-v1-ify mcp #28/#31; all-in-one stdio image mcp #29; EULA core #58 (RF); SHA-pin fix core #59/mcp #33; images on GHCR (`rustok-wallet` public, `rustok-core` private); **ClawHub skill 0.3.0** (mcp #34, > 0.2.2), smoke-verified |
| Production cutover | new stack on `api.rustokwallet.com` | ✅ done 2026-06-14; audit-consumer #55/#56 + gateway lazy-channel #57 built & deployed; monolith stopped (retained for rollback) |

**Core workspace (12 crates):** `types`, `crypto`, `keyring`, `provider`, `router`, `txguard`, `sign`, `audit`, `wallet`, `gateway`, `grpc`, `positions` (PR-6.1) (~190 `#[test]` functions as of PR #43; positions added more). The Phase-4 `events` crate (Redis Streams) was removed 2026-06-17 (core #63/#64).

**End-to-end chain works with real data:** MCP → Gateway → Core returns real address, balances, `tx_hash`, EIP-191 signatures (Phase 4.5 gate passed). All 5 original MCP tools are real since 2026-06-10 (read path: Gateway `GET /api/v1/wallet/context` + `/balance`, core #44; MCP wiring, mcp #22 — verified end-to-end via stdio). Full stack runs via `meta/docker-compose.yml` (meta #11).

**DeFi positions shipped (PR-6.1, 2026-06-14):** a 6th MCP tool `get_positions` (Aave v3 + ERC-4626) — new core `crates/positions` + `GetPositions` gRPC + gateway `GET /api/v1/wallet/positions` (core #52), mcp tool gated by READ_WALLET (mcp #27). Clean-room port of v1 `agent-dapps`; best-effort, no price oracle. Verified live full-chain (MCP → Gateway → Core) against a real mainnet Aave v3 position. **The org MCP has replaced the legacy ClawHub distribution: `temrjan/rustok-wallet` is republished at 0.3.2 (Docker/stdio, `ghcr.io/rustok-org/rustok-wallet`); the namespace stays `@temrjan` to preserve the install base. The old npm package `rustok-agent-mcp` (v1, AGPL) is left as-is — npm is not a distribution channel for the new wallet, and the obsolete package is harmless.**

---

## In Progress 🔄

| Task | Repo | Blocker |
|------|------|---------|
| — (Phase 7 shipped; only cleanup tails remain — see Next Immediate Steps) | | |

---

## Next Immediate Steps

### Primary: cleanup tails (Phase 7 shipped)
1. ~~**GH Actions Node 20→24**~~ ✅ **done** — core `docker-publish` (#66), meta `checkout` v6.0.3 (#23); mcp workflows already pin Node-24-era actions (`actions/checkout@v6.0.3`, `docker/build-push-action@v7.2.0`).
2. ~~**Reproducibility:** commit deploy-v2 compose + `Caddyfile-v2` into `meta`~~ ✅ **done** (meta #24; Caddyfile-v2 security headers + body cap #26).
3. **Remove the old monolith** (`rustok-api`) after 24–48h prod stability (cutover 2026-06-14) — *server-side, verify.*
4. **Alchemy RPC:** swap the public RPCs for an Alchemy key (needs the key) —
   `/opt/rustok/deploy-v2/.env` → `docker compose up -d core`.
5. **Verify prod is on the post-06-15 core** (#63/#64 Redis removal, #65 gateway hardening, #67 graceful exit) — the repo carries them; confirm the running stack matches and Redis is actually gone from the host.
6. **Re-enable GHCR publish workflows** once Actions minutes reset (manual local push was a one-off; mcp `wallet-publish` is currently manual-trigger #45).

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
| GitHub Free — no branch protection for private repos (`core`) | Can accidentally push to `main` | **Mitigated** — pre-push hook + process discipline. |
| `uniffi` FFI bridge does not exist (no crate) | Mobile cannot call core | Low priority until Mobile phase. Needs `uniffi-bindgen-react-native` scaffold. |
| `simulateAssetChanges` not implemented in provider | Cannot preview swap/stake asset changes | Documented in `meta/docs/ALCHEMY-INTEGRATION.md`. Deferred to provider v2. |

---

## Deferred Decisions

| Decision | Options | Status |
|----------|---------|--------|
| `llm` stack | Rust (Rig) vs Python (FastAPI + LangChain) | **Under review.** `meta/docs/LLM-ARCHITECTURE.md` recommends Rust (Rig) with gRPC isolation fallback; reconcile at LLM kickoff. |
| `llm` repo visibility | AGENTS.md says private; repo is **public** on GitHub | Verify intent — flip visibility or fix docs. |
| ClawHub/Smithery legacy skill (`temrjan/rustok-wallet`, ~323 downloads) | Keep / republish on Python MCP | **Resolved (2026-06-25).** Republished at 0.3.2 on the new MCP (Docker/stdio image); namespace stays `@temrjan` to preserve the install base. Legacy npm `rustok-agent-mcp` (v1, AGPL) left as-is (obsolete/harmless) — npm is not a distribution channel for the new wallet; the `publish-rustok` token is obsolete. |
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
| 2026-06-15 | **"Replace ClawHub" SHIPPED:** capability-grant fix for standard stdio clients (mcp #30) + README de-v1 (mcp #31); proprietary EULA, governing law RF (core #58); gateway↔core lazy/reconnecting channel (core #57) — race fixed & **deployed to prod** (`/health` core:serving, no manual restart); publish-workflow SHA-pin fix (core #59, mcp #33); skill version 0.3.0 (mcp #34). Images built locally + pushed to GHCR (`rustok-wallet` **public**, `rustok-core` private; Actions minutes were exhausted); **ClawHub skill republished at 0.3.0** (> 0.2.2); public image smoke-verified (anon pull → 6 tools). |
| 2026-06-17→19 | **Redis decommission + post-ship hardening:** authoritative SQLite audit, **Redis event path removed** (core #63, 06-17), **Redis decommissioned from the new stack** (core #64, 06-17; deploy-v2 meta #25, 06-17) → `events` crate dropped (**13→12 crates**); gateway startup + CORS hardening (core #65, I6/I7, 06-18); graceful exit on an unknown `core-server` subcommand (core #67, 06-18); CI to **Node 24** docker-publish (core #66) + drop Coverage & macOS/iOS cross-check (core #68, 06-18); Caddyfile-v2 security headers & body cap (meta #26, 06-18); mcp — skill 0.3.1 (#41) / 0.3.2 (#44), keyring-password-in-argv fix (#42), ephemeral MCP API key in entrypoint (direct commit `2dc3bb9`), `wallet-publish` manual-trigger (#45, 06-19). *(Repo state; confirm prod redeploy of the core changes.)* |

---

## Repo Snapshot

| Repo | Visibility | Stack | State |
|------|-----------|-------|-------|
| `core` | Private | Rust 2024 (12 crates) | Phases 0–7 done; prod-deployed (gateway lazy-channel #57); Redis decommissioned (#63/#64); proprietary EULA (RF); image on GHCR (private) |
| `mcp` | Public | Python 3.12 + FastAPI + uv | Complete: 6 tools, stdio process-trusted (all caps by default); all-in-one wallet image **public on GHCR**; ClawHub skill **0.3.0** |
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
| 2026-06-15 | **"Replace ClawHub" shipped + tails:** capability fix (mcp #30), README de-v1 (#31), EULA RF (core #58), gateway lazy-channel (core #57, deployed to prod), publish-workflow SHA fix (core #59/mcp #33), skill 0.3.0 (mcp #34); GHCR images (wallet public / core private) via local build+push (Actions minutes hit); ClawHub republished 0.3.0; docs sync (meta this PR). |

---

## How to Update This File

At session end:
1. Move completed items to **Completed**
2. Update **In Progress**
3. Rewrite **Next Immediate Steps**
4. Add new blockers or mark resolved
5. Update date at top
