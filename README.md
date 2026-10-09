# Curatal WorkHub

> **Internal Jira-like Software Development Lifecycle & Project Management Platform**  
> Built on **OpenProject Community Edition v17.9.1** (GPL-3.0)

Curatal WorkHub is Curatal's centralized, self-hosted platform for agile software development, sprint planning, issue and bug tracking, release roadmaps, and cross-functional QA workflows across Curatal's portfolio of candidate, recruiter, interviewer, assessment, and backend services.

---

## 1. Repository Structure

The workspace is organized into two independently versioned Git repositories, orchestrated by root documentation and automation:

```text
curatal_jira/
├── curatal-workhub/              # OpenProject Application Source Code
│   ├── app/                      # Ruby on Rails core models, controllers, and services
│   ├── frontend/                 # Angular single-page components and styling
│   ├── .github/workflows/ci.yml  # Application source integrity CI pipeline
│   └── (Branch: curatal/develop, Pinned Tag: v17.9.1, Commit: c7ec3efaeb2740944b0368b267901b568d6f2144)
│
├── curatal-workhub-infra/        # Docker & Infrastructure Automation
│   ├── docker-compose.yml        # Development Compose specification (127.0.0.1:8088)
│   ├── docker-compose.staging.yml# Hardened Staging Compose specification
│   ├── .env.example              # Documented environment secrets template
│   ├── scripts/                  # Lifecycle, backup, and health check scripts
│   │   ├── workhub.ps1           # Windows PowerShell lifecycle manager
│   │   ├── workhub.sh            # Linux/WSL2 lifecycle manager
│   │   ├── backup.ps1            # Timestamped PostgreSQL & asset backup runner
│   │   ├── restore.ps1           # Disaster recovery restoration runner
│   │   └── smoke_test.rb         # Automated regression smoke test runner
│   ├── Makefile                  # Developer CLI commands
│   └── (Branch: curatal/infra-develop, Tracked Upstream: stable/17)
│
├── docs/                         # Architectural & Operational Documentation
│   ├── 01_ENVIRONMENT_ASSESSMENT.md
│   ├── 02_OPENPROJECT_ARCHITECTURE_AUDIT.md
│   ├── 03_BACKUP_RESTORE_RUNBOOK.md
│   ├── 04_RESOURCE_BASELINE.md
│   ├── 05_JIRA_FEATURE_GAP_ANALYSIS.md
│   ├── 06_CURATAL_PROJECT_CONFIGURATION_BLUEPRINT.md
│   ├── 07_DEPLOYMENT_RUNBOOK.md
│   ├── 08_UPSTREAM_UPGRADE_STRATEGY.md
│   ├── 09_SECURITY_CHECKLIST.md
│   └── PHASE_0_COMPLETION_REPORT.md
│
└── .gitignore                    # Root protection for secrets and runtime data
```

---

## 2. Technical Stack Summary

| Layer | Technology | Details |
| :--- | :--- | :--- |
| **Backend Core** | Ruby on Rails 8.1 / 7.x (Ruby 4.0.6) | High-performance Ruby MRI runtime with YJIT + PRISM |
| **Web Server** | Puma 8.0.2 Cluster | 2 worker processes, 4–16 threads (`RAILS_MIN_THREADS=4`) |
| **Frontend UI** | Angular (TypeScript) + Primer Design System | Single-page reactive boards and work package detail panels |
| **Relational DB** | PostgreSQL 16 Alpine (`postgres:16-alpine`) | UTF-8/Unicode collation, 217 schema tables |
| **In-Memory Cache**| Memcached 1.6 (`memcached`) | Session storage and fragment query caching |
| **Collaborative** | OpenProject Hocuspocus 17.9.1 | Node.js WebSocket Yjs real-time editor |
| **Edge Proxy** | Caddy 2.x Reverse Proxy | Internal routing between Puma and Hocuspocus |
| **Background** | GoodJob (PostgreSQL backed) | Async notification dispatch, exports, and webhooks |

---

## 3. Quick Start Guide (Local Development)

### 3.1 Initial Setup
From the repository root or `curatal-workhub-infra/`:

**Windows (PowerShell):**
```powershell
cd curatal-workhub-infra
# 1. Generate .env with cryptographically secure random secrets
.\scripts\workhub.ps1 setup

# 2. Start services (db, cache, proxy, web, worker, hocuspocus, cron)
.\scripts\workhub.ps1 start
```

**Linux / macOS / WSL2:**
```bash
cd curatal-workhub-infra
./scripts/workhub.sh setup
./scripts/workhub.sh start
```

### 3.2 Accessing the Application
- **URL:** [http://127.0.0.1:8088/](http://127.0.0.1:8088/) or [http://localhost:8088/](http://localhost:8088/)
- **Health Check:** [http://127.0.0.1:8088/health_checks/default](http://127.0.0.1:8088/health_checks/default)
- **Initial Credentials:** Set during first login or managed via database admin.

---

## 4. Lifecycle & Operations Commands

| Operation | PowerShell Command | Makefile Command |
| :--- | :--- | :--- |
| **Start Containers** | `.\scripts\workhub.ps1 start` | `make start` |
| **Stop Containers** | `.\scripts\workhub.ps1 stop` | `make stop` |
| **Restart Containers** | `.\scripts\workhub.ps1 restart` | `make restart` |
| **Service Status** | `.\scripts\workhub.ps1 status` | `make status` |
| **Tail Application Logs** | `.\scripts\workhub.ps1 logs` | `make logs` |
| **Run Health Checks** | `.\scripts\workhub.ps1 healthcheck` | `make healthcheck` |
| **Create Backup** | `.\scripts\workhub.ps1 backup` | `make backup` |
| **Restore from Backup** | `.\scripts\workhub.ps1 restore <backup_dir>` | — |

---

## 5. Phase 0 Documentation Index

For in-depth architectural and operational specifications, consult the following deliverables:

- [01_ENVIRONMENT_ASSESSMENT.md](file:///d:/Curatal-Jira/curatal_jira/docs/01_ENVIRONMENT_ASSESSMENT.md) — Host metrics, active containers (`jobnavigator`, `livekit`), and isolation strategy.
- [02_OPENPROJECT_ARCHITECTURE_AUDIT.md](file:///d:/Curatal-Jira/curatal_jira/docs/02_OPENPROJECT_ARCHITECTURE_AUDIT.md) — Component breakdown, Mermaid architecture diagram, and license boundaries.
- [03_BACKUP_RESTORE_RUNBOOK.md](file:///d:/Curatal-Jira/curatal_jira/docs/03_BACKUP_RESTORE_RUNBOOK.md) — RPO/RTO metrics, pg_dump custom format backups, and recovery procedures.
- [04_RESOURCE_BASELINE.md](file:///d:/Curatal-Jira/curatal_jira/docs/04_RESOURCE_BASELINE.md) — Measured empirical RAM (~2.15 GiB) and CPU consumption.
- [05_JIRA_FEATURE_GAP_ANALYSIS.md](file:///d:/Curatal-Jira/curatal_jira/docs/05_JIRA_FEATURE_GAP_ANALYSIS.md) — Capability matrix comparing OpenProject CE vs. Jira.
- [06_CURATAL_PROJECT_CONFIGURATION_BLUEPRINT.md](file:///d:/Curatal-Jira/curatal_jira/docs/06_CURATAL_PROJECT_CONFIGURATION_BLUEPRINT.md) — Curatal's 11 projects and 14-state Dev-to-QA lifecycle.
- [07_DEPLOYMENT_RUNBOOK.md](file:///d:/Curatal-Jira/curatal_jira/docs/07_DEPLOYMENT_RUNBOOK.md) — Staging and self-hosted deployment procedures (Mode A & Mode B).
- [08_UPSTREAM_UPGRADE_STRATEGY.md](file:///d:/Curatal-Jira/curatal_jira/docs/08_UPSTREAM_UPGRADE_STRATEGY.md) — Upstream tracking, rebasing, and schema upgrade workflows.
- [09_SECURITY_CHECKLIST.md](file:///d:/Curatal-Jira/curatal_jira/docs/09_SECURITY_CHECKLIST.md) — Defense-in-depth posture, secret hygiene, and staging security gates.
- [PHASE_0_COMPLETION_REPORT.md](file:///d:/Curatal-Jira/curatal_jira/docs/PHASE_0_COMPLETION_REPORT.md) — Comprehensive technical completion report across Sections A–K.