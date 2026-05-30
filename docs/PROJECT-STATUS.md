# Project Status — Rustok Org

> Updated: 2026-05-31  
> Read this at session start to understand where we are.

---

## Current Phase: Phase 1 — Core Scaffold ✅ COMPLETE

**Goal:** All core crates compile and pass tests. Alchemy integration spec-ready. Cross-compile Android/iOS green.

| Crate | Status | Tests | Note |
|-------|--------|-------|------|
| `types/` | ✅ | 7 | DTOs + `CoreError` (9 variants), `AuditEvent` DTOs added |
| `crypto/` | ✅ | 32 | BIP-39, BIP-32/44, Secp256k1, golden + property tests, low-s normalization, **PrivateKey newtype** |
| `keyring/` | ✅ | 19 | Argon2id + AES-256-GCM, `LocalKeyring`, import/export, label persistence |
| `provider/` | ✅ | 14 | Alchemy/Infura/Public, MultiProvider, retry, circuit breaker, **get_fee_data** |
| `router/` | ✅ | 9 | Builder, preview, execute, broadcast, receipt tracking, **audit integration**, **gas auto-fetch** |
| `txguard/` | ✅ | 24 | Parser (ERC-20/721/Permit), rules engine, lightweight simulator |
| `sign/` | ✅ | 18 | `Signer` trait, `LocalSigner`, `CompositeSigner`, EIP-191/712/legacy/EIP-1559 |
| `audit/` | ✅ | 4 | SQLite WAL append-only log, router integration (execute + preview) |
| `uniffi/` | ⏳ | 0 | Placeholder directory, no `Cargo.toml` yet |
| `mcp-server/` | ⏳ | 0 | Not started — Phase 2 product target |

**Total tests:** 127 passed. All CI gates green (`fmt`, `clippy`, `test`, `deny`, `audit`).

---

## Completed ✅

| Date | Task |
|------|------|
| 2026-05-27 | Created `rustok-org` GitHub organization |
| 2026-05-27 | Created 5 repos: `core` (private), `mobile`, `mcp`, `llm` (private), `meta` |
| 2026-05-27 | Pushed initial scaffold to all repos |
| 2026-05-27 | Hardened CI/CD: concurrency, pinned SHA actions, caching, security audits, coverage |
| 2026-05-27 | Created `STANDARDS-MAP.md`, `RUSTOK-v1-ANALYSIS.md`, `AGENTS.md` in all repos |
| 2026-05-27 | **Phase 0: `crates/crypto`** — BIP-39, BIP-32/44, Secp256k1, golden + property tests |
| 2026-05-27 | **Phase 0: `crates/keyring`** — Argon2id + AES-256-GCM, `LocalKeyring`, 17 tests |
| 2026-05-28 | **Security Hardening PR #6 (`core`)** — low-s normalization (`EIP-2`), `Seed` timing fix, `subtle` dev-dep |
| 2026-05-28 | **Security Hardening PR #7 (`core`)** — `aes-gcm` zeroize feature, error masking in keyring |
| 2026-05-28 | **Security audit Steps 1–5** — 7/9 findings fixed, 2 backlogged. Audit log created |
| 2026-05-28 | **`crates/provider/` scaffold** — AlchemyProvider, InfuraProvider, PublicRpcProvider, MultiProvider |
| 2026-05-28 | **`crates/txguard/` scaffold** — parser, rules engine, simulator. 24 tests |
| 2026-05-28 | **Design doc:** `docs/PROVIDER-DESIGN.md`, `docs/SECURITY-AUDIT-PLAYBOOK.md` |
| 2026-05-29 | **`crates/router/` scaffold** — builder, preview, execute. `docs/PHASE1-HANDOFF.md` updated |
| 2026-05-29 | **`crates/sign/` scaffold** — `Signer` trait, `LocalSigner`, `CompositeSigner`. 18 tests |
| 2026-05-29 | **PR #20** — EIP-1559 signing, gas validation, `keyring::to_signer` integration |
| 2026-05-29 | **Edition 2021 → 2024**, MSRV 1.85 → 1.91, `reqwest` 0.12 → 0.13 |
| 2026-05-30 | **`crates/audit/` scaffold (PR #21)** — SQLite WAL, `AuditLog` trait, `SqliteAuditLog`. 4 tests |
| 2026-05-30 | **Audit-router integration (PR #22)** — `execute` + `preview` log `Success`/`Failed`/`PreviewBlocked` |
| 2026-05-30 | **Docs updated** — `meta/docs/CORE-API-SPEC.md`, `meta/docs/LLM-ARCHITECTURE.md` enhanced |
| 2026-05-30 | **`core/AGENTS.md`** updated to include `audit/`, corrected `router/` description |
| 2026-05-30 | **`core/docs/PHASE1-HANDOFF.md`** updated — Phase 1 marked complete, 116 tests, next steps defined |
| 2026-05-30 | **PR #24 (`core`)** — `PrivateKey` newtype struct, zeroization, no `PartialEq`. 4 new unit tests. Security debt closed |
| 2026-05-30 | **PR #26 (`core`)** — Router gas auto-fetch via `Provider::get_fee_data()`. `FeeData` newtype with `from_gas_price()`. 3 new tests |

---

## In Progress 🔄

| Task | Repo | Blocker |
|------|------|---------|
| `uniffi` placeholder → real FFI bridge | `core` | Needs `MOBILE-UX-WIREFRAMES.md` + mobile team alignment |
| Phase 0 docs completion | `meta` | `ALCHEMY-INTEGRATION.md` ✅ exists, `MOBILE-UX-WIREFRAMES.md` ✅ exists — need review |

---

## Next Immediate Steps

### Option A: Close Phase 0 formally (recommended)
1. Review + approve `meta/docs/CORE-API-SPEC.md` and `meta/docs/LLM-ARCHITECTURE.md`
2. Review `meta/docs/ALCHEMY-INTEGRATION.md` + `meta/docs/MOBILE-UX-WIREFRAMES.md`
3. Mark Phase 0 as complete
4. Proceed to Phase 2: LLM Brain scaffold

### Option B: Security debt + dependency cleanup ✅ IN PROGRESS
1. ✅ Fix `PrivateKey` `PartialEq` timing leak (newtype struct) — **PR #24 merged**
2. Fix `derive_key` error masking (`argon2::Error`)
3. Evaluate `bip32` / `coins-bip39` upgrade path
4. Clean `cargo deny` duplicate warnings (`base64`, `block-buffer`, `digest`)

### Option C: Begin Phase 2 LLM Brain
1. Create `rustok-org/llm` repo scaffold
2. Define Intent JSON schema
3. Write 50 golden tests for Intent Parser
4. Scaffold FastAPI + LangGraph (or Rust Rig)

### Option D: Mobile bridge
1. Create `crates/uniffi/Cargo.toml`
2. Add `uniffi` to workspace members
3. Expose minimal FFI: `wallet_create`, `tx_preview`, `tx_execute`

---

## Blockers 🚧

| Blocker | Impact | Resolution |
|---------|--------|------------|
| GitHub Free — no branch protection for private repos (`core`, `llm`) | Can accidentally push to `main` | **Mitigated** — pre-push hook + process discipline. Direct push to main occurred on 2026-05-30 and was reverted. |
| `simulateAssetChanges` not implemented in provider | Cannot preview swap/stake asset changes | Documented in `meta/docs/ALCHEMY-INTEGRATION.md`. Deferred to Phase 5 or provider v2. |
| `uniffi` directory is empty | Mobile cannot call core yet | Needs scaffold + `uniffi-bindgen-react-native` setup. Low priority until Phase 4. |

---

## Deferred Decisions

| Decision | Options | Status |
|----------|---------|--------|
| `llm` stack | Rust (Rig) vs Python (FastAPI + LangChain) | **Under review.** `meta/docs/LLM-ARCHITECTURE.md` recommends Python for prototyping, Rust for v1.0. `meta/AGENTS.md` previously resolved Rust (Rig). Reconcile at Phase 2 kickoff. |
| React Native version | 0.76 (current) vs latest | **Resolved:** Stay on 0.76 until Phase 4. |
| Hardware signers | Ledger, Keystone, air-gapped | After core v1.0 |
| Swap integration | 1inch, 0x, CoW | After txguard v2 + `simulateAssetChanges` |

---

## Session History (last 10)

| Session | What Was Done |
|---------|---------------|
| 2026-05-27 | Org creation, repo setup, CI/CD hardening, AGENTS.md, `crypto` + `keyring` complete |
| 2026-05-28 | Security hardening PR #6 + #7, audit Steps 1–5, `provider` + `txguard` scaffold, audit log |
| 2026-05-29 | `router` + `sign` scaffold, EIP-1559 integration, edition bump 2021→2024, MSRV 1.91 |
| 2026-05-30 | `audit` crate (PR #21), audit-router integration (PR #22), docs: CORE-API-SPEC + LLM-ARCHITECTURE, AGENTS.md + PHASE1-HANDOFF updated |

---

## How to Update This File

At session end:
1. Move completed items to **Completed**
2. Update **In Progress**
3. Rewrite **Next Immediate Steps**
4. Add new blockers or mark resolved
5. Update date at top
