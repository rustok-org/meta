# Standards Map — Rustok Org

> ⚠️ **Частично устарело — до разворота 2026-06-25 (custodial-эра).** Части про «ключ/подпись
> на сервере» неактуальны: продукт развёрнут на non-custodial device-signing. Текущее состояние —
> `PROJECT-STATUS.md`; незыблемое — `core/.claude/NORTH-STAR.md`. Держится как исторический контекст.

> Quick reference: which Codex standard applies to which repo.
> Full standards live in `~/Workspace/Codex/standards/` — read them before coding.

| Repo | Primary Standards | Read Before |
|------|-------------------|-------------|
| `core` | [`rust.md`](../../Codex/standards/rust.md) → [`architecture.md`](../../Codex/standards/architecture.md) | Every Rust session |
| `mobile` | [`react.md`](../../Codex/standards/react.md) → [`typescript.md`](../../Codex/standards/typescript.md) | Every TS/RN session |
| `mcp` | [`rust.md`](../../Codex/standards/rust.md) | Every Rust session |
| `llm` | [`rust.md`](../../Codex/standards/rust.md) → Rig framework | Every Rust session |
| `meta` | [`devops.md`](../../Codex/standards/devops.md) | Every infra session |
| **All** | [`pipeline.md`](../../Codex/standards/pipeline.md) | Every session start |

## Key Decisions (Codex → Rustok)

| Codex Rule | Rustok Override |
|------------|-----------------|
| `mod.rs` discouraged (Rust 2024) | **Override:** use `mod.rs` for crate roots (clearer for multi-file modules) |
| `anyhow` for binaries | **Override:** `thiserror` everywhere (core is lib-first; binaries are thin wrappers) |
| `reqwest` default TLS | **Override:** custom `rustls` + `webpki-roots` (Android OCSP issue) |
| React Compiler auto-memo | **Override:** manual `useMemo`/`useCallback` until RN 0.85+ supports React Compiler |
| Default exports allowed | **Override:** named exports only (Codex agrees, old rustok mixed) |
| Zustand slices pattern | **Adopt:** old rustok used slices, keep it |
| `unsafe` with `// SAFETY:` | **Adopt:** strict; seL4-inspired minimal TCB |
| `cargo-deny` + `cargo-audit` | **Adopt:** already in CI; block copyleft in `core` |

## Dependabot & CI Maintenance Policy

| Decision | Rule |
|----------|------|
| **GitHub Actions** | Major version bumps (e.g. v4 → v7) are **ignored** by Dependabot. Updated manually in dedicated CI maintenance sprints only. |
| **Minor / Patch** | Dependabot groups and auto-PRs weekly. Reviewer checks for security advisories before merge. |
| **Rust crates** | Dependabot groups minor + patch. Major bumps require human review and gate run. |
| **npm packages** | Same as Rust — grouped minor/patch, major deferred unless security advisory. |

Rationale: major bumps of GitHub Actions often introduce breaking changes (artifact format, input names, permissions). They must be tested end-to-end, not merged ad-hoc.

## Old Codebase Analysis

See [`RUSTOK-v1-ANALYSIS.md`](./RUSTOK-v1-ANALYSIS.md) for detailed critique of `~/Workspace/projects/rustok`.
