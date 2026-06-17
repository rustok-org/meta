# deploy-v2 — production deployment artifacts

Mirror of `/opt/rustok/deploy-v2/` on the prod host (`ssh rustok-prod`), committed
for reproducibility. The new stack (compose project `rustokv2`): Core + Gateway +
MCP, fronted by Caddy on `api.rustokwallet.com`.

- `docker-compose.yml` — the deployed compose (env-interpolated; **no secrets**).
- `Caddyfile-v2` — TLS reverse-proxy config for `api.rustokwallet.com`.
- `.env.example` — required env keys; copy to `.env` (chmod 600) on the host and fill.

Images are shipped privately via `docker save | ssh | docker load` (Core source
stays private). Public images live on GHCR (`rustok-wallet` public, `rustok-core`
private). See `../docs/PROJECT-OVERVIEW.md` §6.
