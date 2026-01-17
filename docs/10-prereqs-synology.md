# Synology Prerequisites

This document describes the host-level assumptions and prerequisites for running the self-hosting stack on a Synology NAS.

The goal is to make the host *predictable* and *boring* before any containers are deployed.

---

## Supported Hardware

- Synology NAS capable of running Docker / Container Manager
- x86_64 CPU architecture (Intel or AMD)
- Minimum 8 GB RAM (16 GB recommended)
- Storage:
  - 2 HDDs for RAID1 durable storage
  - 2 NVMe drives for RAID1 high-IOPS storage (optional but recommended)

This setup assumes a **single-node** NAS.  
No clustering or Synology HA is used.

---

## DSM Version

- DSM 7.x
- All system updates applied

No reliance on deprecated DSM 6 features.

---

## Required Packages

The following Synology packages must be installed and enabled:

- **Container Manager**  
  Used to run all services via Docker Compose.

No other Synology-specific services are required.

- **Portainer**
  Used to manage container via internet browser of choice.

- **Watchtower**
  Used to automatically check and update containers.

---

## Volume Layout

This setup assumes two mounted volumes.

### `/volume1` — HDD RAID1

- Created using Synology Storage Manager
- Backed by mdadm RAID1
- Used for:
  - persistent application data
  - databases
  - configuration
  - backups

### `/volume2` — NVMe RAID1

- Created using Synology Storage Manager
- Backed by mdadm RAID1
- Used for:
  - caches
  - thumbnails
  - performance-critical paths

Using NVMe drives as block devices requires a small Synology modification, as NVMe drives are officially supported only as cache devices.

This setup follows the approach documented here:  
https://github.com/007revad/Synology_HDD_db

**Rule:**  
No irreplaceable data is stored exclusively on `/volume2`.

---

## Folder Layout Convention

All container data follows a consistent layout:

```text
/volume1/docker/<service>/
/volume2/nvme-cache/<service>/
```

## User and Permissions

Containers run as non-root where supported.

Host directories are owned by a dedicated service user:
- UID and GID are consistent across services
- Permissions are set explicitly on volume paths

## Networking Assumptions

- The NAS has outbound internet access
- No inbound ports are forwarded on the router
- All external access is via Cloudflare Tunnel
- Docker bridge networking is used (no macvlan)
- Firewall on NAS is disabled

## Time and DNS

- System time is synchronized via NTP
- DNS resolution works from both host and containers
- No internal DNS server is required.
