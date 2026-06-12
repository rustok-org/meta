# PR-5.1 remainder: TLS reverse proxy for the full stack

> Roadmap: `core/docs/CORE-MCP-ROADMAP.md` Phase 5, PR-5.1.
> Base: `meta/docker-compose.yml` (full compose w/o TLS, meta #11/#14).
> Depends on: MCP inbound auth (mcp #24, merged) — see "D2" below.

## Goal — one paragraph

Put the public entry point of the stack behind a reverse proxy with automatic
Let's Encrypt TLS, so `docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d`
on a server serves MCP over HTTPS while Core and Redis stay unreachable from
outside. Local development flow (`docker compose up -d`, loopback ports, no
domain) keeps working unchanged.

## Mechanism (Gate-1 decision, supersedes the original profile approach)

The first draft gated Caddy behind a compose **profile** (`--profile prod`).
Switched to the standard Docker **base + prod-override** pattern after D2 (below)
showed profiles cannot make an env var required in prod only for a base service
without also breaking the dev flow. So:

- `docker-compose.yml` — dev base, unchanged from today (no Caddy; loopback ports).
- `docker-compose.prod.yml` — prod override: adds the `caddy` service and makes
  `RUSTOK_MCP_INBOUND_API_KEY` **required** on the `mcp` service. Run with
  `-f docker-compose.yml -f docker-compose.prod.yml`.

## D2 — inbound auth is a hard prerequisite (from the ESCALATE that froze this PR)

Exposing `mcp:3001` publicly without inbound auth would open the wallet API to
the internet. MCP inbound bearer auth shipped in mcp #24 (`RUSTOK_MCP_INBOUND_API_KEY`).
This PR must therefore:
- pass `RUSTOK_MCP_INBOUND_API_KEY` into the `mcp` service in the prod override,
  marked **required** (`:?`) so the prod stack refuses to start without it;
- prove with an acceptance test that an unauthenticated request through Caddy
  gets `401`.

## Decision (Gate 1, settled): Caddy over Traefik

Roadmap names Traefik; we use **Caddy** (Codex `devops.md` primary, built-in
Let's Encrypt, ~10-line config, already in v1 production; Traefik's dynamic
discovery is unused in a fixed topology). Approved at the first Gate 1.

## Prerequisites (Captain decisions, before/at Gate 1)

1. **Docker is not installed on the workstation** (verified 2026-06-12).
   Either Captain installs it (`curl -fsSL https://get.docker.com | sh` —
   needs sudo), or the test plan degrades to YAML review locally + full
   verification on the target server. Engineer's evidence for Gate 2 depends
   on this choice.
2. **Deploy target conflict:** v1 production Caddy already binds 80/443 on its
   server (api.rustokwallet.com). The org stack needs a dedicated host, or
   integration into the existing proxy at deploy time. Out of PR scope;
   documented in README as a prerequisite.

## PR scope

- **Title:** `feat: TLS reverse proxy for public MCP endpoint (PR-5.1)`
- **Repo:** `meta` only.
- **Included:**
  1. `docker-compose.yml` (base) — **revert to the `main` version**: the frozen
     WIP commit (`4930754`) added a profile-based `caddy` service + the two
     volumes to this file; drop all of that so base is byte-identical to `main`
     (dev unchanged, no unused volumes). Loopback publishing of `gateway`
     (:3000) / `mcp` (:3001) stays as on `main` (`127.0.0.1` unreachable
     externally).
  2. `docker-compose.prod.yml` (new) — prod override; declares the
     `caddy-data` + `caddy-config` top-level volumes here (Compose merges
     top-level `volumes` across `-f` files, so they stay out of dev):
     - `caddy` service:
       - image pinned to version + digest (review F1):
         `caddy:2.11.4-alpine@sha256:77c07d5ebfa5be9fd6c820d2094ae662c9e7eeb9bf98346b7f639900263ee2a2`,
         bumped via Dependabot;
       - ports `80:80`, `443:443`, `443:443/udp` (HTTP/3); `edge` network;
       - `restart: unless-stopped`; `cap_drop: [ALL]` + `cap_add: [NET_BIND_SERVICE]`;
         `security_opt: [no-new-privileges:true]`; `read_only: true`;
         `tmpfs: /tmp`; rw named volumes `caddy-data` (ACME certs) + `caddy-config`.
         `read_only` is consistent because the official caddy image sets
         `XDG_CONFIG_HOME=/config` and `XDG_DATA_HOME=/data` (both volumes) —
         note inline;
       - **deviation:** no `user:` override — entrypoint starts as root to bind
         80/443 (no file caps on the binary); compensated by `cap_drop: [ALL]`;
       - `environment: RUSTOK_PUBLIC_DOMAIN: ${RUSTOK_PUBLIC_DOMAIN:?...}`,
         `RUSTOK_ACME_EMAIL: ${RUSTOK_ACME_EMAIL:?...}` — fail-fast on prod up;
       - healthcheck via busybox `wget` on the local admin endpoint
         `http://127.0.0.1:2019/config/` (admin API is on by default;
         server-verified per Prerequisite 1);
       - `depends_on: mcp: condition: service_healthy`.
     - `mcp` service override — **`environment` key only, no `ports`** (Compose
       *appends* `ports` across files; re-declaring would duplicate the host
       binding). Adds
       `RUSTOK_MCP_INBOUND_API_KEY: ${RUSTOK_MCP_INBOUND_API_KEY:?set inbound key for public exposure}`
       — **required in prod only** (D2). Dev base does not reference it, so
       `docker compose up` stays open + warns (mcp behavior). Loopback
       `127.0.0.1:3001` is inherited from base and kept (unreachable externally).
  3. `Caddyfile` (new):
     - global options block: `email {$RUSTOK_ACME_EMAIL}`;
     - site block `{$RUSTOK_PUBLIC_DOMAIN}` → `reverse_proxy mcp:3001`;
     - security headers (HSTS, nosniff, frame-deny, hide `Server`).
     - SSE: Caddy flushes immediately for `Content-Type: text/event-stream`
       (verified in official `reverse_proxy` docs) — no `flush_interval` needed.
  4. `.env.example` (new): `RUSTOK_PUBLIC_DOMAIN`, `RUSTOK_ACME_EMAIL`,
     `RUSTOK_KEYRING_PASSWORD`, `RUSTOK_ALLOWED_CHAINS`, `RUSTOK_ALCHEMY_API_KEY`,
     `RUSTOK_MCP_API_KEY`, `RUSTOK_MCP_INBOUND_API_KEY` (with `openssl rand -hex 32`
     hint) — names + comments, no values.
  5. `README.md`: dev vs prod usage (the two-file prod command), cert storage
     note (`caddy-data` survives recreation), DNS A-record prerequisite,
     Let's Encrypt staging-first advice, the v1 port-conflict note, and the
     required inbound key.
  6. `.gitignore`: `# Workflow artifacts` section — `.claude/reports/`, `.review.diff`.
- **Explicitly NOT included:**
  - Observability (PR-5.2), Postgres (PR-5.3).
  - Server provisioning (UFW, fail2ban, DNS records) — README prerequisites only.
  - Rate limiting at proxy (Gateway already rate-limits; revisit with PR-5.2 metrics).
  - Any change in `core`/`mcp` repos.
  - Real production deployment (separate Captain-approved step).
  - `.gitignore` macOS-template cleanup (noticed, not touching).

## Files touched

| File | Change |
|---|---|
| `docker-compose.yml` | reverted to `main` (WIP caddy dropped) — net: no change vs `main` |
| `docker-compose.prod.yml` | new — caddy service, volumes, required inbound key on mcp |
| `Caddyfile` | new |
| `.env.example` | new |
| `README.md` | dev/prod usage, prerequisites |
| `.gitignore` | + workflow artifacts section |

## Acceptance criteria — "PR is ready when..."

1. `docker compose up -d` (base only) behaves exactly as today: gateway on
   `127.0.0.1:3000`, MCP on `127.0.0.1:3001`, no caddy container, healthchecks
   green, no inbound key required.
2. `RUSTOK_PUBLIC_DOMAIN=localhost RUSTOK_ACME_EMAIL=dev@localhost
   RUSTOK_MCP_INBOUND_API_KEY=$(openssl rand -hex 32)
   docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d` starts
   Caddy (localhost → internal CA); `curl -k https://localhost/health` returns
   MCP health through the proxy; `curl -kN https://localhost/mcp/sse` (with a
   valid bearer token) streams the first SSE event without buffering delay.
3. **D2:** through Caddy, `GET /mcp/sse` (or `POST /mcp/message`) without a
   bearer token → `401`; with `Authorization: Bearer $RUSTOK_MCP_INBOUND_API_KEY`
   → passes.
4. **D2:** the prod stack refuses to start without `RUSTOK_MCP_INBOUND_API_KEY`
   (compose `:?` error); likewise without `RUSTOK_PUBLIC_DOMAIN`.
5. Only Caddy binds non-loopback host ports in prod; `core`/`redis` publish no
   ports in any mode.
6. `docker compose config` (base) and `docker compose -f docker-compose.yml -f
   docker-compose.prod.yml config` are both valid.
7. Roadmap gate holds: full stack up < 30 s (excluding image builds).

If Docker stays unavailable on the workstation (Prerequisite 1), criteria 1–7
that need a running stack are verified on the target server before merge, and
the Gate 2 report says so explicitly. `docker compose config` lint (criterion 6)
is runnable wherever a compose binary exists.

## Definition of Done

- PR merged to `main` (squash), branch deleted.
- Acceptance criteria demonstrated in the Gate 2 report (or explicitly
  deferred to server verification per Prerequisite 1).
- `docs/PROJECT-STATUS.md` updated in this PR (PR-5.1 remainder → done).

## Test plan

1. `docker compose config` (base) and `... -f docker-compose.prod.yml config` —
   both valid; expected port/service sets (criterion 6).
2. Dev mode smoke: existing healthchecks green; `curl 127.0.0.1:3001/health` OK;
   no inbound key needed (criterion 1).
3. Prod mode locally with `localhost` domain + a generated inbound key:
   criterion 2 (health + SSE via proxy); `ss -tlnp` shows only Caddy on 80/443
   (criterion 5).
4. D2 auth through proxy: `curl -k https://localhost/mcp/sse` without token →
   401; with `-H "Authorization: Bearer $KEY"` → not 401 (criterion 3).
5. Negative / D2: prod up without `RUSTOK_MCP_INBOUND_API_KEY` → compose `:?`
   error; without `RUSTOK_PUBLIC_DOMAIN` → `:?` error (criterion 4).
6. Real Let's Encrypt issuance is not locally testable — verified at deploy
   time (staging endpoint first), per README.
