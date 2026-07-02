# Exposure Seam — how the signing path is wired

> ⚠️ **Частично устарело — до разворота 2026-06-25 (custodial-эра).** Части про «ключ/подпись
> на сервере» неактуальны: продукт развёрнут на non-custodial device-signing. Текущее состояние —
> `PROJECT-STATUS.md`; незыблемое — `core/.claude/NORTH-STAR.md`. Держится как исторический контекст.

> Companion to [UNISWAP-INTEGRATION.md](./UNISWAP-INTEGRATION.md). Records the load-bearing
> design decisions for **how an agent reaches EIP-712 signing without a bypass**. Source of
> truth — changes go through a PR, not silent drift.

---

## The gap this resolves

The core gRPC `SignTypedData(domain_separator, struct_hash)` RPC exists and signs, but at
the start of PR-2 there was **no path to it from the agent**: the gateway has no
`sign_typed_data` route (`lib.rs:96-102`) and no client method (`client.rs`), the MCP has no
tool, and the dead `eip712` branch in the gateway's `sign_message` maps to core `SignMessage`
which **rejects** eip712 (`server.rs:316`). The decode / encode logic must stay TypeScript
(the official UniswapX SDK is consumed as-is — never re-implement the witness/domain). So the
seam spans **TS glue ↔ gateway ↔ core gRPC**, and it must have exactly one route to signing.

---

## Decisions (locked)

- **Q1 = (A) — orchestrator + gates live in the TS glue (`rustok-org/uniswap`).** One runtime
  where decode / encode / reconstruct / gate / sign-seam co-locate, so the gate and the signer
  see one order object and there is minimal seam for a bypass to hide. Core stays a *dumb*
  EIP-712 signer (it never decodes the order); the MCP stays a thin capability-gated proxy.

- **Q2 = (B) — transport to core signing is via the gateway (HTTP), not a direct gRPC client.**
  Add `/api/v1/wallet/sign_typed_data` (route + gateway gRPC client method) and the TS
  orchestrator calls it with Bearer auth, exactly as the MCP calls the other wallet routes.
  Rationale: the gateway is the **single policy/auth/rate-limit/audit boundary** to core, and
  core gRPC stays **internal / non-public** (only the gateway is exposed at `api.rustokwallet.com`).
  A direct gRPC client would open a *second door* to core and bypass that boundary.

- **Three F5 locks (no bypass to signing) — with realization status.** The original three were stated
  present-tense as if all locked; **two were not yet true**, so each now carries an honest status:
  1. ~~the TS orchestrator is the **sole owner** of the route/credential to the sign operation~~ —
     **RETRACTED (wrong-by-design).** The gateway gates **all** routes with one shared Bearer
     (`RUSTOK_MCP_API_KEY`, key #2), so the orchestrator is **not** the sole owner of the sign
     credential *even when fully built*. Honest replacement: **no-bypass holds for an honest,
     capability-restricted agent — NOT for a holder of inbound key #1 absent the O1 bundle** (mTLS on
     #1 + approval code-gate on the decoded order (the **off-chain order-SIGN** gate — still pending
     `request_swap` / lock #2; its **on-chain execute** analogue is code-landed dormant in Slice-1, live
     in Slice-1b — see the O1-a bullet below) + server-side SSE capability restriction; see
     [CREDENTIAL-LADDER.md](./CREDENTIAL-LADDER.md) "O1"). *(This retraction is the "O2" reconciliation.)*
  2. the MCP exposes only **`request_swap(intent)`** — never `sign(hashes)`, never a direct gRPC
     sign-client — **DESIGN-LOCKED, NOT YET LIVE.** `request_swap` does not exist in `mcp/src` yet, and
     `sign_message` is still exposed as a tool (`mcp handlers.py:285`) with capability `EXECUTE_TX`
     (`mcp capabilities.py:30`). Lands in **Slice 2c** (`request_swap`-only + the eip712-enum drop).
  3. the **dead `eip712` branch** in the gateway's `sign_message` is **removed** — **DONE (gateway):**
     shipped in Slice 2a (`core` #71, main `5dfc845`). The MCP **`eip712`-enum drop** (the tool schema
     still advertising `eip712`) is **pending Slice 2c**.

- **On-chain execute-approval code-gate (O1-a) — CODE-LANDED in Slice-1, fail-closed-DISABLED.** The core
  arbitrary-transaction handle (`PreviewTransaction` / `ExecuteTransaction`, Slice-1 — `core` #72)
  **removes** the old un-gated `PreviewSend` / `ExecuteSend` (the prompt-only execute hole), so the
  **only** on-chain execute path is `ExecuteTransaction`, which **rejects fail-closed unless it carries a
  valid user approval** bound to the `preview_id` (transitively to the immutable signed bytes), verified
  in-core **before the signer and before the one-shot preview is consumed**. In Slice-1, production ships
  **no approval issuer**, so `ExecuteTransaction` **always rejects in prod** — the accept path is compiled
  only under an off-by-default `test-approval` cargo feature, and a CI guard proves the release binary
  contains **no approval-accepting code** by asserting the literal `rustok-test-approval:` is **absent**
  from it (the durable invariant; the `approval-guard` job is the current mechanism). The issuer — the
  Telegram-`initData` approval channel (authenticated against a secret the agent lacks) that makes a valid
  approval issuable and thereby enables production execute — is **Slice-1b** (the un-gate).
  **Scope — two distinct surfaces, don't conflate:** this gate is the **on-chain execute** surface
  (generic calldata via `ExecuteTransaction`, approval bound to the decoded *transaction*). O1's "approval
  code-gate **on the decoded order**" in its original framing is the **off-chain swap-SIGN** surface
  (`request_swap` → `SignTypedData` over the decoded UniswapX order) — a **separate surface that is not
  yet built** (`request_swap` does not exist in `mcp/src`; lock #2 above). So Slice-1 is the
  execute-surface instantiation of the O1 principle "approval is a code-gate, not a prompt"; the
  order-SIGN code-gate is still pending lock #2. Preview-time simulation (eth_call revert **tri-state** +
  decode of `approve` / `transfer` / `transferFrom` / `setApprovalForAll` / `permit` / `Permit2.approve` /
  `increaseAllowance`, with explicit field presence — extended in B1 preview-hardening, `core` #73)
  surfaces WHO is authorized + whether the tx would fail, for informed approval.

- **Credential residence (F6 closure).** The orchestrator runs **server-side** and holds **key #2 (a
  service credential)** — that is **not** a personal-local key. The "personal / local / not on our
  server" requirement applies **only to key #1** (the public client→MCP credential); the wallet itself
  (keys #3/#4) is **not hosted externally**. Per-key target residence:
  [CREDENTIAL-LADDER.md](./CREDENTIAL-LADDER.md).

- **Single sign-route by construction:** the orchestrator has exactly one call site that can
  sign — an injected `Signer` seam. Enforced by test (runtime call-count + structural: the only
  `.signDigest(` caller outside the signer impl is the orchestrator).

---

## Slice split (forced by repo boundary + the hermetic form)

`uniswap`, `meta` and `core` are **separate repos** — one PR cannot span them. And the hermetic
smoke must not depend on a running core. So:

| Slice | Where | What |
|---|---|---|
| **Slice 1** ✅ | `uniswap` | hermetic orchestrator: reconstruct-before-sign → recipient-aware minOut → freshness → injected `Signer` seam (dev key, recover-verified). No core / gateway / submit. |
| **Slice 2** | `core` (gateway) + `uniswap` | live wiring: gateway `sign_typed_data` route + client + remove the dead eip712 branch + a gateway-HTTP `Signer` impl + E2E on a funded dev wallet (wallet-as-MCP). |

The `Signer` interface is the seam: Slice 1 plugs a dev-key impl; Slice 2 plugs the gateway-HTTP
impl. Same single route — only the implementation changes.

---

## Deferred to Slice 2 (named so the live slice does not omit them)

- **`recover == order.swapper`** — sign only your *own* order. Slice 1 verifies only the seam
  (`recover == dev addr`; the dev key ≠ the order's swapper). The live slice must bind the
  wallet's address to the order's `swapper`.
- **`now` must be a trusted orchestrator clock** — never from the agent or the quote, or freshness
  is bypassable. Slice 1 injects `now` for deterministic tests; the live slice binds it to a
  trusted source.
- **Destination-token gate** (`output token == intent.tokenOut`) — minOut is *amount*-safety only;
  the token check is a later gate PR.
