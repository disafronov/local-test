# Local NPM + Dev CA (minimal)

Run a local Nginx Proxy Manager with a self-signed CA and server certificates for the `*.local.test` domain.

## Quick start

```bash
# Start everything
docker compose up -d
```

DNS: ensure `local.test` and `*.local.test` resolve to `127.0.0.1` ([Linux](linux.md)).

Download and install the CA certificate: <http://local.test/ca.pem>
Note: your browser may warn about an untrusted connection when downloading the CA. This is expected for the initial bootstrap with a self-signed CA.

Open: <https://local.test>

## What gets generated

- Private CA (persisted in volume `npm-ca`):
  - `/srv/ca/ca.key` — CA private key
  - `/srv/ca/ca.pem` — CA certificate (public)
  - `/srv/ca/ca.srl` — CA serial file
- Server materials (shared volume `npm-ssl`):
  - `/srv/ssl/privkey.pem` — server private key
  - `/srv/ssl/cert.pem` — server certificate signed by the CA

## Services

- `npm-ssl`: generates CA and server certificates; no runtime install steps; exits after generation
- `npm-server`: `jc21/nginx-proxy-manager` — reverse proxy with web UI
- `npm-postgresql`: `postgres` — keeps NPM data separate, so you can safely reset NPM without touching cert volumes
- `portainer`: `portainer/portainer-ce` — Docker management UI (accessible via `portainer.local.test`)
- `watchtower`: `nickfedor/watchtower` — automatic container updates (polls every 3 hours)

Customize host ports via environment variables:

| Variable            | Default     | Description                             |
|---------------------|-------------|-----------------------------------------|
| `HOST_MAIN_HTTP`    | `80`        | Host port for HTTP (container: 80)      |
| `HOST_MAIN_HTTPS`   | `443`       | Host port for HTTPS (container: 443)    |
| `HOST_MAIN_ADDR`    | `0.0.0.0`   | Host address for main port bindings     |
| `HOST_ADMIN_PORT`   | `81`        | Host port for NPM admin (container: 81) |
| `HOST_ADMIN_ADDR`   | `127.0.0.1` | Host address for NPM admin port binding |

```bash
# Use non-default ports
HOST_MAIN_HTTP=8080 HOST_MAIN_HTTPS=8443 HOST_MAIN_ADDR=127.0.0.1 HOST_ADMIN_PORT=9081 \
  docker compose up -d
```

The admin port binding address is also configurable:

```bash
HOST_ADMIN_PORT=9081 HOST_ADMIN_ADDR=0.0.0.0 \
  docker compose up -d  # expose admin on all interfaces
```

Service dependencies: services start in order with healthchecks ensuring readiness.

## Notes

- Compose config values are in `configs.npm-ssl-params` and are sourced by the `npm-ssl` service.
- CA serial is kept private under the `npm-ca` volume (`/srv/ca/ca.srl`).
- If <https://local.test> still shows a warning, restart your browser after importing the CA.
- Portainer is automatically proxied via NPM at `portainer.local.test`.
- Watchtower runs in polling mode (every 3 hours) and can be configured via environment variables (see `docker-compose.yml`).

### Extra SANs

Append extra SANs via env var when recreating `npm-ssl` (comma-separated). Accepted tokens: `DNS:name`, `IP:addr`. Invalid tokens are ignored with a warning.

```bash
# Add DNS and IP SANs for this run
SERVER_SAN_EXTRA="DNS:local.dev,IP:127.0.0.1" \
  docker compose up -d --no-deps --force-recreate npm-ssl

# Or via .env file
echo 'SERVER_SAN_EXTRA=DNS:local.dev,IP:127.0.0.1' >> .env
docker compose up -d --no-deps --force-recreate npm-ssl
```

**Note:** After regenerating certificates, restart `npm-server` to pick up the new certificates:

```bash
docker compose restart npm-server
```
