# Overview

This repository documents a single-node self-hosting setup running on a Synology NAS.
The design prioritizes:

- reproducibility (rebuildable from scratch)
- operational simplicity
- minimal external exposure
- portability to non-Synology hardware if required

The stack is intentionally conservative: boring components, well-understood failure modes.

---

## High-level Architecture

```text
                ┌──────────────┐
                │   Internet   │
                └──────┬───────┘
                       │
                       ▼
              ┌──────────────────┐
              │  Cloudflare Edge │
              │  (DNS + Tunnel)  │
              └──────┬───────────┘
                       │  (encrypted tunnel)
                       ▼
              ┌──────────────────┐
              │   cloudflared    │
              │   (Docker)       │
              └──────┬───────────┘
                       │
                       ▼
              ┌──────────────────┐
              │     Traefik      │
              │  (reverse proxy)│
              └──────┬───────────┘
                       │
        ┌──────────────┼─────────────────┐
        ▼              ▼                 ▼
    Immich         Nextcloud           Ghost
  photos.zidar.io  cloud.zidar.io   ghost.zidar.io

                       │
                       ▼
                   Grafana
                grafana.zidar.io
```

## Traffic Flow

    1.	Public DNS is managed by Cloudflare.
    2.	All inbound traffic enters via a Cloudflare Tunnel.
    3.	No ports are exposed directly on the NAS.
    4.	The tunnel terminates at the cloudflared container.
    5.	Traffic is forwarded internally to Traefik.
    6.	Traefik handles:
        - TLS termination
        - host-based routing
        - service discovery via Docker labels
    7.	Authentication middleware (Authelia) is planned but not yet enabled.
    8.	Services are isolated on Docker networks.

## Networking Model

Docker networks in use:
- tunnel_net
Used for Cloudflare Tunnel ↔ Traefik ↔ public-facing services.

- monitoring
Isolated network for Prometheus, Node Exporter, and Grafana.

## Storage Model

Persistent data is split across two volumes:

`/volume1` — HDD RAID1
- Durable data
- Databases
- User uploads
-Configuration
- Anything that must survive cache loss

`/volume2` — NVMe RAID1
- High-IOPS paths only
- Caches
- Thumbnails
- Application state where performance matters

**Design rule:**
No service relies exclusively on `/volume2` for irreplaceable data.

## Security Posture

- Zero inbound ports exposed
- TLS enforced everywhere
- Cloudflare provides:
    - DDoS shielding
    - edge TLS
    - optional future Access policies
- Secrets are injected via environment variables or files
- No secrets are committed to Git

Authelia is planned for:
- 2FA
- SSO
- service-level access control
- Passkeys

## Failure Domains

| Component        | Impact if down                          |
|------------------|------------------------------------------|
| Cloudflare       | External access unavailable               |
| cloudflared      | External access unavailable               |
| Traefik          | All services inaccessible                 |
| Individual app   | Only that service impacted                |
| Monitoring stack | Non-critical, safe to ignore temporarily |

Monitoring is intentionally **non-critical** and excluded from recovery priority.

---

## Portability Notes

This setup does **not** depend on:
- Synology-specific networking
- Synology packages beyond Docker
- DSM-specific reverse proxy features

It can be migrated to:
- another Synology model
- a generic Linux host

with only path adjustments and Docker reinstall.
