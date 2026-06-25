# Rustok — Project Overview (single source of truth)

> **Updated:** 2026-06-22
> **Purpose:** the master orientation doc. Where everything lives, where we came
> from, what's running now, and where we're going. Read this first.
> Detailed specs live in the per-repo docs/specs; this ties them together.

---

## 1. What Rustok is

A **self-custody, AI-native Ethereum wallet**. An LLM agent operates the wallet
through a constrained, capability-gated tool surface (MCP), with all key material
and signing isolated in a Rust core. Two consumer surfaces:

- **Agent wallet / MCP** — an agent (Claude Desktop, Cursor, OpenClaw/ClawHub,
  scripts) calls wallet tools (context, balances, positions, preview/execute,
  sign) over MCP.
- **Mobile app** (v1, React Native) — being retired in favour of the new stack.

Core capabilities today: wallet context, multi-chain balances, **DeFi positions
(Aave v3 + ERC-4626)**, ETH send preview/execute, EIP-191 signing, append-only
audit, txguard risk analysis.

---

## 2. Where everything lives

### Active org (the rewrite) — `~/Dev/projects/personal/rustok-org/`
| Repo | GitHub | Visibility | Stack | Role |
|------|--------|-----------|-------|------|
| `core` | rustok-org/core | **Private / proprietary** | Rust 2024 | Wallet logic + gRPC + HTTP Gateway |
| `mcp` | rustok-org/mcp | Public | Python 3.12 (FastAPI, uv) | MCP server (SSE + stdio), thin client to Gateway |
| `meta` | rustok-org/meta | Public | Docs + Docker Compose | Infra, compose, observability, **this doc** |
| `llm` | rustok-org/llm | Public | TBD | LLM "brain" — scaffold only, stack undecided |
| `mobile` | rustok-org/mobile | Public | React Native 0.76 + TS | Scaffold only |

### Reference / tooling
- **v1 monolith (frozen, AGPL-3.0):** `~/Dev/projects/personal/rustok`
  (github.com/temrjan/rustok). Read for architecture only — **clean-room, never
  copy-paste** (license incompatibility). Crates: `core`, `txguard`, `api`,
  `cli`, `agent-wallet`, `agent-dapps`, `agent-mcp`, `rustok-mobile-bindings`.
- **Codex standards:** `~/Dev/codex` (github.com/temrjan/codex) — the coding
  standards every repo's AGENTS.md references. `standards/`, `commands/` (slash
  commands), `agents/` (fleet reviewers). Load before writing code.

### Core crates (`core/crates/`, 12)
`types` · `crypto` · `keyring` · `provider` · `router` · `txguard` · `sign` ·
`wallet` · `audit` · `grpc` (gRPC server + binary `core-server`) ·
`gateway` (Axum HTTP + binary `gateway`) · `positions` (DeFi positions, PR-6.1).

---

## 3. Architecture

```
Agent (Claude Desktop / Cursor / OpenClaw / scripts)
        │  MCP (JSON-RPC over SSE or stdio)
        ▼
MCP server  (mcp repo, Python/FastAPI)         ── thin client, holds NO secrets
        │  HTTP/JSON  (+ Bearer)
        ▼
Gateway     (core/crates/gateway, Axum)         ── auth, rate-limit, CORS, /health, REST
        │  gRPC (tonic)
        ▼
Core        (core/crates/grpc, tonic server)    ── crypto/keyring/provider/router/txguard/sign/positions
        │                                          writes each action directly → audit (SQLite WAL, append-only)
        ▼
Ethereum RPC (Alchemy primary, public RPC fallback, eth_call)
```

- **Core ↔ Gateway:** gRPC (`wallet.proto`): WalletContext, GetBalance,
  GetPositions, PreviewSend, ExecuteSend, SignMessage (+ Health).
- **Gateway REST:** `GET /health`, `GET /api/v1/wallet/{context,balance,positions}`,
  `POST /api/v1/wallet/{preview_send,execute_send,sign_message}`. Bearer auth on
  `/api/v1/*` (key = `RUSTOK_MCP_API_KEY`); `/health` open.
- **MCP tools (6):** `get_wallet_context`, `get_balances`, `get_positions`,
  `preview_send`, `execute_send`, `sign_message` — capability-gated.
- **Audit:** Core writes each wallet action to an append-only **SQLite WAL** log
  (authoritative single source of truth). *The Phase-4 Redis Streams event bus was
  decommissioned 2026-06-17 (core #63/#64) — the `events` crate and the `redis`
  service are gone.*
- **Observability (opt-in):** OTLP push (traces+metrics+logs) → Alloy → Tempo /
  Prometheus / Loki → Grafana. Enabled via `RUSTOK_OTLP_ENDPOINT`. Off by default.

---

## 4. Where we came from

- **v1** = a single AGPL Rust monolith + RN mobile app, live at
  `api.rustokwallet.com`, plus an OpenClaw/ClawHub agent skill
  (`temrjan/rustok-wallet`, ~323 installs) that ran `agent-mcp` on a demo box.
- **Why the rewrite/org split:** AGPL is a poison pill for a commercial core, so
  the wallet logic was split into a **private/proprietary `core`** + public
  adapters (`mcp`, `meta`). Architecture upgraded: monolith → **Core (gRPC) +
  Gateway (Axum) + MCP (Python)**; `revm` simulation dropped for lightweight
  `eth_call` + Alchemy; capability-based security instead of a separate
  agent-wallet crate.
- **Deliberately dropped:** v1's hard spend-limit/budget/blocklist policy. The
  user must consciously accept that funds on the agent wallet are at risk; future
  limits will be **opt-in** user settings, not forced defaults. `txguard`
  (scam/approval/permit risk verdict) is kept.

### Roadmap progress (`core/docs/CORE-MCP-ROADMAP.md`)
Phases **0–4.5 ✅** (core hardening → gRPC service → Axum Gateway → MCP server →
event bus + audit → real wallet integration). Phase **5 ✅** observability
(logs+traces+metrics, OTLP push). **PR-6.1 ✅** DeFi positions (core #52, mcp #27,
meta #20). Optional remaining: PR-5.3 Postgres.

---

## 5. Current state (2026-06-15; Redis decommissioned 2026-06-17 — see §3 Audit)

- All core gates green; full chain **MCP → Gateway → Core** verified end-to-end
  against real mainnet (a real Aave v3 position decodes through `get_positions`).
- **Production cutover done:** `api.rustokwallet.com` now serves the **new stack**
  (was the v1 monolith). See §6.
- **"Replace ClawHub" shipped (2026-06-15):** the `rustok-wallet` all-in-one image
  is **public** on GHCR (`:latest` / `:v0.1.0`); the ClawHub skill
  `temrjan/rustok-wallet` is republished at **0.3.2** (over the old 0.2.2),
  namespace kept as `@temrjan` to preserve the install base. A standard MCP
  client gets all 6 tools out of the box (capability fix, mcp #30). The legacy
  npm package `rustok-agent-mcp` (v1, AGPL) is being deprecated with a pointer —
  the new wallet ships as a Docker image, not on npm.
  `rustok-core` image stays **private** (binary-only; the all-in-one is
  self-contained). Public image smoke-verified (anonymous pull → 6 tools).
- **Audit consumer P1 resolved & deployed** (#55 + #56) and the **gateway↔core
  startup race fixed (core #57, lazy tonic channel) & deployed** — `/health` now
  reports `core: serving`. See §8.
- Proprietary **EULA** in place (governing law: Russian Federation, core #58) —
  it gates the public image visibility.

---

## 6. Production deployment

**Server:** `185.197.195.191` (`vmi3189897`, Ubuntu 24.04). **Access:** `ssh
rustok-prod` (alias → port **9281**, user **root**, key `~/.ssh/id_ed25519`).
Port 22 closed.

**Running (Docker):**
- New stack — `/opt/rustok/deploy-v2/` (compose project `rustokv2`):
  `rustok-core`, `rustok-gateway` (test port `127.0.0.1:3001`), `rustok-mcp`
  (`127.0.0.1:3002`). Keystore in volume `rustokv2_rustok-data`
  (agent wallet `0x0C58C2a797c1c6E966321cD76F3369E13a0357ae`). Secrets in
  `/opt/rustok/deploy-v2/.env` (chmod 600). RPC = public, chains `1,8453`.
- `rustok-caddy` (caddy:2-alpine, 80/443) — TLS termination, certs in
  `deploy_caddy_data`. Routes `api.rustokwallet.com → rustok-gateway:3000` (+ JSON
  access logs) and serves `aiwell.dev` static. Live config
  `/opt/rustok/deploy/Caddyfile` (backup `.bak`; cutover source
  `/opt/rustok/deploy-v2/Caddyfile-v2`).
- **Untouched / off-limits:** `shadowbox` (Outline VPN), `watchtower`
  (scope=outline), sshd:9281, `aiwell.dev`.
- **Old monolith** `rustok-api`: **stopped, restart=no, image + `/opt/rustok/deploy`
  compose retained** for rollback (remove after ~24–48h stability).

**Image delivery (prod):** built locally (`sg docker build`) and shipped via
`docker save | ssh | docker load` — core source never touches the server (stays
private; no registry/token needed).

**Public distribution (GHCR):** `ghcr.io/rustok-org/rustok-wallet` (**public**) +
`ghcr.io/rustok-org/rustok-core` (**private**) were built locally and pushed
directly to GHCR. The `docker-publish` / `wallet-publish` Actions are `v*`-tag
gated but were blocked by the private-repo Actions-minute limit; a local
build + push avoids it ($0, source stays private). Re-enable the workflows once
minutes reset.

**Rollback (until the monolith is deleted):** restore `Caddyfile.bak` → `caddy
reload` → `docker start rustok-api`.

**Core-image rollback:** each deploy also tags the image `rustok-core:main-<sha>`
(immutable) alongside the moving `v0.1.0`. To revert a bad core deploy, retag the
prior `main-<sha>` as `v0.1.0` and `up -d core gateway`. (The startup race is
fixed — the gateway uses a lazy, reconnecting tonic channel (#57), so no manual
gateway restart is needed after a recreate.)

**Deploy/cutover model:** new stack brought up alongside, verified on a loopback
test port, then Caddy `reverse_proxy` repointed + graceful `caddy reload`
(zero-downtime), monolith stopped only after the public domain verified.

---

## 7. Security & conventions

- **Capabilities:** `read_wallet` / `preview_tx` / `execute_tx` gate the MCP tools
  (fail-closed). Gateway enforces Bearer auth on `/api/v1/*`.
- **Keys:** Argon2id (64MiB) + AES-256-GCM keystore; `zeroize`; private keys never
  cross the `types`/FFI boundary; secrets in env / `.env` (600), never in code or
  logs (RPC errors masked; Caddy redacts the Authorization header).
- **No policy/budget enforcement** by default (see §4). `txguard` kept.
- **Clean-room:** v1 (AGPL) is read for *what*, never copied. ABIs/addresses from
  public sources (e.g. Aave pool addresses from the official bgd-labs address-book).
- **Workflow (`/1rules`):** INTAKE → spec → `/check` (Gate 1) → standards
  (`/rust` / `/python`) → CODE → `/review` → report → Gate 2 → commit → push (only
  with the Captain's explicit permission). Team: Engineer (Claude) implements,
  Reviewer (Kimi) verdicts, Captain approves gates & merges.
- **Authorship:** commits/PRs are always `Temrjan <omadgo@protonmail.com>`, **no
  AI attribution** anywhere.

---

## 8. Where we're going (next)

**"Replace ClawHub" — ✅ DONE (2026-06-15):**
- De-v1-ified mcp skill/distribution (#28, #29) + onboarding (core #53);
  **capability grant fixed** for standard stdio clients (mcp #30); README de-v1
  (mcp #31); skill version 0.3.0 (mcp #34).
- Proprietary EULA, governing law RF (core #58); publish-workflow SHA pins fixed
  (core #59 + mcp #33).
- Images built locally + pushed to GHCR: `rustok-wallet` **public**, `rustok-core`
  private. Skill republished on **ClawHub at 0.3.0** (> 0.2.2). Public image
  smoke-verified (anonymous pull → all 6 tools for a standard MCP client).

**Operational — resolved:**
- ~~audit consumer P1~~ ✅ #55 + #56, deployed.
- ~~gateway↔core startup race~~ ✅ **fixed (core #57, lazy/reconnecting tonic
  channel) & deployed** — `/health` reports `core: serving`; no manual gateway
  restart on a core/gateway recreate.

**Remaining follow-ups:**
1. **Alchemy RPC:** swap the public RPCs for an Alchemy key (edit
   `/opt/rustok/deploy-v2/.env`, `docker compose up -d core`). Needs the key.
2. **Reproducibility:** commit the deploy artifacts (`docker-compose` v2 +
   `Caddyfile-v2`) into `meta` (currently only on the server).
3. **Remove the old monolith** after 24–48h of stability (cutover 2026-06-14).
4. **GH Actions Node 20→24:** the docker actions (v3.10.0/v5.7.0/v6.16.0) run on
   Node 20, deprecated 2026-06-16 — bump to Node24-compatible versions.
5. **Re-enable the GHCR publish workflows** once Actions minutes reset (or raise
   the spending limit) — the manual local push is a one-off.

**Later / optional:**
6. PR-5.3 Postgres migration. LLM brain (`llm` repo, stack decision pending).
   Mobile rebuild. Hardware signers, swap aggregation (deferred per roadmap).

---

## 9. Quick operational reference

```bash
# Server
ssh rustok-prod
docker ps                                   # rustok-core/gateway/mcp + caddy + (VPN)
docker compose -f /opt/rustok/deploy-v2/docker-compose.yml ps
docker logs rustok-core --tail 50
docker logs rustok-caddy --since 10m | grep api.rustokwallet.com   # access logs

# Public health / smoke (from anywhere)
curl https://api.rustokwallet.com/health
# get_positions needs Bearer (RUSTOK_MCP_API_KEY from the server .env)

# Build + ship a new image (core source stays private)
sg docker -c 'docker build -t rustok-core:v0.1.0 ./core'
sg docker -c 'docker save rustok-core:v0.1.0 | gzip' | ssh rustok-prod 'gunzip | docker load'
docker compose -f /opt/rustok/deploy-v2/docker-compose.yml up -d <svc>

# Local gates
# core: cargo fmt --check && cargo clippy --workspace --all-targets -- -D warnings && cargo test --workspace
# mcp:  uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest
```

**Status doc:** `meta/docs/PROJECT-STATUS.md` (phase/PR ledger). **This doc:**
the orientation map. Keep both updated at session end.
