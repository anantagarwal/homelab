# AGENTS.md

## Project Overview
- **Purpose:** Self-hosted homelab running on a Raspberry Pi 5 (`arm64`), orchestrated with Docker Compose (v2.20+ `include:`) and Traefik v3.
- **Layout:**
  - `compose-global.yaml` — Top-level orchestrator including all service stacks on the external `homelab-network` bridge.
  - `hl` — Bash CLI wrapper around `docker compose -f compose-global.yaml`.
  - `<service>/compose.yaml` — Self-contained service definitions (`traefik/`, `glance/`, `bentopdf/`, `dockhand/`, `paperless-ngx/`, `docling/`).
  - `traefik/config/traefik.yaml` — Traefik static configuration (Cloudflare DNS-01 ACME, Docker provider).
  - `glance/config/glance.yml` — Declarative dashboard configuration.

## Commands & Verification
- **Manage stack:** `./hl up [svc]`, `./hl down [svc]`, `./hl restart [svc]`, `./hl update [svc]`
- **Inspect stack:** `./hl list`, `./hl status`, `./hl logs [svc]`, `./hl exec <svc> [cmd]`
- **Validate compose changes (required before finishing):**
  - Full stack: `docker compose -f compose-global.yaml config -q`
  - Single service: `docker compose -f <service>/compose.yaml config -q`
- **Verify `hl` syntax if edited:** `bash -n hl`

## Conventions
- **Networking:** Every `compose.yaml` must define:
  ```yaml
  networks:
    default:
      name: homelab-network
      external: true
  ```
- **Adding a new service:**
  1. Create `<service>/compose.yaml` with `restart: unless-stopped` and the external `homelab-network`.
  2. Add `<service>/compose.yaml` to `include:` in `compose-global.yaml`.
  3. Add Traefik labels if web-accessible, and optionally register it in `glance/config/glance.yml`.
- **Traefik routing:** `exposedByDefault` is `false`. Exposed services must set `traefik.enable=true`, define an HTTP router on `web` using `traefik-https-redirect`, an HTTPS router on `websecure` with `tls=true` and `tls.certresolver=cloudflare`, and `traefik.http.services.<name>.loadbalancer.server.port`.
- **Domains:** Internal-only services use `<service>.internal.anantagarwal.me`.
- **Architecture:** Target host is `linux/arm64` (Raspberry Pi 5). Prefer `arm64`/multi-arch images; document `amd64`-only caveats in comments if unavoidable.

## Boundaries
- **Never commit secrets:** Do not commit `.env` files, `traefik/certs/`, API tokens, or credentials. Provide `.env.example` templates instead.
- **Preserve stateful data:** Never run `docker compose down -v` or delete named volumes (`paperless_data`, `paperless_media`, `paperless_redis`, `dockhand_data`, `docling_cache`).
- **Keep internal services internal:** Do not expose `*.internal.anantagarwal.me` services via `cloudflare_tunnel` unless explicitly requested.
