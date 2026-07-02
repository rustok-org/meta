# Credential Key-Ladder — where each secret lives

> ⚠️ **Частично устарело — до разворота 2026-06-25 (custodial-эра).** Части про «ключ/подпись
> на сервере» неактуальны: продукт развёрнут на non-custodial device-signing. Текущее состояние —
> `PROJECT-STATUS.md`; незыблемое — `core/.claude/NORTH-STAR.md`. Держится как исторический контекст.

> Companion to [EXPOSURE-SEAM.md](./EXPOSURE-SEAM.md). Records the **target residence** of each of
> Rustok's credentials — "where the secret lives, how long, how scoped" — on proven, standard patterns.
> Source of truth: changes go through a PR, not silent drift. Scope: **N=1** (one operator, one wallet;
> verified `core grpc/main.rs:23` one-shot onboarding, `:89-105` refuses to overwrite an existing
> keystore) — **no per-user registry, no provisioning system** (that would be invented multi-tenancy).

---

## Two axes — do not conflate

| Axis | Question | Covered by |
|---|---|---|
| **A — residence** | *Where does the secret live, how long, how scoped?* | **this ladder** |
| **B — no-bypass to signing** | *Can an authorized-but-manipulated agent misuse the sign path?* | the capability model + code-approval — see [EXPOSURE-SEAM.md](./EXPOSURE-SEAM.md) |

This doc is **axis A only**. The "GitHub ladder" (below) solves *credential theft / leakage* — the same
problem GitHub solved over 15 years. It does **not** solve axis B (agent manipulation): a leaked secret
is a thief, not a manipulated-but-authorized bot. Axis B stays with the capability model + approval
code-gate; this doc names the boundary but does not re-spec capabilities. **The wallet itself is not
hosted externally** — keys #3/#4 are the safe (see "Off-ladder" below).

---

## The ladder (worst → best)

Each rung is a *form of residence*, borrowed from a proven GitHub/SSH primitive:

| Rung | Form | Proven analog |
|---|---|---|
| **R0** | standing shared secret — static, long-lived, shared, server-stored | (the thing we are leaving) |
| **R1** | scoped + expiring stored secret — least-privilege + TTL | GitHub fine-grained PAT |
| **R2** | short-lived, minted on demand from a parent key (~1h) | GitHub App installation token |
| **R3** | secretless federation — exchange a verifiable identity (JWT) for an ephemeral scoped token | GitHub Actions OIDC |
| **R4** | asymmetric, private half **local on the operator device** + per-action presence; server holds **only a public verifier** | SSH / FIDO2 / mTLS |

> **Authz controls are not rungs.** Approval-code-gate and server-side capability restriction are axis-B
> controls, not residence rungs — they never appear in this ladder.

---

## Per-key target residence

| Key | Counterparty / role | Current | Target | Status |
|---|---|---|---|---|
| **#1** `RUSTOK_MCP_INBOUND_API_KEY` | public client → MCP | R0 | **R4** — mTLS client cert at the Caddy edge | **VERIFIED** |
| **#2** `RUSTOK_MCP_API_KEY` | internal MCP → gateway (private net) | R0 | **R2** — short-lived minted token | **UNVERIFIED** + task |
| **#3** `RUSTOK_KEYRING_PASSWORD` | unlocks the core keystore | server-side | — | **OFF-LADDER** (the safe itself) |
| **#4** wallet private key (`keystore.json`) | signs | server-side | — | **OFF-LADDER** (the safe itself) |
| **#5** provider keys (Alchemy/Infura) | gateway/core → RPC | R0 | **R3** if supported, else **R1** | **UNVERIFIED** + task |
| **#6** Trading-API `x-api-key` | service | R0 | **R2/R3** if supported, else **R1** | **UNVERIFIED** + task |

**Status is 3-valued, honesty-gated:** a rung is **VERIFIED** only with an inline source; otherwise
**UNVERIFIED + a named task**; the safe itself is **OFF-LADDER** (not "unverified").

### #1 — VERIFIED (R4, the Captain's hard-constraint key)
Caddy supports mutual-TLS client auth in exactly the needed form:
`tls { client_auth { mode require_and_verify; trust_pool file <CA.pem> } }`. `trust_pool file` holds
**only the CA (public)** on the server; the operator holds the **client certificate + private key
locally**. So *"the server holds no signing secret for #1"* is achievable **by construction** — the
private half never reaches our server. Source:
<https://caddyserver.com/docs/caddyfile/directives/tls>.

### #3 / #4 — OFF-LADDER, not UNVERIFIED
These are the **safe itself**. Moving them off-server (hardware-backed, never-exported, FIDO2/enclave
style) is a separate **self-custody epic**, not a rung of this ladder. Marking them "UNVERIFIED" would
falsely imply there is something to climb *here*.

### Sequencing
#1 (R4) is the **load-bearing first move** — the axis-A half of the O1 bundle. **#2 / #5 / #6 are
post-O1, low priority** — not parallel work; they advance only after O1 lands (~M1). Their target rung
depends on counterparty support, left UNVERIFIED until implementation time (see Tasks).

---

## Load-bearing consequence of #1 → R4 (locked)

mTLS on #1 requires the legitimate client to **present a client certificate**. The Caddy side is
trivial; the binding constraint is **client-side**: the cert is held by an **operator-controlled,
operator-local client** (a local MCP client / the operator's device). This **excludes an arbitrary
hosted-agent connector** (e.g. a cloud connector that cannot present the client cert) from the #1 path.
Consistent under N=1 — but stated here so it does not surface as a surprise at O1 implementation
("added mTLS → the agent connector broke").

**Forward-note (O1-time):** the #1 cutover R0 → R4 must be **dual-run** — accept both the old Bearer
and the new mTLS during migration, then drop the old — to avoid a flag-day break of the live wallet.

---

## O1 — the trigger-bundle (not a rung)

O1 is the bundle that must be in place **before the sign path touches a real-value wallet** (~M1
trigger). It spans **both axes** and **none of its parts suffices alone**:

- **axis A:** #1 → **R4** (mTLS at the edge) — *VERIFIED here.*
- **axis B:** (a) **approval as a code-gate** on the *decoded* order (the FIDO2 per-action-presence
  analog — proof a human consented to *this* swap, not a session grant); (b) **server-side capability
  restriction** on the public SSE surface (it ignores self-declared caps).

O1 is a **trigger**, not a ladder rung; the axis-B parts are tracked in
[EXPOSURE-SEAM.md](./EXPOSURE-SEAM.md), not here.

---

## Tasks (counterparty verification — implementation-time, post-O1)

- **#2** — confirm the gateway auth layer can mint/validate short-lived tokens (R2) instead of the static
  shared Bearer; design the dual-run cutover. *Internal hop, private net — lower urgency than #1.*
- **#5** — check whether the RPC provider (Alchemy/Infura) supports OIDC/STS-style federation (→ R3);
  else scoped + expiring keys (→ R1).
- **#6** — check whether the Trading-API supports minted/federated creds (→ R2/R3); else R1.

> No claim above asserts a vendor "supports X" without a source — that is the honesty gate. These stay
> UNVERIFIED until checked at implementation time.

---

## References

- Caddy `tls` directive — `client_auth { mode require_and_verify; trust_pool file }` (server holds the
  CA only): <https://caddyserver.com/docs/caddyfile/directives/tls> — **backs #1 → R4 (VERIFIED).**
- GitHub Actions OIDC — secretless, short-lived per-job tokens:
  <https://docs.github.com/en/actions/concepts/security/openid-connect> — backs **R3**.
- GitHub App installation tokens — short-lived, org-scoped, minted on demand:
  <https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/deciding-when-to-build-a-github-app> — backs **R2**.
- Yubico — securing git with SSH + FIDO2 (private key never leaves the token; touch-to-sign):
  <https://developers.yubico.com/SSH/Securing_git_with_SSH_and_FIDO2.html> — backs **R4** + the
  approval-presence analog.
