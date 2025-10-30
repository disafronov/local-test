# Local NPM + Dev CA (minimal)

Run a local Nginx Proxy Manager with a self-signed CA and server certificates for the `*.test` domain.

## Quick start

```bash
# Start everything
docker compose up -d
```

DNS: ensure the `*.test` zone resolves to `127.0.0.1` (out of scope here).

Download and install the CA certificate: http://ca.test/

Open: https://test

## What gets generated

- Private CA (persisted in volume `ca`):
  - `/srv/ca/ca.key` — CA private key
  - `/srv/ca/ca.srl` — CA serial file
- Server materials (shared volume `ssl`):
  - `/srv/ssl/ca.pem` — CA certificate (public)
  - `/srv/ssl/server_key.pem` — server private key
  - `/srv/ssl/server_certificate.pem` — server certificate signed by the CA

## Services

- `ssl`: generates CA and server certificates; no runtime install steps; exits after generation
- `app`: `jc21/nginx-proxy-manager`
- `db`: `postgres` — keeps NPM data separate, so you can safely reset NPM without touching cert volumes

Ports: 80 (HTTP), 443 (HTTPS), 81 (NPM admin — bound to 127.1.0.1 only)

## Notes

- Compose config values are in `configs.ssl_params` and are sourced by the `ssl` service.
- CA serial is kept private under the `ca` volume (`/srv/ca/ca.srl`).
- If https://test still shows a warning, restart your browser after importing the CA.
