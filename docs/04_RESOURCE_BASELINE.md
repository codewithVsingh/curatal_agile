# Curatal WorkHub — Phase 0: Resource Baseline & Observability Report

**Document ID:** `docs/04_RESOURCE_BASELINE.md`  
**Measurement Method:** Live empirical sampling (`docker stats --no-stream`) & host diagnostic queries  
**Organization:** Curatal  
**Date:** October 9, 2026  
**Author:** DevOps Engineer & Infrastructure Architect  

---

## 1. Executive Summary

A core objective of evaluating OpenProject Community Edition for Curatal WorkHub was validating that its operational footprint remains modest and predictable, avoiding the heavy memory consumption observed in competing alternatives like Plane.

**Empirical Result:**
Curatal WorkHub operates stably within **~2.15 GiB of total RAM** across all 7 micro-services (Web Puma cluster, Background Worker, PostgreSQL 16 database, Hocuspocus collaborative server, Memcached, Caddy reverse proxy, and Cron scheduler). Idle CPU consumption is below 0.5% aggregate.

---

## 2. Empirical Measured Resource Allocation

The following metrics were captured directly from the running Curatal WorkHub container cluster on the host system:

### 2.1 Container Service Breakdown
| Service / Container | Role | Measured RAM | Memory % (of Host) | Idle CPU % | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`curatal-workhub-infra-web-1`** | Puma Rails Web (2 workers) | **1.162 GiB** | 7.51% | 0.05% (idle) | Peak ~100% (1 core) during initial boot, then drops to idle |
| **`curatal-workhub-infra-worker-1`** | GoodJob Background Worker | **705.6 MiB** | 4.46% | 0.03% | Processes async queues, notifications, search indices |
| **`curatal-workhub-infra-hocuspocus-1`**| Realtime Collaborative Server | **136.1 MiB** | 0.86% | 0.16% | Node.js WebSocket Yjs server |
| **`curatal-workhub-infra-db-1`** | PostgreSQL 16 Alpine | **124.1 MiB** | 0.78% | 0.03% | Active connections pool, shared buffers |
| **`curatal-workhub-infra-cache-1`** | Memcached 1.6 | **17.31 MiB** | 0.11% | 0.04% | Fast in-memory key-value cache |
| **`curatal-workhub-infra-proxy-1`** | Caddy Reverse Proxy | **10.97 MiB** | 0.07% | 0.00% | Edge HTTP routing & WebSocket multiplexing |
| **`curatal-workhub-infra-cron-1`** | Scheduled Maintenance | **852 KiB** | 0.01% | 0.00% | Lightweight periodic runner |
| **TOTAL WORKHUB STACK** | **Complete Infrastructure** | **~2.15 GiB** | **13.8%** | **< 0.5%** | **Substantially below 4 GB baseline** |

---

## 3. Host System Impact & Co-Existence

### 3.1 Host Overview
- **Host CPU:** 13th Gen Intel Core i5-13450HX (10 physical cores, 16 logical cores).
- **Host Total RAM:** 31.69 GB.
- **Available RAM:** ~4.98 GB free physical memory prior to startup; ~2.8 GB free physical memory with all containers active.
- **Co-Existing Containers Impact:**
  - Existing business containers (`jobnavigator-frontend`, `jobnavigator-backend`, `jobnavigator-db`, `jobnavigator-caddy`, `livekit-server`) consume ~183 MiB combined.
  - Zero performance degradation or socket conflicts observed between Curatal WorkHub and the existing workloads.

---

## 4. Workload Condition Benchmarks

### Condition A: System Boot and Idle
- **Application Startup Duration:**
  - Database schema initialization + migration (`seeder`): ~55 seconds on first boot.
  - Web Puma boot time: ~30 seconds.
  - Proxy & cache startup: < 3 seconds.
- **Steady-State Idle Usage:**
  - Database: ~124 MiB RAM.
  - Web: ~1.16 GiB RAM.
  - Worker: ~705 MiB RAM.
  - CPU usage: Flutters between 0.02% and 0.20%.

### Condition B: Typical User Operations (Smoke Testing)
- **Actions Tested:**
  - Initial login page rendering (`/login`): 48.1 KB payload, server runtime ~1.08s (cold cache).
  - Health check verification (`/health_checks/default`): HTTP 200, response duration 25.35ms.
  - In-memory cache hit rate: > 90% for repeated session validation.

### Condition C: Concurrency Projection (10–30 Active Users)
With 2 Puma workers configured with 4–16 threads each (`RAILS_MIN_THREADS=4`, `RAILS_MAX_THREADS=16`):
- Max concurrent requests: 32 concurrent HTTP requests.
- Given typical internal SDLC usage (where 10–30 developers generate ~2–5 requests per second combined):
  - Current configuration operates well under maximum concurrency limits.
  - Headroom to scale: Puma can comfortably handle 30 users without increasing memory limits beyond 1.5 GB.

---

## 5. Optimization & Tuning Recommendations

1. **Keep Worker Count at 2:** Do not increase Puma worker count beyond 2 for < 30 users; each additional worker consumes ~500 MB RAM with negligible latency improvement for small teams.
2. **Puma Thread Sizing:** Current setting (`4-16` threads) provides optimal thread concurrency for I/O-bound database operations.
3. **Database Maintenance:** Schedule weekly `VACUUM ANALYZE` via cron to maintain query plan efficiency as work package volume increases.
4. **Memory Guardrails:** Maintain staging container limits (`web: 1536M`, `worker: 1024M`, `db: 1024M`) to prevent out-of-memory cascades on shared host servers.
