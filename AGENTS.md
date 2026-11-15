# AGENTS.md

## Project Overview
- **Purpose:** Self-hosted homelab running on a Raspberry Pi 5 (`arm64`), orchestrated with Docker Compose (v2.20+ `include:`) and Traefik v3.
- **Layout:**
  - `pi/compose.yaml` — Raspberry Pi top-level orchestrator (`name: homelab`) including Pi stacks on `homelab-network`.
  - `nuc/media/compose.yaml` — Intel NUC `media` CT (`amd64` Proxmox CT `202`) orchestrator.
  - `hl` — Bash CLI wrapper (`./hl pi <cmd>` or `./hl <ct> <cmd>`, e.g. `./hl media <cmd>`).
  - `pi/<service>/compose.yaml` — Pi service definitions (`traefik/`, `glance/`, `bentopdf/`, `dockhand/`).
  - `nuc/<ct>/<service>/compose.yaml` — NUC per-CT service definitions (e.g. `nuc/media/jellyfin/`, `nuc/media/paperless-ngx/`).
  - `pi/traefik/config/traefik.yaml` — Traefik static configuration (Cloudflare DNS-01 ACME, Docker + File providers).
  - `pi/traefik/config/dynamic/nuc.yaml` — Dynamic Traefik routes proxying NUC services (`192.168.1.202`).
  - `pi/glance/config/glance.yml` — Declarative dashboard configuration.

## Commands & Verification
- **Manage stack:** `./hl [pi|<ct>] up [svc]`, `./hl [pi|<ct>] down [svc]`, `./hl [pi|<ct>] restart [svc]`, `./hl [pi|<ct>] update [svc]`
- **Inspect stack:** `./hl [pi|<ct>] list`, `./hl [pi|<ct>] status`, `./hl [pi|<ct>] logs [svc]`, `./hl [pi|<ct>] exec <svc> [cmd]`
- **Validate compose changes (required before finishing):**
  - Pi stack: `docker compose -f pi/compose.yaml config -q`
  - NUC CT stack: `docker compose -f nuc/<ct>/compose.yaml config -q`
  - Single service: `docker compose -f <host-or-ct>/<service>/compose.yaml config -q`
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
