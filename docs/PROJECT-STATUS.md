# Project Status — Rustok Org

> Updated: 2026-05-27  
> Read this at session start to understand where we are.

---

## Current Phase: Phase 1 — Core Scaffold (~55%)

**Goal:** All core crates compile and pass tests. Alchemy integration spec-ready. Cross-compile Android/iOS green.

- `types/` ✅ — DTOs + unified `CoreError` (9 variants)
- `crypto/` ✅ — BIP-39, BIP-32/44, Secp256k1, golden + property tests
- `keyring/` ✅ — Argon2id + AES-256-GCM, `LocalKeyring`, `SecretString` API
- `provider/` 🔄 — scaffold ready, awaiting PR merge
- `router/` ⏳ — pending
- `txguard/` ⏳ — pending
- `sign/` ⏳ — pending
- `audit/` ⏳ — pending
- `uniffi/` ⏳ — pending

---

## Completed ✅

| Date | Task |
|------|------|
| 2026-05-27 | Created `rustok-org` GitHub organization |
| 2026-05-27 | Created 5 repos: `core` (private), `mobile`, `mcp`, `llm` (private), `meta` |
| 2026-05-27 | Pushed initial scaffold to all repos |
| 2026-05-27 | Configured branch protection (public repos: PR + 1 approval required) |
| 2026-05-27 | Hardened CI/CD: concurrency, pinned SHA actions, caching, security audits, coverage |
| 2026-05-27 | Added Dependabot, CODEOWNERS, PR templates |
| 2026-05-27 | Created `STANDARDS-MAP.md` — Codex → repo mapping |
| 2026-05-27 | Created `RUSTOK-v1-ANALYSIS.md` — keep/drop/rewrite from old codebase |
| 2026-05-27 | Created `AGENTS.md` in all repos — AI session onboarding |
| 2026-05-27 | **Phase 0: `crates/crypto`** — BIP-39 mnemonic, BIP-32/44 derivation, Secp256k1 signing (EIP-191, EIP-712), golden + property tests, full docs, security hardening (Phrase zeroizes internal Mnemonic), all gates green |
| 2026-05-27 | **Phase 0: `crates/keyring`** — Argon2id + AES-256-GCM keystore, `LocalKeyring`, import/export, 17 tests, all gates green |
| 2026-05-27 | **Security Hardening PR #5 (`core`)** — Argon2id params 64 MiB/3 iters, `OsRng` for salt/nonce, `secrecy::SecretString` for passwords, `CoreError` expansion (9 variants), removed `.expect()` from `mnemonic::to_seed`, extracted `vec_to_private_key` helper, cleaned up deps |
| 2026-05-27 | **Security Hardening PR #8 (`mcp`)** — `alloy-primitives` aligned to 1.6 (workspace sync), dependabot ignore rules verified |
| 2026-05-27 | **`/review-codex`** — Full rust-review + security-review on PR #5 + #8. 12 issues found/fixed. 0 critical, 0 high. Verdict: PASS |
| 2026-05-27 | **Phase 0: Dependabot policy** — ignore rules for major GitHub Actions bumps in `core`, `mcp`, `mobile`; documented in `STANDARDS-MAP.md` |

---

## In Progress 🔄

| Task | Repo | Blocker |
|------|------|---------|
| Phase 1: `crates/provider/` | `core` | Awaiting merge of PR #5 → `main` |
| Merge PR #5 (`fix/security-hardening-phase1`) | `core` | Human approval required (branch protection) |
| Merge PR #8 (`fix/alloy-primitives-alignment`) | `mcp` | Human approval required |

---

## Next Immediate Steps

1. **Create `crates/provider/`** — MultiProvider (Alchemy → Infura → public node fallback), chain config, RPC client
2. **Create `crates/txguard/`** — Rule engine + Alchemy `simulateAssetChanges` integration, risk levels
3. **Create `crates/router/`** — Send, nonce management, gas estimation, atomic ops
4. **Create `crates/sign/`** — EIP-191, EIP-712, generic tx signing abstraction
5. **Create `crates/audit/`** — Append-only SQLite WAL log, every action recorded
6. **Create `crates/uniffi/`** — FFI bridge for mobile, thin wrapper over keyring + provider
7. **Update `core/Cargo.toml`** — add remaining workspace members as crates are created
8. **Cross-compile check** — Android (NDK) + iOS after `uniffi` is ready
9. **Run `cargo deny check`** after each new crate — validate no copyleft deps

---

## Blockers 🚧

| Blocker | Impact | Resolution |
|---------|--------|------------|
| GitHub Free — no branch protection for private repos (`core`, `llm`) | Can accidentally push to `main` | Pre-commit hooks + self-discipline. Buy GitHub Pro later. |
| No `package-lock.json` in `mobile` initially | CI used `npm install` instead of `npm ci` | **Fixed** — added `.npmrc` + `package-lock.json` |
| Mobile peer dependency conflict | `npm install` fails without `--legacy-peer-deps` | **Fixed** — `.npmrc` with `legacy-peer-deps=true` |

---

## Deferred Decisions

| Decision | Options | When to Decide |
|----------|---------|----------------|
| `llm` stack | ~~Rust (Tokio + axum) vs Python (FastAPI) vs Node (Express)~~ | **Resolved:** Rust (Rig framework) |
| React Native version | 0.76 (current scaffold) vs latest | **Resolved:** Stay on 0.76 for now; evaluate upgrade at Phase 4 kickoff |
| Hardware signers | Ledger, Keystone, air-gapped | After core v1.0 |
| Swap integration | 1inch, 0x, CoW | After txguard v2 |

---

## Session History (last 5)

| Session | What Was Done |
|---------|---------------|
| 2026-05-27 | Org creation, repo setup, CI/CD hardening, AGENTS.md, docs |
| 2026-05-27 | `crypto` + `keyring` crates completed, reviewed, gates green |
| 2026-05-27 | Dependabot major-bump ignore policy implemented |
| 2026-05-27 | Security hardening + review-codex — PR #5 & #8 ready for merge |

---

## How to Update This File

At session end:
1. Move completed items to **Completed**
2. Update **In Progress**
3. Rewrite **Next Immediate Steps**
4. Add new blockers or mark resolved
5. Update date at top
