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

---

## In Progress 🔄

| Task | Repo | Blocker |
|------|------|---------|
| Phase 0 planning | `meta` | None — ready to start coding |

---

## Next Immediate Steps

1. **Read `~/Workspace/Codex/standards/rust.md`** — refresh Rust rules
2. **Audit old crypto code** — `~/Workspace/projects/rustok/crates/core/src/keyring/`, `crates/core/src/sign.rs`
3. **Create `crates/crypto/`** — scaffold crate, define traits
4. **Implement BIP-39 mnemonic generation** + golden tests (known vectors)
5. **Implement key derivation** (BIP-32/44) + golden tests
6. **Implement Secp256k1 signing** + golden tests
7. **Run `cargo deny check`** — validate no copyleft deps

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
