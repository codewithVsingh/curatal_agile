# Curatal WorkHub — Phase 0: Environment Assessment Report

**Document ID:** `docs/01_ENVIRONMENT_ASSESSMENT.md`  
**Organization:** Curatal  
**Product:** Curatal WorkHub (OpenProject Community Edition)  
**Date:** October 9, 2026  
**Auditor / Role:** Senior Software Architect, DevOps & Infrastructure Security Engineer  

---

## 1. Executive Summary

This report documents the baseline infrastructure assessment of the host development environment hosting Curatal WorkHub. The assessment was conducted prior to installing application components or modifying infrastructure to guarantee isolation, verify capacity against official OpenProject operational guidelines, and safeguard active services running on the host machine.

**Key Findings:**
- **Host Capacity:** 10 CPU cores (16 logical threads), 31.69 GB total RAM (~5.0 GB currently free), and 272+ GB free disk space on drive `D:`. System resources substantially exceed the official OpenProject Community Edition baseline (4 cores, 4 GB RAM, 20 GB disk).
- **Active Business Services Detected:** The host actively runs business containers for `jobnavigator` (Caddy reverse proxy on ports 80/443, Python backend on port 8000, PostgreSQL 16 on internal 5432) and `livekit-server` (ports 7880, 7881, 7882). Unrelated local services also bind port 3000 (Node.js) and ports 5432/5433 (Host PostgreSQL).
- **Isolation Policy:** Curatal WorkHub **must not** bind host ports 80, 443, 3000, 5432, 5433, 7880, or 8000. It must operate on an isolated Docker network (`curatal_workhub_backend`, `curatal_workhub_frontend`), use dedicated persistent volumes (`curatal_workhub_pgdata`, `curatal_workhub_assets`), and bind its reverse proxy to a dedicated host port (`127.0.0.1:8088`).

---

## 2. Measured System Specifications

### 2.1 Hardware and Operating System
| Component | Measured Specification | Evaluation |
| :--- | :--- | :--- |
| **Operating System** | Microsoft Windows 11 Home Single Language (Build 26300, 64-bit) | Supported via Docker Desktop WSL2 engine |
| **Processor (CPU)** | 13th Gen Intel(R) Core(TM) i5-13450HX | 10 Cores (6P + 4E), 16 Logical Processors |
| **Total Visible RAM**| 33,231,568 KB (~31.69 GB RAM) | Ample headroom |
| **Free Physical RAM**| 5,221,512 KB (~4.98 GB available) | Sufficient for 2 web workers + 1 background worker |
| **Primary OS Disk (C:)**| Total: 527.8 GB \| Free: 80.9 GB | System runtime |
| **Data Drive (D:)** | Total: 400.9 GB \| Free: 272.2 GB | **Primary location for WorkHub repositories & volumes** |
| **Swap / Pagefile** | Managed by Windows & WSL2 `.wslconfig` | Monitor during container bootstrapping |

### 2.2 Container Runtime & Developer Tooling
| Tool | Installed Version | Status / Notes |
| :--- | :--- | :--- |
| **Git** | `2.54.0.windows.1` | Operational |
| **Docker Engine** | `29.7.2` (build `a7dcaa6`) | Running via Docker Desktop WSL2 engine |
| **Docker Compose** | `v5.5.0` | Compose v2+ compatible CLI |
| **WSL2 Runtime** | `docker-desktop` | Operational state: Running |

---

## 3. Co-Existing Workloads & Conflict Audit

A strict audit was performed to inventory existing host processes and Docker containers. In accordance with Curatal security and reliability mandates, **no unrelated service was interrupted, modified, or terminated**.

### 3.1 Existing Docker Containers
```text
CONTAINER ID   IMAGE                           PORTS                                NAMES
e31a82b1a58e   livekit/livekit-server:latest   0.0.0.0:7880-7881->7880-7881/tcp,    livekit-server
                                               0.0.0.0:7882->7882/udp
b7ecb41b4f0f   jobnavigator-frontend           80/tcp                               jobnavigator-frontend-1
1c81fe48df5d   jobnavigator-backend            127.0.0.1:8000->8000/tcp             jobnavigator-backend-1
00bd6f004517   postgres:16-alpine              5432/tcp                             jobnavigator-db-1
ebdc4d7b90b0   caddy:2-alpine                  0.0.0.0:80->80/tcp,                  jobnavigator-caddy-1
                                               0.0.0.0:443->443/tcp
```

### 3.2 Existing Docker Networks & Volumes
- **Networks:** `bridge`, `host`, `none`, `curatal-livekit_default`, `jobnavigator_default`, `tutors-parliament-rtc-infra_default`.
- **Volumes:** `jn_fe_node_modules`, `jobnavigator_caddy_config`, `jobnavigator_caddy_data`, `jobnavigator_codex_auth`, `jobnavigator_pgdata`.

### 3.3 Host Port Collision Matrix
| Host Port | Protocol | Owning Process / Container | Conflict with Stock OpenProject? | WorkHub Mitigation Strategy |
| :--- | :--- | :--- | :--- | :--- |
| **80 / 443** | TCP | `jobnavigator-caddy-1` | **YES** (Standard HTTP/HTTPS) | Do not bind 80/443 on host. Bind WorkHub proxy to `127.0.0.1:8088`. |
| **3000** | TCP | Host Node.js (`node.exe`, PID 39348) | **YES** (Default Rails Puma port) | Container web service remains internal on Docker network. |
| **5432** | TCP | Host PostgreSQL (`postgres.exe`, PID 7448) | **YES** (Default PostgreSQL port) | WorkHub DB port is **not** exposed on host. Exposed only on private bridge network. |
| **5433** | TCP | Host PostgreSQL (`postgres.exe`, PID 7688) | **YES** (Secondary Postgres port) | Kept isolated inside Docker network. |
| **7880–7882**| TCP/UDP| `livekit-server` | Potential RTC collision | No collision; WorkHub does not use LiveKit ports. |
| **8000** | TCP | `jobnavigator-backend-1` | Potential alternate web port | WorkHub uses port 8088. |
| **8090** | TCP | `WsToastNotification.exe` (PID 26076) | Minor local utility | Avoid port 8090. |

---

## 4. Resource Allocation & Worker Sizing for Phase 0

OpenProject Community Edition documentation recommends a baseline of:
- **4 CPU cores**
- **4 GB RAM**
- **20 GB disk**

### Target Configuration for 10–15 Users (Curatal Initial Scale)
To ensure reliable operation without starving the host or existing containers:
1. **Web Workers (Puma):**
   - 2 Puma cluster workers.
   - 4 to 16 threads (`RAILS_MIN_THREADS=4`, `RAILS_MAX_THREADS=16`).
   - Expected memory: ~800 MB to 1.2 GB.
2. **Background Worker (GoodJob / Delayed::Job):**
   - 1 background worker process.
   - Expected memory: ~500 MB to 800 MB.
3. **Database (PostgreSQL 17 / 16):**
   - Shared buffers: 256 MB.
   - Work memory: 16 MB.
   - Expected memory: ~300 MB to 600 MB.
4. **Cache (Memcached):**
   - Allocated memory: 64 MB.
5. **Realtime Collaborative Server (Hocuspocus):**
   - Node.js runtime: ~100 MB.
6. **Reverse Proxy (Caddy / Nginx):**
   - Reverse proxy: ~30 MB.

**Total Projected Memory Footprint:** ~2.0 GB – 2.8 GB RAM.  
**Available Free RAM:** ~5.0 GB RAM.  
**Verdict:** Green / High Readiness. The system has adequate capacity for the planned initial workload.

---

## 5. Security & Isolation Mandates

1. **Dedicated Docker Network:** All Curatal WorkHub containers must run attached to dedicated networks `curatal_workhub_frontend` and `curatal_workhub_backend`.
2. **No Shared Database Instances:** Under no circumstances will WorkHub connect to `jobnavigator-db-1` or host PostgreSQL instances. A dedicated containerized PostgreSQL instance with a unique database name and unique internal credentials must be provisioned.
3. **Localhost Binding:** Development proxy binds exclusively to `127.0.0.1:8088`.
4. **Secrets Separation:** Passwords, `SECRET_KEY_BASE`, and collaborative server secrets must reside strictly in local `.env` files that are `.gitignore`-excluded. No real credentials may be committed to Git.
5. **Zero Disruption Policy:** Any command modifying global Docker resources (such as `docker system prune -a --volumes`) is strictly prohibited.
