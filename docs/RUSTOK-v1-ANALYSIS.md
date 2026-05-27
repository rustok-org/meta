# Rustok v1 Analysis — What to Keep, Drop, Rewrite

> Source: `~/Workspace/projects/rustok` (frozen, AGPL-3.0)
> Date: 2026-05-27

## Architecture Overview (v1)

**Rust workspace:** 10 crates
- `types` — shared DTOs
- `core` — wallet logic (keyring, provider, send, router, swap, explainer)
- `txguard` — transaction security (parser, rules, revm simulator, enrichment)
- `api` — Axum HTTP server
- `cli` — CLI binary
- `rustok-mobile-bindings` — uniffi FFI for React Native
- `agent-wallet` — separate AI-agent keystore
- `agent-dapps` — DeFi connectors (Aave, ERC-4626)
- `agent-mcp` — MCP server (HTTP + stdio)

**Mobile:** React Native 0.85.2, New Architecture, Zustand 5 + MMKV, React Navigation v7, uniffi bridge

## Keep ✅

| Decision | Why |
|----------|-----|
| **Workspace / multi-crate** | Clean boundaries, separate compilation, mobile can depend only on types+bindings |
| **uniffi bridge** | Proven pattern; RN ↔ Rust FFI with tokio runtime works |
| **txguard pipeline** | Parse → Rules → Verdict is correct mental model; keeps security central |
| **Custom TLS (rustls + webpki-roots)** | Fixes Android Let's Encrypt OCSP revocation bug |
| **Argon2id in spawn_blocking** | Correct — KDF is CPU-heavy, must not block async executor |
| **Zeroize + subtle** | Security baseline; secrets must not linger in memory |
| **Zustand + MMKV** | Fast, persistent, minimal boilerplate. Better than Redux/Context |
| **React Navigation v7** | Industry standard, type-safe, stable |
| **MCP dual transport** | HTTP for remote, stdio for Claude Desktop — correct flexibility |
| **Exhaustive docs** | Phase handoffs, incident reports, ADRs — keep the culture |
| **cargo-deny + cargo-audit** | License/CVE gate — already in CI |

## Drop ❌

| Decision | Why | Replacement |
|----------|-----|-------------|
| **AGPL-3.0 license** | Poison pill for commercial use; forces open-sourcing derivatives | **Proprietary** in `core`, MIT in `mcp` |
| **revm simulation** | 36MB dependency, slow compile, complex state forking | **Lightweight:** `eth_call` + Alchemy `simulateAssetChanges` |
| **Agent-wallet as separate crate** | Duplicates keyring logic, splits security model | **Capability-based permissions** inside core |
| **Tauri desktop app** | Deprecated, unmaintained, drift from mobile | **Kill it**; mobile is primary |
| **Single-wallet invariant** | Too restrictive for power users | **Single by default**, multi-wallet deferred to v2 |
| **GoPlus enrichment** | External API dependency, rate limits, privacy leak | **Defer** until txguard v2 |
| **Swap aggregation (1inch, 0x)** | Complex, requires API keys, fragile | **Defer** to dedicated `swap` crate later |
| **rusqlite audit log** | SQLite in Rust is fine, but overkill for MVP | **Append-only JSONL** or structured logs |

## Rewrite / Refactor 🔧

| Module | v1 State | v2 Plan |
|--------|----------|---------|
| `core/keyring` | Argon2id + AES-GCM, local file keystore | **Keep algorithm**, refactor to trait-based (`Keyring` trait) for testability |
| `core/provider` | `MultiProvider`, chain definitions | **Keep concept**, migrate to alloy 0.12+ provider builder pattern |
| `core/send` | Orchestrates txguard → sign → broadcast | **Keep flow**, but txguard is now lightweight |
| `core/sign` | alloy-signer-local | **Keep**, add hardware signer trait (Ledger, Keystone deferred) |
| `txguard/parser` | ABI decoding, calldata parsing | **Keep**, but remove revm dependency |
| `txguard/rules` | 8 rules engine | **Keep rules**, simplify engine to pure functions |
| `txguard/simulator` | revm + AlloyDB | **Replace** with `eth_call` wrapper + Alchemy simulation |
| `mobile/screens` | 16 screens, nested navigators | **Refactor**: chat-first UX (Armor/MoonPay pattern), dashboard secondary |
| `mobile/stores` | 8+ Zustand stores | **Consolidate** to 3-4 stores: wallet, ui, settings, chat |
| `mobile/components` | 16 design-system components | **Keep**, evolve for chat-first interface |
| `agent-mcp` | HTTP + stdio, own keystore | **Refactor**: use core keyring via capability tokens, not own crate |

## Dependency Upgrades

| Crate | v1 | v2 Target | Note |
|-------|-----|-----------|------|
| `alloy-*` | 1.8 | 0.12 | Newer API, better modularity |
| `revm` | 36 | — | **Remove** |
| `react-native` | 0.85.2 | 0.76+ | New scaffold uses 0.76; upgrade path TBD |
| `uniffi` | 0.31 | latest | Keep current |
| `axum` | 0.8 | 0.7 | Match scaffold |

## Critical Decisions for v2

1. **Chat-first UI** — main screen is conversation, not dashboard. Wallet actions are tools LLM invokes.
2. **Open hooks** — core exposes gRPC/REST public API + MCP adapter. External projects connect without forking.
3. **Capability-based security** — not "allow all" or "deny all", but per-action permissions (send ≤ $100, swap on whitelist, etc.)
4. **seL4-inspired isolation** — minimal TCB, component boundaries, zeroize. Not full seL4 runtime, but principles applied.
5. **Trunk-based development** — `main` only, PR + squash, conventional commits. No long-lived feature branches.

## Files to Reference (v1)

When rewriting, check these for logic (not copy-paste):
- `crates/core/src/keyring/` — encryption flow
- `crates/core/src/send.rs` — send orchestration
- `crates/txguard/src/rules/` — rule definitions
- `crates/txguard/src/parser/` — ABI decoding patterns
- `mobile/src/stores/walletStore.ts` — wallet state machine
- `crates/rustok-mobile-bindings/src/lib.rs` — uniffi export patterns
