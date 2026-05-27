# AGENTS.md — Rustok Organization

> This file governs `rustok-org/` and all sub-repositories.
> Deeper `AGENTS.md` files override these instructions for their subtrees.

---

## Project Identity

**Name:** Rustok Org  
**Mission:** Production-grade self-custody crypto wallet with LLM-first interface.  
**Status:** Phase 0 (foundation laying)  
**License:** `core` + `llm` = Proprietary; `mobile` + `mcp` + `meta` = MIT

---

## Repositories

| Repo | Visibility | Stack | Purpose |
|------|-----------|-------|---------|
| `core` | **Private** | Rust (workspace) | Wallet engine: crypto, keyring, provider, router, txguard, sign, swap |
| `mobile` | Public | React Native + TypeScript | Primary UI: chat-first wallet interface |
| `mcp` | Public | Rust | MCP server: HTTP + stdio dual transport |
| `llm` | **Private** | TBD | LLM agent integration layer |
| `meta` | Public | Markdown / Docker | Docs, standards, Docker Compose, API specs |

---

## Architecture Decisions (Frozen)

1. **Chat-first UX** — main screen is conversation, not dashboard. Wallet actions are tools LLM invokes. (Armor Wallet / MoonPay CLI pattern)
2. **Capability-based security** — per-action permissions (send ≤ $X, swap on whitelist), not blanket auth
3. **Lightweight txguard** — `eth_call` + Alchemy `simulateAssetChanges`. No `revm`.
4. **Open hooks** — core exposes gRPC/REST public API + MCP adapter. External projects connect without forking.
5. **seL4-inspired isolation** — minimal TCB, component boundaries, zeroize. Not full seL4 runtime.
6. **Mobile: TS = UI only, Rust = crypto** — all crypto via uniffi bridge. No JS crypto.
7. **Trunk-based development** — `main` only. PR + squash merge. Conventional commits.

---

## Security Principles

- `unsafe_code = "forbid"` in workspace lints
- `zeroize` for all secrets
- `cargo deny` blocks copyleft in `core`
- Argon2id KDF in `spawn_blocking`
- Custom `rustls` + `webpki-roots` TLS (Android OCSP fix)
- No secrets in code — only `.env` / GitHub Secrets

---

## Development Workflow

### Before writing ANY code

1. **Read Codex standard** → `~/Workspace/Codex/standards/<stack>.md`
2. **Read STANDARDS-MAP** → `meta/docs/STANDARDS-MAP.md` (overrides / decisions)
3. **Read v1 analysis** → `meta/docs/RUSTOK-v1-ANALYSIS.md` (what to keep/drop)
4. **Check CLAUDE.md** in target repo — current phase / blockers
5. **Plan changes** — explain what and why in ≤5 bullets

### Gates (non-negotiable)

**Rust:**
```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo deny check licenses bans sources
```

**React Native:**
```bash
npm run typecheck
npm run lint
npm test
```

### Commits

- Conventional commits: `feat:`, `fix:`, `refactor:`, `test:`, `docs:`, `ci:`, `chore:`
- Atomic commits — one logical change per commit
- Messages in English

### Pull Requests

- Small PRs (< 300 lines, < 1 hour review)
- Fill PR template checklist
- Squash merge only
- Delete branch on merge

---

## Standards & References

| Document | Purpose |
|----------|---------|
| `~/Workspace/Codex/standards/_INDEX.md` | Map of all Codex standards |
| `meta/docs/STANDARDS-MAP.md` | Which standard → which repo + our overrides |
| `meta/docs/RUSTOK-v1-ANALYSIS.md` | Keep / drop / rewrite from old codebase |
| `meta/docs/CORE-API-SPEC.md` | Core module boundaries and public API |
| `meta/docs/LLM-ARCHITECTURE.md` | LLM-first UX research and proposal |

---

## Old Codebase

- **Location:** `~/Workspace/projects/rustok`
- **Status:** FROZEN — read-only reference. Do NOT modify.
- **License:** AGPL-3.0 (incompatible with new proprietary core)
- **Usage:** Reverse-engineer concepts, not copy-paste code. Check `RUSTOK-v1-ANALYSIS.md` for mapping.

---

## Session Start Checklist

```bash
# 1. Check where we are
git status
git log --oneline -5

# 2. Read project status
 cat meta/docs/PROJECT-STATUS.md

# 3. Read relevant standard
# (see STANDARDS-MAP.md)

# 4. Start task
```

**At session end:** update `meta/docs/PROJECT-STATUS.md` with what was done and what's next.

---

## Emergency Contacts

- **Standards repo:** `~/Workspace/Codex/standards/`
- **Pipeline rules:** `~/Workspace/Codex/standards/pipeline.md`
- **Session close skill:** `~/.claude/skills/session-close/SKILL.md`
