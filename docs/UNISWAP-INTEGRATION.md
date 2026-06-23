# Uniswap Integration Plan

**Reference integration: Uniswap × Rustok Wallet — the self-custody signing layer for agent-driven Uniswap execution.**

> **Living document, not frozen law.** We follow Highest Effectiveness, not the plan when the two diverge — but every deviation is surfaced with a falsifiable reason and written back here (no silent drift). See [Principle: effectiveness over plan](#principle-effectiveness-over-plan).

---

## Status (2026-06-23)

- **Core EIP-712 signer — DONE.** `SignTypedData(domain_separator, struct_hash)` gRPC RPC signs locally and audits (`core/crates/grpc/src/server.rs`). Keys never leave core; fail-closed when the wallet is locked. The agent-facing path (`mcp → gateway → eip712`) is intentionally **not yet wired** to the agent surface; it opens only behind the safety spine.
- **Order → digest verifier — DONE for post-fill; quote-time path is the PR-2 target.** `rustok-org/uniswap` `encode.ts` reproduces the EIP-712 digest from a `CosignedV2DutchOrder`'s `encodedOrder` (post-fill — shipped in T2). A spike (2026-06-23, live Trading API `/quote`) showed the **quote-time** order is an `UnsignedV2DutchOrder` whose re-encode reproduces the API's `permitData` digest **byte-for-byte** — proven *feasible*, but **not yet in `encode.ts`**; it lands in PR-2. So reconstruct-before-sign is end-to-end-feasible, yet wired in code only for post-fill today.
- **Spike facts folded in:** the quote-time order is an `UnsignedV2DutchOrder` (pre-cosign), **not** `CosignedV2DutchOrder` (which `encode.ts` currently targets); routing is size-dependent (UniswapX wins at larger sizes, small swaps route CLASSIC).
- **Next:** the safety spine (gates) and the end-to-end vertical. See [Delivery roadmap](#delivery-roadmap).

---

## Why this exists

Uniswap's agent stack **plans and builds** transactions but leaves **signing and custody to the user** (bring-your-own key). Rustok is a self-custody Ethereum agent wallet (local Docker image, keys never leave the machine, MCP over stdio, `txguard`). This repo is the glue: Uniswap builds the order/transaction → Rustok risk-checks and signs under human approval. It makes agent-driven Uniswap execution end-to-end with self-custody.

---

## How the Uniswap side actually works

- **uniswap-ai skills** (`swap-integration`, `liquidity-planner`, `viem-integration`, `v4-security-foundations`, …) are **coding-agent assistants** that help *wire* the integration. They are not a runtime signing service, and Uniswap ships **no secure signer** — signing is left to the developer's wallet plus a prompt-level confirm. That gap is exactly what Rustok fills (a programmatic risk gate + key isolation).
- **Runtime execution** comes from:
  - **UniswapX** — off-chain, **EIP-712-signed orders** filled by resolvers (gasless, MEV-protected). → our primary swap path.
  - **Uniswap Trading API** (`trade-api.gateway.uniswap.org/v1`, `x-api-key`) — for UniswapX routes, `/quote` returns `quote.encodedOrder` + a ready-to-sign `permitData {domain, types, values}`; you sign, then `POST /swap`. → our order source (verified, never trusted blindly — see invariants).
  - **Universal Router SDK** — on-chain swap calldata. → on-chain path (roadmap).
  - **v4 SDK `PositionManager`** — LP add/remove calls. → LP execution (roadmap).

---

## The real dependency: Rustok's signing gap

Rustok signs **ETH sends + EIP-191 messages**, and now **EIP-712 typed-data** (`SignTypedData`, done). The two execution paths need different primitives:

| Path | Primitive needed in Rustok | Size | Safety surface |
|---|---|---|---|
| **UniswapX order** | **EIP-712 typed-data signing** (DONE) | small | clean — the glue/orchestrator decodes the order (tokens, minOut, deadline, recipient) and gates it *before* requesting a signature; core signs two hashes and never sees the order (`txguard` only flags, never blocks) |
| On-chain (Universal Router / PositionManager) | general **contract-call write-path** | large, net-new | harder — decode arbitrary calldata |

**→ EIP-712 order signing first.** Smallest addition, cleanest thing to risk-check. The general contract-call path is roadmap.

---

## Architecture: read → plan → check → approve → re-quote → sign → log

1. **Read** — Uniswap planning + Rustok `get_wallet_context` / `get_positions`.
2. **Plan** — Uniswap side builds a UniswapX order (via Trading API `/quote`), or later on-chain calldata.
3. **Check** — deterministic Layer-1 gates run on the decoded order.
4. **Approve** — human-in-the-loop above a configured threshold.
5. **Re-quote (freshness)** — a quote/order goes stale (price decay + deadline); re-quote immediately before signing and re-run the gates on the fresh order.
6. **Sign** — Rustok signs locally the order it independently reconstructed; keys never exposed to the LLM.
7. **Log** — every intent → preview → decision → outcome to SQLite.

**Trust boundary:** the planning agent (touches market data) runs with Rustok capability `read_wallet` only. A separate narrow step holds signing and acts only on a validated, approved, **freshly re-quoted** order. Compromise of the planner cannot reach signing.

---

## Invariants (load-bearing — never violate)

1. **Reconstruct-before-sign.** Never sign a digest we did not independently rebuild from the order and verify equals what we were asked to sign. Proven feasible by the spike (our `encodedOrder` re-encode == the API's `permitData` digest, byte-for-byte). Wired in code for post-fill `Cosigned` orders today; the quote-time `UnsignedV2DutchOrder` path lands in PR-2.
2. **No path to signing bypasses the gates.** A single orchestrator owns the *only* route to `SignTypedData`; every signature passes Layer-1 gates first. No side door.
3. **A signed order is a bearer instrument until its deadline.** Once signed, anyone holding it can submit it until expiry / nonce consumption. Keep deadlines short; never leak a signed order; treat the signature as a live secret until consumed or expired. To cancel a signed-but-unsubmitted order, **spend / invalidate its Permit2 nonce on-chain** (the only reliable abort-after-sign).
4. **Freshness — refuse to sign when stale.** Re-quote immediately before signing/submission; if the order is older than the staleness window, refuse and re-quote (the ready Uniswap pattern — a stale quote returns empty swap data).

---

## Net-new infrastructure (explicit)

Previously implicit; named so scope is honest:

- **mainnet-fork simulation harness** — gates run against forked on-chain state (UniswapX has no testnet resolver network).
- **corpus of adversarial ("evil") orders** — each gate ships fixtures proving it *rejects*.
- **exposure-seam design note** — resolves where gates run (TS glue vs Python-MCP) and the transport TS-glue → core. Open fork; resolved by spike + design note, not frozen here.
- **Trading API HTTP client.**
- **transport TS-glue → core** (gRPC direct vs via Python-MCP).
- **submission / orchestration** — the single owner of the signing route.

---

## Scope — committed vs roadmap

**Committed:**

- **Phase 0 — read-only (zero funds at risk).** Uniswap planning + Rustok `read_wallet`. Agent sees pools/positions, plans, explains risk; cannot sign.
- **Phase 1 — UniswapX swap (first deliverable).** EIP-712 order signing in Rustok. Gates: minOut/slippage (decoded from the order), destination-token safety (honeypot / fee-on-transfer), price cross-check vs DeFiLlama, bounded approval. Atomic, gasless, MEV-protected — smallest build, cleanest safety surface. This is the demo + smoke test.
- **Phase 2 — LP analytics (read-only).** Model fees vs **impermanent loss**, out-of-range time, per-pool hook behaviour. High value, zero execution risk.

**Roadmap (not committed):**

- General contract-call write-path in Rustok → on-chain swaps (Universal Router) and **LP execution** (PositionManager), behind the full spine, with mandatory IL warning + hook allowlist.

---

## Safety spine

**Layer 1 — deterministic hard gates (code, not LLM judgment):** order decode + `simulate`-matches-intent; minOut/slippage; destination-token safety; price cross-check (DeFiLlama coins API, **fail-closed** when no price); contract/pool allowlist; bounded (non-infinite) approval; position cap; hook allowlist (LP).

**Layer 2 — LLM explanation (Claude):** plain-language risk + IL modeling; nuance the gates can't express. The LLM **explains and warns; it cannot override a Layer-1 gate.**

**Controls:** per-tx & daily limits · approval thresholds · human-in-the-loop above X · audit log (SQLite) · kill switch.

> Limits live in this glue layer: Rustok core has **no hard spending limits by design** (`txguard` flags, does not block). This is the load-bearing safety piece of the combined system.

> The **LP hook-allowlist gate** draws on Uniswap's own `v4-security-foundations` skill, which documents how malicious hook permissions (e.g. `BEFORE_SWAP_RETURNS_DELTA`) can drain funds. (That skill targets hook *authoring* security, not agent-execution controls — the rest of the spine follows general agent-safety best-practices.)

---

## Delivery roadmap

PR-by-PR vertical, each = branch + CI + two gates. Small reversible slices: every boundary is a cheap chance to re-ask "is this still the most effective next step?" — a big-bang branch would lock us into a stale plan.

| Step | What | Why separate |
|---|---|---|
| **Spike** (done) | one live `/quote` (DUTCH_V2) — confirmed live shape + that our re-encode reproduces the API digest on a quote-time order | discovery, throwaway, kills the main unknown cheaply |
| **PR-0** | this doc into `meta/docs` + reconcile with reality | source of truth must be versioned |
| **PR-1** | read-path M0 (`get_wallet_context`/`positions` + Uniswap planning) — confirm/finish | the vertical reuses it; if ready, near no-op |
| **PR-2** | narrow full-height vertical: glue → core sign through one **orchestrator** (sole route to signing) + **one load-bearing gate (minOut)** + freshness (refuse-when-stale, re-quote-before-submit, re-gate-on-requote). E2E on dev keyring, recover-verified | this is "full-height but narrow"; closes the no-bypass invariant |
| **PR-3** | mainnet-fork harness + adversarial-order corpus | net-new foundation, reused by every gate |
| **PR-4…N** | one gate per PR, full ceremony (grill → check → 2 reviews → security-review): destination-token (honeypot/fee-on-transfer on fork) → price cross-check DeFiLlama (fail-closed) → bounded approval → allowlist | security-critical; each PR = gate + fixtures proving rejection |
| **Final** | M1 demo: green = a gate rejected a deliberately bad order | the plan's literal criterion |

---

## Principle: effectiveness over plan

The plan is **not** the final truth — Highest Effectiveness is. Never follow the plan at the expense of effectiveness. But:

- "More effective" means a **falsifiable reason** + a written change here / in STATE — never "felt faster" in passing, never silent scope drift.
- Effectiveness ≠ a quick win. Effective = **production-correct the first time** (no MVP, no temporary hacks). Both substitutions are bugs.

Plain words: the plan is a living map, not a law — but when we leave the route, we redraw the map out loud, we don't ignore it.

---

## Milestones

- **M0** — read-only demo (Phase 0).
- **M1** — UniswapX swap: gates validated on a **mainnet-fork + adversarial-order fixtures**; execution = a **tiny mainnet swap** (UniswapX has no testnet resolver network, so a real testnet fill is impossible). Green = a gate **rejected a deliberately bad order**, not just that a swap settled. The load-bearing criterion (a gate rejects a bad order) runs on the fork and is **independent of routing size**; only the secondary "tiny mainnet swap settled" smoke depends on Open fork #2.
- **M2** — LP analytics module (Phase 2).
- *(Roadmap)* — general contract-call write-path → LP execution.

---

## Open forks (resolved by spike + design note, not frozen)

1. **Where gates run + transport** (TS glue vs Python-MCP; gRPC direct vs via MCP) — design note in PR-2's lead-up.
2. **Can UniswapX routing be forced at small size?** The spike showed small swaps route CLASSIC and only larger sizes route DUTCH_V2 — so a tiny mainnet UniswapX smoke may not be achievable without forcing the route. Resolve before the mainnet smoke.
