# Rustok Meta

Shared infrastructure and documentation.

## Structure
- `docker-compose.yml` — base stack (MCP → Gateway → Core + Redis), loopback-only
- `docker-compose.prod.yml` — prod overlay: Caddy TLS termination + required inbound auth
- `Caddyfile` — public TLS entry point (reverse proxy to MCP)
- `docs/` — architecture specs and plans

## Running the stack

### Development (loopback, no TLS)

```bash
RUSTOK_KEYRING_PASSWORD=... docker compose up -d
```

Gateway on `127.0.0.1:3000`, MCP on `127.0.0.1:3001`. Core and Redis publish no
ports. MCP inbound auth is disabled (it logs a warning) — fine for loopback.

### Production (TLS via Caddy)

```bash
RUSTOK_KEYRING_PASSWORD=... \
RUSTOK_PUBLIC_DOMAIN=wallet.example.com \
RUSTOK_ACME_EMAIL=ops@example.com \
RUSTOK_MCP_INBOUND_API_KEY=$(openssl rand -hex 32) \
  docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Caddy is the only service binding public ports (80/443, plus 443/udp for HTTP/3)
and obtains a Let's Encrypt certificate automatically. The prod overlay refuses
to start if `RUSTOK_PUBLIC_DOMAIN`, `RUSTOK_ACME_EMAIL`, or
`RUSTOK_MCP_INBOUND_API_KEY` is unset.

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
