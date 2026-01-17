# Cloudflare Tunnel

This document describes how Cloudflare Tunnel is used to expose services securely without opening inbound ports on the NAS.

The tunnel is treated as a critical external dependency and a deliberate security boundary.

---

## Purpose

Cloudflare Tunnel is used to:

- expose services to the internet without port forwarding
- terminate public DNS at Cloudflare
- encrypt all inbound traffic
- reduce attack surface on the NAS
- avoid maintaining firewall rules on the host or router

No service is reachable directly via the NAS IP.

---

## Architecture Position

Cloudflare Tunnel sits between the public internet and the internal reverse proxy:

```text
Internet → Cloudflare Edge → Tunnel → cloudflared → Traefik → Service
```

The tunnel endpoint runs as a container on the NAS.

## Deployment Model

- `cloudflared` runs as a Docker container.
- It is attached to the `tunnel_net` Docker network.
- It forwards traffic to Traefik using service names, not IPs.
- One tunnel is used for all services.

The tunnel is long-lived and reconnects automatically on failure.

---

## Tunnel Configuration

The tunnel is configured using:

- a Cloudflare-managed tunnel
- a credentials JSON file
- a local `config.yml` file

Sensitive files (credentials, tokens) are **never committed** to Git.

---

## Ingress Rules

Ingress rules are defined in `config.yml` and route hostnames to Traefik as an example because they have to be defined on Cloudflare's web interface.

Example (illustrative only):

```yaml
ingress:
  - hostname: photos.zidar.io
    service: http://traefik:80
  - hostname: cloud.zidar.io
    service: http://traefik:80
  - hostname: ghost.zidar.io
    service: http://traefik:80
  - hostname: grafana.zidar.io
    service: http://traefik:80
  - service: http_status:404
  ```