# Networking and DNS

This document describes the networking and DNS model used by the self-hosting stack.

The design goal is to:
- minimize exposed surface area
- keep networking simple and observable
- avoid host-level networking complexity
- remain portable across hosts

---

## High-Level Networking Model

- All services run in Docker containers.
- No container uses host networking.
- No inbound ports are exposed on the NAS.
- All external traffic enters via Cloudflare Tunnel.
- Internal traffic is routed via Docker bridge networks.

The NAS acts only as a container host and does not participate in routing or proxying traffic directly.

---

## Docker Networks

Two Docker networks are used.

### `tunnel_net`

Purpose:
- Carries all externally exposed traffic.
- Connects `cloudflared`, `traefik`, and public-facing services.

Characteristics:
- Docker bridge network
- No direct host exposure
- Shared only by containers that must receive external traffic

Containers attached:
- cloudflared
- traefik
- immich
- nextcloud
- ghost
- grafana

---

### `monitoring`

Purpose:
- Isolates monitoring components from application traffic.

Characteristics:
- Docker bridge network
- No external exposure
- Used only for internal metrics scraping and visualization

Containers attached:
- prometheus
- node-exporter
- grafana

Grafana is intentionally dual-homed (`tunnel_net` + `monitoring`) to:
- expose the UI externally
- access monitoring backends internally

---

## DNS Model

### Public DNS

- All public DNS records are managed in Cloudflare.
- Hostnames point to Cloudflare Tunnel endpoints.
- No A/AAAA records point directly to the NAS IP.

Example hostnames:
- `photos.zidar.io`
- `cloud.zidar.io`
- `ghost.zidar.io`
- `grafana.zidar.io`

DNS changes are required only when adding or removing services.

---

### Internal DNS

- Docker’s embedded DNS is used for container-to-container resolution.
- Containers refer to each other by service name.
- No internal DNS server is deployed.

Example:
```text
http://traefik
http://prometheus
```

This avoids hardcoding IP addresses and simplifies redeployment.

## Routing Responsibility

| Layer            | Responsibility                         |
|------------------|----------------------------------------|
| Cloudflare       | Public DNS, tunnel ingress, edge TLS   |
| cloudflared      | Secure tunnel transport                |
| Traefik          | TLS termination, routing, middleware   |
| Docker           | Network isolation and service discovery|

Each layer has a single, clearly defined role.

---

## Non-Goals

The following are intentionally avoided:

- macvlan or ipvlan networking
- static container IPs
- host-based reverse proxying
- local DNS overrides
- exposing services on LAN IPs

---

## Failure Considerations

- Loss of Cloudflare or the tunnel results in total external inaccessibility.
- Internal container networking remains unaffected.
- Services remain accessible locally via Docker networking if required for debugging.

