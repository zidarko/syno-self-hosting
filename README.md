# Synology Self-Hosting Stack (zidar.io)

This repo documents and version-controls my self-hosted services running on a Synology NAS.

Goal: reproducible rebuild + low-drama operations + portability in case of NAS change

No Synology-specific features are required beyond Docker and mounted volumes.

## High-level
Traffic flow:
Cloudflare Tunnel → Traefik → (Authelia, not setup yet) → Service

Key hostnames:
- photos.zidar.io (Immich)
- cloud.zidar.io (Nextcloud)
- ghost.zidar.io (Ghost)
- grafana.zidar.io (Grafana)

# Assumptions

- Single-node Synology NAS
- No ports exposed directly to the internet
- All external access via Cloudflare Tunnel
- TLS terminated at Traefik
- Docker Compose managed via Synology Container Manager
- Portainer required for operation

## Folder layout on NAS
All persistent service data lives under:
- /volume1/docker/
and
- /volume2/nvme-cache/

Example:
- /volume1/docker/immich/
- /volume1/docker/nextcloud/
- /volume1/docker/traefik/
- /volume2/nvme-cache/immich/
- /volume2/nvme-cache/nextcloud/
- /volume2/nvme-cache/traefik/

Design rationale:
- `/volume1` (HDD RAID1) stores all durable data and anything that must survive cache loss.
- `/volume2` (NVMe RAID1) is used for high-IOPS paths only (DBs, caches, thumbnails).
- No service relies exclusively on `/volume2` for irreplaceable data.

## Deploy order (from scratch)
1. Create docker networks: `tunnel_net`, `monitoing` (see docs/20-networking-dns.md)
2. Deploy `cloudflared` (stacks/cloudflared)
3. Deploy `traefik` (stacks/traefik)
4. Deploy apps: Immich / Nextcloud / Ghost
5. Deploy monitoring: Prometheus / Node exporter / Grafana

Monitoring is non-critical and can be skipped during disaster recovery.

## Secrets
Never commit `.env` files, tunnel credential JSON, or Authelia secrets.
Use `env.example` templates and add your secrets locally.

## Docs
Start here: [docs/00-overview.md](./docs/00-overview.md)
Ops playbook: docs/90-ops-playbook.md

## Repo Structure

    ```text
    synology-selfhosting/
    ├─ README.md
    ├─ docs/
    │  ├─ 00-overview.md
    │  ├─ 10-prereqs-synology.md
    │  ├─ 20-networking-dns.md
    │  ├─ 30-cloudflare-tunnel.md
    │  ├─ 40-traefik.md
    │  ├─ 50-auth-authelia.md
    │  ├─ 60-immich.md
    │  ├─ 70-nextcloud.md
    │  ├─ 80-grafana-prometheus.md
    │  ├─ 90-ops-playbook.md
    │  └─ 99-incident-notes.md
    ├─ stacks/
    │  ├─ traefik/
    │  │  ├─ docker-compose.yml
    │  │  └─ config/            # static config + dynamic
    │  ├─ cloudflared/
    │  │  ├─ docker-compose.yml
    │  │  └─ cloudflared/       # tunnel config.yml (sanitize secrets)
    │  ├─ authelia/
    │  │  ├─ docker-compose.yml
    │  │  └─ config/
    │  ├─ immich/
    │  │  ├─ docker-compose.yml
    │  │  └─ env.example
    │  ├─ nextcloud/
    │  │  ├─ docker-compose.yml
    │  │  └─ notes.md
    │  └─ monitoring/
    │     ├─ docker-compose.yml
    │     └─ dashboards/         # exported json dashboards
    ├─ .env.example              # top-level env template (optional)
    └─ scripts/
    ├─ backup-config.sh
    ├─ restore-notes.md
    └─ sanitize.sh
    ```