# PR-5.1 remainder: TLS reverse proxy for the full stack

> Roadmap: `core/docs/CORE-MCP-ROADMAP.md` Phase 5, PR-5.1.
> Base: `meta/docker-compose.yml` (full compose w/o TLS, meta #11/#14).

## Goal — one paragraph

Put the public entry point of the stack behind a reverse proxy with automatic
Let's Encrypt TLS, so a single `docker compose --profile prod up -d` on a server
serves MCP over HTTPS while Core and Redis stay unreachable from outside.
Local development flow (`docker compose up -d`, loopback ports, no domain)
keeps working unchanged.

## Decision required at Gate 1: Traefik vs Caddy

Roadmap PR-5.1 names **Traefik**. Engineer recommends **Caddy** instead:

| Criterion | Caddy | Traefik |
|---|---|---|
| Codex standard (`devops.md` v1.1) | **primary** for new projects | not covered |
| Let's Encrypt | built-in, zero config | resolvers + storage config |
| Config size for 1-2 hosts | ~10-line Caddyfile | static + dynamic labels |
| Proven in this product | v1 runs Caddy in production | — |
| Dynamic service discovery | no (not needed: fixed topology) | yes (unused here) |

Traefik's advantage (label-based discovery for many dynamic services) does not
apply to a fixed topology. The roadmap's actual goal — "SSL via Let's Encrypt,
Core isolated in `internal` network" — is met by either. **If Reviewer/Captain
insist on Traefik, only the proxy service block and its config file change;
the rest of this spec stands.**

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
  1. `caddy` service in `docker-compose.yml` under the `prod` compose profile
     (profiles gate whole services; base services stay profile-less and run in
     both modes):
     - image pinned to an exact version + digest (review F1:
       `caddy:2.11.4-alpine@sha256:...`), bumped via Dependabot policy;
     - ports `80:80`, `443:443`, `443:443/udp` (HTTP/3); `edge` network;
     - `restart: unless-stopped`, `security_opt: [no-new-privileges:true]`,
       `read_only: true` with rw named volumes `caddy-data` (certs) and
       `caddy-config`;
     - **deviation from service baseline:** runs as image default user (root) —
       required to bind 80/443; documented inline;
     - healthcheck via busybox `wget` against the local admin endpoint
       (`http://127.0.0.1:2019/config/`), OR an inline comment stating why
       none — decided at implementation, one of the two is mandatory;
     - `environment: RUSTOK_PUBLIC_DOMAIN: ${RUSTOK_PUBLIC_DOMAIN:?set public domain}`
       — fail-fast when the profile is up'd without a domain;
     - `depends_on: mcp: condition: service_healthy` (matches the stack's
       health-gated startup pattern from meta #14).
  2. `Caddyfile`:
     - global options block: `email {$RUSTOK_ACME_EMAIL}`;
     - site block `{$RUSTOK_PUBLIC_DOMAIN}` → `reverse_proxy mcp:3001`;
     - security headers (HSTS, nosniff, frame-deny, hide `Server`).
     - SSE: confirm against current official Caddy docs that `reverse_proxy`
       auto-disables buffering for `text/event-stream`; add explicit
       `flush_interval -1` only if docs require it (no guessing).
  3. Loopback port publishing of `gateway` (:3000) / `mcp` (:3001) **stays
     as-is** in the base file. Rationale: `127.0.0.1` binding is unreachable
     from outside the host in any mode; in `prod` only Caddy binds public
     ports. Gateway is NOT proxied — its only consumer is MCP over the
     `edge` docker network.
  4. `.env.example` (new): `RUSTOK_PUBLIC_DOMAIN`, `RUSTOK_ACME_EMAIL`,
     `RUSTOK_KEYRING_PASSWORD`, `RUSTOK_ALLOWED_CHAINS`, `RUSTOK_ALCHEMY_API_KEY`,
     `RUSTOK_MCP_API_KEY` — names + comments, no values.
  5. `README.md`: dev vs prod usage (two commands), cert storage note
     (`caddy-data` volume — survives recreation), DNS A-record prerequisite,
     Let's Encrypt staging-first advice, the v1 port-conflict note.
  6. `.gitignore`: `# Workflow artifacts` section — `.claude/reports/`,
     `.review.diff`.
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
| `docker-compose.yml` | + `caddy` service under `prod` profile |
| `Caddyfile` | new |
| `.env.example` | new |
| `README.md` | dev/prod usage, prerequisites |
| `.gitignore` | + workflow artifacts section |

## Acceptance criteria — "PR is ready when..."

1. `docker compose up -d` (no profile) behaves exactly as today: gateway on
   `127.0.0.1:3000`, MCP on `127.0.0.1:3001`, no caddy container, healthchecks
   green.
2. `RUSTOK_PUBLIC_DOMAIN=localhost RUSTOK_ACME_EMAIL=dev@localhost docker
   compose --profile prod up -d` additionally starts Caddy (localhost → Caddy
   internal CA); `curl -k https://localhost/health` returns MCP health through
   the proxy; `curl -kN https://localhost/sse` streams the first SSE event
   without buffering delay.
3. Only Caddy binds non-loopback host ports in `prod`; `core`/`redis` publish
   no ports in any mode (unchanged).
4. `docker compose config` and `docker compose --profile prod config` are valid;
   `--profile prod` without `RUSTOK_PUBLIC_DOMAIN` fails with the clear message.
5. Roadmap gate holds: full stack up < 30 s (excluding image builds).

If Docker stays unavailable on the workstation (Prerequisite 1), criteria 1–2
and 5 are verified on the target server before merge, and the Gate 2 report
says so explicitly.

## Definition of Done

- PR merged to `main` (squash), branch deleted.
- Acceptance criteria demonstrated in the Gate 2 report (or explicitly
  deferred to server verification per Prerequisite 1).
- `docs/PROJECT-STATUS.md` updated in this PR (PR-5.1 remainder → done).

## Test plan

1. `docker compose config` for both modes — valid; expected port/service sets
   (criterion 4).
2. Dev mode smoke: existing healthchecks green; `curl 127.0.0.1:3001/health` OK.
3. Prod mode locally with `localhost` domain: criterion 2 (health + SSE via
   proxy); `ss -tlnp` shows only Caddy on 80/443.
4. Negative: missing `RUSTOK_PUBLIC_DOMAIN` → fail-fast message (criterion 4).
5. Real Let's Encrypt issuance is not locally testable — verified at deploy
   time (staging endpoint first), per README.
