# PR-5.2a: Observability stack (meta) — Grafana / Prometheus / Loki / Tempo / OTel Collector

> Roadmap: `core/docs/CORE-MCP-ROADMAP.md` Phase 5, PR-5.2 (`feat/observability`).
> First of three sub-PRs (Captain-approved decomposition):
> - **5.2a (this PR, repo `meta`):** monitoring stack + log shipping + rate-limit docs.
> - 5.2b (repo `core`): Rust OTel/JSON-logs/`/metrics`/trace-propagation.
> - 5.2c (repo `mcp`): Python OTel/`/metrics`/traceparent.
> The roadmap e2e-trace-in-Grafana gate is met after all three; this PR stands up
> the backend so 5.2b/c have somewhere to export to.

## Goal — one paragraph

Stand up an observability backend as an optional compose overlay layered on the
base stack: Grafana Alloy (single agent — OTLP ingest + container-log shipping +
export), Tempo (traces), Prometheus (metrics), Loki (logs), and Grafana
(provisioned dashboards + datasources). It runs only when explicitly enabled,
stays off the public internet, and requires no changes to `core`/`mcp` (they
wire up in 5.2b/c). Also harden the public edge with documented host-level rate
limiting (PR-5.1 review #9 carry-forward).

## Mechanism — overlay file, always layered on base

`docker compose -f docker-compose.yml -f docker-compose.prod.yml -f docker-compose.obs.yml up -d`

Keeping observability in its own `docker-compose.obs.yml` means prod runs with or
without it and dev is untouched. The overlay is **not standalone**: Alloy joins
the `edge` network (defined in base) to be reachable by apps, so the obs overlay
is always layered on at least `docker-compose.yml` (`-f docker-compose.yml -f
docker-compose.obs.yml` minimum). Running obs alone fails (no `edge`).

## Design decisions (Gate 1 sign-off)

- **D1 — Grafana Alloy as the single agent (ingest + shipping + export).** Alloy
  is an OTel-Collector distribution, so one service does all of: receive OTLP
  (`:4317`/`:4318`) from apps (5.2b/c), tail container logs, and export to Tempo
  (traces), Prometheus (metrics), Loki (logs). Decouples apps from backend
  choices and avoids running a separate collector + shipper. Rejected: separate
  `otel-collector` + `promtail` (two agents for one job); apps exporting directly
  to each backend (N×M coupling).
- **D2 — Container logs via Alloy reading the Docker socket, read-only.**
  Self-contained in 5.2a, no app changes. **Security trade-off (from /check):**
  log discovery needs `/var/run/docker.sock`; a socket-holding container is
  root-equivalent on the host. Contained by: mounting the socket **`:ro`**,
  keeping Alloy on the `observability` + `edge` networks but **publishing no
  host port**, full hardening (`no-new-privileges`, resource limits), and
  documenting the residual risk. Alternative noted for later: per-service
  `logging: driver: loki` (needs the Loki Docker plugin on the host). Apps'
  structured JSON logs (5.2b/c) enrich this later; OTLP-logs from apps are
  deferred to 5.2b/c.
- **D3 — Nothing observability-related is published to the public edge.**
  Grafana publishes on `127.0.0.1:3030` (loopback) only — reached via SSH
  tunnel / VPN. Prometheus/Tempo/Loki/Collector/Alloy publish **no** host ports
  (internal network only). A public Grafana subdomain behind Caddy+auth is a
  later, separate decision.
- **D4 — Grafana auth required.** `GF_SECURITY_ADMIN_PASSWORD: ${...:?}`,
  `GF_USERS_ALLOW_SIGN_UP: "false"`, anonymous access off. Datasources +
  dashboards provisioned read-only from files.
- **D5 — Rate limiting = host-level, documented (review #9).** Caddy terminates
  TLS so Gateway's own limiter doesn't cover the edge. A Caddy rate-limit
  plugin needs a custom `xcaddy` build (breaks our pinned digest), so for now
  the primary control is host-level: a ready-to-use `nftables` per-IP example +
  a `fail2ban` jail on the Caddy access log, both in the README. Caddy-plugin
  rate limiting is noted as a future option.
- **D6 — Dedicated `observability` network** for the obs services; Alloy also
  joins `edge` so apps (on `edge`) can reach it at `alloy:4317`. Backends
  (Tempo/Prometheus/Loki) are on `observability` only — reachable from Alloy and
  Grafana, not from the app network.

## PR scope (repo `meta` only)

- **Included:**
  1. `docker-compose.obs.yml` (new): services `alloy`, `tempo`, `prometheus`,
     `loki`, `grafana` (5 — Alloy replaces collector+shipper, F3). All: pinned
     image+digest (resolved at implementation, F5; Dependabot bumps),
     `restart: unless-stopped`, `no-new-privileges`, non-root where the image
     allows, healthchecks, named volumes for state (prometheus/tempo/loki/
     grafana data). Only `grafana` publishes a host port (`127.0.0.1:3030:3000`).
     `observability` network; Alloy also on `edge` (D6). Resource limits +
     log rotation on each (consistency with Caddy). Alloy mounts
     `/var/run/docker.sock:/var/run/docker.sock:ro` (D2 trade-off).
  2. `observability/` config dir (new):
     - `alloy/config.alloy` — OTLP receivers (gRPC 4317 / HTTP 4318) → export to
       Tempo (traces) + Prometheus (metrics); discover Docker containers, tail
       logs → Loki, label by compose service.
     - `prometheus.yml` — scrape self + Alloy; app targets (`gateway`, `mcp`)
       defined but commented/optional until 5.2b/c expose `/metrics`.
     - `tempo.yaml`, `loki.yaml` — minimal single-binary local-storage config.
     - `grafana/provisioning/datasources/*.yaml` — Prometheus, Loki, Tempo
       (with trace↔log correlation).
     - `grafana/provisioning/dashboards/*` — a starter "Rustok overview"
       dashboard (stack health; panels for app metrics render once 5.2b/c land).
  3. `.env.example`: add `GF_SECURITY_ADMIN_PASSWORD` (with note + gen hint).
  4. `README.md`: how to start with the obs overlay (always layered on base, F2),
     access Grafana via SSH tunnel, the host-level rate-limit setup (nftables +
     fail2ban examples).
  5. `.github/workflows/ci.yml`: extend to validate the three-file compose config
     (passing a dummy `GF_SECURITY_ADMIN_PASSWORD=ci`, F4).
- **Explicitly NOT included:**
  - App instrumentation in `core`/`mcp` (5.2b/c) — no `/metrics`, no OTLP export
    from apps yet; Prometheus app-scrape targets stay commented until then.
  - Postgres (PR-5.3).
  - Public Grafana exposure / SSO.
  - Alerting rules / Alertmanager (future; note as follow-up).
  - Caddy-plugin rate limiting (custom build) — host-level only this PR.

## Files touched

| File | Change |
|---|---|
| `docker-compose.obs.yml` | new — 5 obs services (alloy, tempo, prometheus, loki, grafana) |
| `observability/**` | new — alloy/prometheus/tempo/loki/grafana configs |
| `.env.example` | + `GF_SECURITY_ADMIN_PASSWORD` |
| `README.md` | obs usage (layered on base), Grafana access, host rate-limit examples |
| `.github/workflows/ci.yml` | validate 3-file compose config (dummy GF password) |

## Acceptance criteria — "PR is ready when..."

1. Dev (`docker compose up`) and prod (two-file) are unchanged — obs services
   appear only with the third `-f docker-compose.obs.yml`.
2. `docker compose -f .. -f .. -f docker-compose.obs.yml config` is valid; obs
   stack fails fast without `GF_SECURITY_ADMIN_PASSWORD`.
3. Obs overlay starts (layered on base); all five services reach `healthy`.
   Grafana on `127.0.0.1:3030`, shows the three datasources green
   (Prometheus/Loki/Tempo).
4. Container logs are visible in Grafana → Loki (Alloy shipping works), labelled
   by service.
5. No observability service binds a non-loopback host port (`ss -tlnp`:
   only Grafana on `127.0.0.1:3030`; nothing else new).
6. Prometheus targets page: self + Alloy UP (app targets intentionally absent
   until 5.2b/c).
7. README documents the host-level rate-limit (nftables + fail2ban) for the
   public edge (review #9).
8. CI validates the three-file compose config.

## Definition of Done

- PR merged to `main` (squash), branch deleted.
- Acceptance criteria demonstrated in the Gate 2 report (Docker available).
- `docs/PROJECT-STATUS.md` updated (PR-5.2a done; 5.2b/c next).

## Test plan (Docker 29.5.3 local, via `sg docker -c`)

1. `docker compose config` for all three overlay combinations — valid; obs only
   in the three-file set.
2. Fail-fast: obs config without `GF_SECURITY_ADMIN_PASSWORD` → `:?` error.
3. Bring up the obs overlay layered on base (`-f docker-compose.yml -f
   docker-compose.obs.yml`, minimal app stub if needed); assert all five obs
   services healthy.
4. Grafana `127.0.0.1:3030`: datasources green; a test log line from a container
   appears in Loki; Tempo/Prometheus reachable (no traces/app-metrics yet — by
   design).
5. `ss -tlnp` confirms only `127.0.0.1:3030` newly bound.
6. Image digests resolved/pulled; `docker compose pull` succeeds.

## Open questions for Reviewer

- Tempo/Loki: single-binary local-storage is fine for now (no S3/object store)?
- Starter dashboard scope: stack-health only, or also pre-build app panels that
  stay empty until 5.2b/c?
