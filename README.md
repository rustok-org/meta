# Rustok Meta

Shared infrastructure and documentation.

## Structure
- `docker-compose.yml` — base stack (MCP → Gateway → Core), loopback-only
- `docker-compose.prod.yml` — prod overlay: Caddy TLS termination + required inbound auth
- `Caddyfile` — public TLS entry point (reverse proxy to MCP)
- `docs/` — architecture specs and plans

## Running the stack

### Development (loopback, no TLS)

```bash
RUSTOK_KEYRING_PASSWORD=... docker compose up -d
```

Gateway on `127.0.0.1:3000`, MCP on `127.0.0.1:3001`. Core publishes no
ports. MCP inbound auth is disabled (it logs a warning) — fine for loopback.

### Production (TLS via Caddy)

```bash
RUSTOK_KEYRING_PASSWORD=... \
RUSTOK_PUBLIC_DOMAIN=wallet.example.com \
RUSTOK_ACME_EMAIL=ops@example.com \
RUSTOK_MCP_API_KEY=... \
RUSTOK_MCP_INBOUND_API_KEY=$(openssl rand -hex 32) \
  docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Caddy is the only service binding public ports (80/443, plus 443/udp for HTTP/3)
and obtains a Let's Encrypt certificate automatically. The prod overlay refuses
to start if `RUSTOK_PUBLIC_DOMAIN`, `RUSTOK_ACME_EMAIL`, `RUSTOK_MCP_API_KEY`
(MCP↔Gateway), or `RUSTOK_MCP_INBOUND_API_KEY` (client→MCP) is unset.

Clients reach MCP over HTTPS and must send `Authorization: Bearer
$RUSTOK_MCP_INBOUND_API_KEY`. See `.env.example` for all variables.

#### Prerequisites & notes

- **DNS:** an `A` record for `RUSTOK_PUBLIC_DOMAIN` must point at the host
  *before* first start, or the ACME HTTP-01 challenge fails.
- **Let's Encrypt staging first:** to avoid rate limits while testing, point
  Caddy at the staging CA before going live (add
  `acme_ca https://acme-staging-v02.api.letsencrypt.org/directory` to the global
  options block), then switch to production.
- **Certificate storage:** issued certs live in the `caddy-data` volume —
  it survives `docker compose down`/recreation. Deleting it forces re-issuance
  (mind LE rate limits).
- **Local TLS test:** set `RUSTOK_PUBLIC_DOMAIN=localhost`; Caddy serves an
  internal-CA certificate (use `curl -k`).
- **Port conflict:** the v1 monolith's Caddy already binds 80/443 on
  `api.rustokwallet.com`. Deploy the org stack on a dedicated host or integrate
  with the existing proxy.
- **Rate limiting (required before go-live):** Caddy terminates TLS and proxies
  straight to MCP, so the Gateway's own rate limiting does NOT protect the public
  edge. The control is host-level (a Caddy rate-limit plugin needs a custom
  `xcaddy` build, which breaks the pinned image — deferred). Set this up before
  exposing the endpoint; see **Edge rate limiting** below. Do not skip it.

### Deploy checklist (before first public start)

1. DNS `A` record for `RUSTOK_PUBLIC_DOMAIN` → host, propagated.
2. Host firewall: allow 80/443 (tcp+udp); everything else default-deny.
3. Host-level rate limiting / fail2ban in place (see above).
4. `.env` populated (`chmod 600`); `RUSTOK_MCP_INBOUND_API_KEY` from
   `openssl rand -hex 32`; `RUSTOK_MCP_API_KEY` set.
5. First bring-up with Let's Encrypt **staging** CA; confirm a cert is issued
   and `https://$DOMAIN/health` returns 200, `GET /mcp/sse` without a token → 401.
6. Switch to the production CA; redeploy; re-verify.
7. Smoke the full chain with a real client: `Authorization: Bearer
   $RUSTOK_MCP_INBOUND_API_KEY` against `/mcp/sse` streams events.

## Observability (optional overlay)

Adds Grafana Alloy (OTLP ingest + log shipping) → Tempo / Prometheus / Loki,
surfaced in Grafana. **Always layered on the base file** (it needs the `edge`
network) — `-f docker-compose.yml` is the minimum, or add it as a third file on
top of prod:

```bash
GF_SECURITY_ADMIN_PASSWORD=$(openssl rand -hex 16) \
RUSTOK_KEYRING_PASSWORD=... \
  docker compose -f docker-compose.yml -f docker-compose.obs.yml up -d
# production:
GF_SECURITY_ADMIN_PASSWORD=... RUSTOK_KEYRING_PASSWORD=... RUSTOK_PUBLIC_DOMAIN=... \
RUSTOK_ACME_EMAIL=... RUSTOK_MCP_API_KEY=... RUSTOK_MCP_INBOUND_API_KEY=... \
  docker compose -f docker-compose.yml -f docker-compose.prod.yml -f docker-compose.obs.yml up -d
```

- Nothing here is public: only Grafana binds `127.0.0.1:3030`. Reach it from a
  workstation over an **SSH tunnel**:
  `ssh -N -L 3030:127.0.0.1:3030 user@host`, then open `http://localhost:3030`
  (login `admin` / `$GF_SECURITY_ADMIN_PASSWORD`).
- **Distributed traces (PR-5.2b):** with this overlay up, core/gateway export
  OTLP/HTTP spans to `alloy:4318` over a dedicated `telemetry` network (the
  overlay sets `RUSTOK_OTLP_ENDPOINT` for them). A REST call to the gateway
  appears in Tempo as one trace spanning gateway → core; their JSON logs carry
  the matching `trace_id` (jump Tempo↔Loki via the provisioned datasource link).
  Without the overlay (plain `docker compose up`) the services emit JSON logs
  only — no trace export. `RUST_LOG` tunes log verbosity (default `info`).
- **MCP traces (PR-5.2c):** MCP is also instrumented and propagates `traceparent`
  over HTTP, and the Gateway continues an inbound trace — so an MCP tool call shows
  in Tempo as **one** trace `rustok-mcp` → `rustok-gateway` → `rustok-core`, with
  matching `trace_id` in the JSON logs.
- **App metrics (PR-5.2d):** gateway / core / mcp push OTLP **metrics** to Alloy on
  the same endpoint as traces; Alloy remote-writes them to Prometheus (no `/metrics`
  scrape). Query e.g. `http_server_request_duration_*` / `rpc_server_duration_*` in
  Grafana. This completes the Phase 5 observability layer (logs + traces + metrics).
- **Docker socket:** Alloy mounts `/var/run/docker.sock` to discover container
  logs. A `:ro` mount does not restrict the Docker API — a compromised Alloy is
  root-equivalent on the host. It is contained by network isolation, no public
  port, resource limits, and `no-new-privileges`. Hardening follow-up: front it
  with a `docker-socket-proxy` (whitelist containers/logs only).

### Retention & disk usage

Local single-binary storage is bounded so it cannot fill the disk:
Tempo `block_retention` 7d, Loki `retention_period` 7d (compactor on),
Prometheus `--storage.tsdb.retention.time=15d --retention.size=5GB`. Multi-node
later → move Tempo/Loki to object storage (S3/MinIO).

Check usage and purge if needed:

```bash
docker system df -v | grep -E 'tempo-data|loki-data|prometheus-data'   # volume sizes
# manual purge (stops obs, drops its volumes — telemetry only, not app data):
docker compose -f docker-compose.yml -f docker-compose.obs.yml down
docker volume rm meta_tempo-data meta_loki-data meta_prometheus-data
```

### Edge rate limiting (set up before public exposure, review #9)

Caddy proxies straight to MCP, so apply per-IP limits at the host.

`nftables` — cap new connections per source IP on 443:

```
table inet rustok {
    chain input {
        type filter hook input priority 0; policy accept;
        tcp dport 443 ct state new meter conns { ip saddr limit rate 20/second burst 40 packets } accept
        tcp dport 443 ct state new drop
    }
}
```

`fail2ban` — ban IPs hammering the Caddy access log (jail sketch):

```
[caddy-mcp]
enabled  = true
port     = https
filter   = caddy-mcp        # matches repeated 401/429 in the Caddy JSON access log
logpath  = /var/log/caddy/access.log
maxretry = 20
findtime = 60
bantime  = 3600
```
