# Project Status — Rustok Org

> Updated: 2026-05-27  
> Read this at session start to understand where we are.

---

## Current Phase: Phase 0 (Foundation)

**Goal:** Core compiles, crypto passes tests, spec frozen.

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

---

## In Progress 🔄

| Task | Repo | Blocker |
|------|------|---------|
| Phase 0 planning | `meta` | None — ready to start coding |

---

## Next Immediate Steps

1. **Create `crates/keyring/`** — Argon2id + AES-256-GCM keystore encryption
2. **Create `crates/provider/`** — Multi-chain RPC provider (Alchemy primary)
3. **Create `crates/txguard/`** — Lightweight rule engine (eth_call + Alchemy simulate)
4. **Update `core/Cargo.toml`** — add remaining workspace members as crates are created
5. **Run `cargo deny check`** after each new crate — validate no copyleft deps

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
| `llm` stack | Rust (Tokio + axum) vs Python (FastAPI) vs Node (Express) | After Phase 0, when LLM architecture is clearer |
| React Native version | 0.76 (current scaffold) vs 0.85 (old code) vs latest | Before mobile Phase 1 |
| Hardware signers | Ledger, Keystone, air-gapped | After core v1.0 |
| Swap integration | 1inch, 0x, CoW | After txguard v2 |

---

## Session History (last 5)

| Session | What Was Done |
|---------|---------------|
| 2026-05-27 | Org creation, repo setup, CI/CD hardening, AGENTS.md, docs |

---

## How to Update This File

At session end:
1. Move completed items to **Completed**
2. Update **In Progress**
3. Rewrite **Next Immediate Steps**
4. Add new blockers or mark resolved
5. Update date at top
