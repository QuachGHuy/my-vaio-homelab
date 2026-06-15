# Services Deployment

This document describes the containerized services running in the homelab, their storage architecture, deployment pattern, Docker networking model, and service access methods.

The objective is to provide a consistent deployment standard while maximizing performance by separating container workloads from bulk media storage.

---

## 1. Deployment Architecture

The homelab uses Docker Compose as the primary deployment method.

All user-facing applications run as containers and are managed through Portainer.

```text
Debian 13 Host
│
├── Docker Engine
│
├── proxy network (External Bridge)
│   ├── nginx-proxy-manager
│   ├── portainer
│   ├── navidrome
│   └── immich_server (Dual-homed: bridges proxy & internal network)
│
|── immich_internal network (Isolated)
|    ├── immich_server
|    ├── immich_postgres
|    └── immich_redis
│
├── SSD Storage
│   └── /var/lib/docker
│
└── HDD Storage
    └── /mnt/nas_storage
```

---

## 2. Storage Design

The homelab separates container workloads from media storage.

### SSD Storage

Docker is installed on a dedicated LVM volume:

```text
/var/lib/docker
```

Backed by:

```text
vg_docker
```

Purpose:

* Container images
* Docker volumes
* Overlay filesystem
* Databases
* Redis
* Application metadata
* Cache files

Advantages:

* Faster container startup
* Better database performance
* Lower application latency
* Improved random I/O performance

---

### HDD Storage

Large media files are stored separately on HDD storage:

```text
/mnt/nas_storage
```

Purpose:

* Music collections
* Photo libraries
* Future backups
* Long-term archival storage

Advantages:

* Lower cost per TB
* Reduced SSD wear
* Easier storage expansion

---

## 3. Directory Structure

Docker service definitions are organized under:

```text
~/docker
```

Current layout:

```text
~/docker
├── nginx-proxy-manager
│   ├── docker-compose.yml
│   └── data/
│
├── portainer
│   ├── docker-compose.yml
│   └── data/
│
├── navidrome
│   ├── docker-compose.yml
│   └── data/
│
└── immich
    ├── docker-compose.yml
    ├── .env
    ├── postgres/
    └── model-cache/
```

This structure keeps every service self-contained and simplifies backup, migration, and maintenance.

---

## 4. Shared Docker Network

Services accessed through Nginx Proxy Manager join a shared Docker network.

Create:

```bash
docker network create proxy
```

Verify:

```bash
docker network ls
```

Expected:

```text
proxy
```

Docker Compose example:

```yaml
networks:
  proxy:
    external: true
```

Benefits:

* Internal DNS resolution
* Container-to-container communication
* No dependency on host networking
* Simplified reverse proxy configuration

Example:

**navidrome** can be reached internally by NPM as:

```text
http://navidrome:4533
```

without exposing additional ports.

---

## 5. Portainer

### Purpose

Portainer provides a web-based interface for Docker management.

Functions:

* Container lifecycle management
* Volume inspection
* Network management
* Stack deployment
* Log inspection

---

### Deployment Path

```text
~/docker/portainer
```

---

### Persistent Storage

```text
portainer_data
```

Stored on:

```text
/var/lib/docker
```

---

### Access

Desktop / Laptop:

```text
https://portainer.homelab.local
```

---

## 6. Nginx Proxy Manager

### Purpose

Nginx Proxy Manager acts as the central reverse proxy.

Functions:

* HTTPS termination
* Internal certificate management
* Reverse proxy routing
* Host-based service discovery

---

### Deployment Path

```text
~/docker/nginx-proxy-manager
```

---

### Access

```text
https://npm.homelab.local
```

---

## 7. Navidrome

### Purpose

Navidrome provides self-hosted music streaming compatible with the Subsonic API.

Features:

* Web music player
* Mobile application support
* Artist and album management
* FLAC and lossless audio support

---

### Deployment Path

```text
~/docker/navidrome
```

---

### Storage Layout

Application data:

```text
/var/lib/docker
```

Music library:

```text
/mnt/nas_storage/<username>/music
```

Example:

```text
/mnt/nas_storage/me/music
```

This keeps large media files on HDD while maintaining fast application metadata on SSD.

---

### Access

Desktop:

```text
https://music.homelab.local
```

Android:

```text
http://192.168.100.199:4533
```

---

## 8. Immich

### Purpose

Immich provides self-hosted photo and video management.

Features:

* Mobile photo backup
* Facial recognition
* Semantic search
* Album management
* Timeline view

---

### Architecture

```text
Immich
├── immich_server
├── immich_postgres
└── immich_redis
```

---

### Deployment Path

```text
~/docker/immich
```

---

### Storage Layout

Application metadata:

```text
/var/lib/docker
```

Photo library:

```text
/mnt/nas_storage/<username>/gallery
```

Example:

```text
/mnt/nas_storage/me/gallery
```

Databases and cache remain on SSD while photos and videos reside on HDD.

---

### Access

Desktop:

```text
https://gallery.homelab.local
```

Android:

```text
http://192.168.100.199:2283
```

---

## 9. Service Verification

Verify running containers:

```bash
docker ps
```

Verify networks:

```bash
docker network ls
```

Verify volumes:

```bash
docker volume ls
```

Verify logs:

```bash
docker logs <container_name>
```

Example:

```bash
docker logs navidrome
docker logs immich_server
```

---

## 10. Current Service Inventory

| Service             | Purpose               | Access Method           |
| ------------------- | --------------------- | ----------------------- |
| Cockpit             | System Administration | system.homelab.local    |
| Portainer           | Docker Management     | portainer.homelab.local |
| Nginx Proxy Manager | Reverse Proxy         | npm.homelab.local       |
| Navidrome           | Music Streaming       | music.homelab.local     |
| Immich              | Photo Management      | gallery.homelab.local   |

---

## Future Expansion

Planned services:

* AdGuard Home
* Cloudflared Tunnel
* Monitoring Stack
* Centralized Logging

All future services should follow the same deployment pattern:

* Docker Compose
* Shared proxy network
* SSD for application data
* HDD for large persistent datasets
* Access through Nginx Proxy Manager
