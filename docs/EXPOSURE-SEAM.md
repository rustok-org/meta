# Exposure Seam — how the signing path is wired

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

- **Three F5 locks (no bypass to signing):**
  1. the TS orchestrator is the **sole owner** of the route/credential to the sign operation;
  2. the MCP exposes only **`request_swap(intent)`** — never `sign(hashes)`, never a direct
     gRPC sign-client;
  3. the **dead `eip712` branch** in the gateway's `sign_message` is **removed**.

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
