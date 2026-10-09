# Curatal WorkHub — Phase 0: OpenProject Architecture Audit

**Document ID:** `docs/02_OPENPROJECT_ARCHITECTURE_AUDIT.md`  
**Target Release:** OpenProject Community Edition v17.9.1 (Tag: `v17.9.1`, Docker Tag: `17-slim`)  
**Infrastructure Track:** `stable/17`  
**License:** GNU General Public License v3.0 (GPL-3.0)  
**Author:** Senior Software Architect & DevOps Engineer  

---

## 1. Executive Architecture Summary

Curatal WorkHub is built upon OpenProject Community Edition v17.9.1. OpenProject is an enterprise-grade open-source project management system designed with a modular Ruby on Rails core, an Angular-based reactive frontend, and decoupled containerized micro-services for background jobs, persistent caching, and real-time collaborative editing.

### Architecture Overview Diagram (Mermaid)

```mermaid
graph TD
    subgraph Client [Browser / Curatal Teams]
        Browser["User Browser (Desktop/Mobile)"]
    end

    subgraph EdgeProxy [Reverse Proxy Layer]
        Proxy["Proxy (Caddy / Nginx)<br/>Port: 127.0.0.1:8088"]
    end

    subgraph AppCluster [OpenProject Application Services (curatal_workhub_net)]
        Web["Web Service (Puma Cluster)<br/>Ruby on Rails 7.x / Ruby 3.3<br/>Port: 8080 (Internal)"]
        Hocus["Hocuspocus Server<br/>Node.js / WebSockets<br/>Live Collaborative Editing"]
        Worker["Background Worker<br/>GoodJob / Delayed::Job<br/>Async queues, notifications"]
        Cron["Cron Service<br/>Scheduled tasks & mail polling"]
        Seeder["Seeder / Migration Runner<br/>Database migrations & seeds"]
    end

    subgraph StorageCache [Data & Cache Infrastructure]
        DB[("PostgreSQL 16 / 17<br/>Database: openproject<br/>Volume: curatal_workhub_pgdata")]
        Cache["Memcached 1.6+<br/>Store: memcache:11211"]
        Assets[("Attachment Storage<br/>Path: /var/openproject/assets<br/>Volume: curatal_workhub_assets")]
    end

    Browser -->|HTTP/HTTPS Request| Proxy
    Browser -->|WebSocket Connection| Proxy
    Proxy -->|Reverse Proxy / HTTP| Web
    Proxy -->|WebSocket /hocuspocus| Hocus
    Web -->|SQL Connection Pool| DB
    Web -->|Session/Fragment Cache| Cache
    Web -->|Store Attachments| Assets
    Worker -->|Fetch Jobs & Update DB| DB
    Worker -->|Query / Invalidate| Cache
    Cron -->|Trigger Maintenance| Web
    Seeder -->|Run Migrations| DB
    Hocus -->|Auth Verification| Web
```

---

## 2. Component-by-Component Technical Audit

### 2.1 Backend Framework & Runtime
- **Runtime:** Ruby 3.3.x (MRI).
- **Core Framework:** Ruby on Rails 7.x.
- **Application Server:** Puma web server operating in cluster mode with 2 worker processes and multi-threaded request processing (`RAILS_MIN_THREADS=4`, `RAILS_MAX_THREADS=16`).
- **Database Abstraction:** ActiveRecord with PostgreSQL adapter (`pg` gem).
- **Directory Layout:**
  - `app/models/` — Core business models (WorkPackage, Project, User, Role, Status, Type, Version).
  - `app/controllers/` — REST and HTML controllers.
  - `app/services/` — Transactional business service objects (WorkPackages::CreateService, UpdateService, etc.).
  - `app/contracts/` — Form and API contract validation layers.
  - `modules/` — First-party plugins (bim, openproject-auth_plugins, storages, webhooks).

### 2.2 Frontend Architecture & Build System
- **Framework:** Hybrid architecture transitioning to Angular (TypeScript) for dynamic single-page components (interactive work package tables, Gantt charts, split-screen details, boards).
- **Legacy / Server-Rendered Views:** Rails ERB views, Turbo/Stimulus for fast partial updates.
- **Build Pipeline:** Angular CLI with Webpack compilation, outputting versioned assets into `public/assets`.
- **UI Design System:** Primer-inspired design tokens, accessible components, SCSS stylesheet hierarchy.

### 2.3 Application Server & Background Workers
- **Web Server:** Dedicated Puma container executing `./docker/prod/web`.
- **Background Worker:** Dedicated worker container executing `./docker/prod/worker`. It processes asynchronous tasks including:
  - In-app and email notification dispatch.
  - Work package activity journal auditing.
  - Export generation (PDF, XLS, CSV).
  - Full-text search index updates.
  - Webhook payload delivery.
- **Cron Scheduler:** Separate cron container executing `./docker/prod/cron` for scheduled system routines, reminder emails, and optional IMAP fetching.

### 2.4 Database & Persistence
- **Engine:** PostgreSQL 16 or 17 (UTF-8 / Unicode encoding).
- **Connection Pooling:** Controlled via `DATABASE_URL` with `pool=20&encoding=unicode&reconnect=true`.
- **Database Migrations:** Executed by the `seeder` container (`./docker/prod/seeder`) during container startup prior to launching the web and worker processes.
- **Supported DB Features:** Relational constraints, JSONB data types for flexible custom fields, full-text search dictionaries.

### 2.5 Caching & Session Management
- **Cache Engine:** Memcached.
- **Storage:** Configured via `OPENPROJECT_RAILS__CACHE__STORE="memcache"` and `OPENPROJECT_CACHE__MEMCACHE__SERVER="cache:11211"`.
- **Data Cached:** User session tokens, project permission maps, view fragments, and internationalization strings.

### 2.6 Collaborative Editing (Hocuspocus)
- **Technology:** Node.js WebSocket server utilizing the Hocuspocus library (Yjs CRDT implementation).
- **Functionality:** Real-time multi-user collaborative editing in description boxes and wiki pages.
- **Security:** Authenticated via shared secret (`COLLABORATIVE_SERVER_SECRET`) verified against the Rails core.

### 2.7 File Storage & Attachments
- **Default Storage:** POSIX filesystem mounted at `/var/openproject/assets`.
- **Content:** Work package attachments, avatars, exported documents, project documents.
- **Cloud Support:** Community Edition supports S3/MinIO ActiveStorage backend configuration via environment variables when cloud hosting is chosen.

### 2.8 APIs, Webhooks, and Extensibility
- **REST API v3:** Compliant with Hypertext Application Language (HAL+JSON) specification:
  - Endpoints: `/api/v3/work_packages`, `/api/v3/projects`, `/api/v3/users`, `/api/v3/versions`, `/api/v3/statuses`.
  - Authentication: Session cookies, HTTP Basic Auth with API Tokens, or OAuth2 access tokens.
- **Webhooks:** Native webhook support via `openproject-webhooks` module for event subscriptions (work package created/updated).
- **Git Integration:** Repository viewers and webhook-based commit linkage.

---

## 3. Community Edition vs. Enterprise Boundaries

A critical requirement of Phase 0 is establishing what OpenProject features are natively available in the Community Edition without violating GPL-3.0 or attempting to bypass license gates.

| Feature Category | Community Edition (CE) | Enterprise Edition (EE) | Curatal WorkHub Strategy |
| :--- | :--- | :--- | :--- |
| **Project Hierarchy** | Full support (unlimited nesting) | Full support | **Use native CE** |
| **Work Packages (Issues/Tasks/Bugs)** | Unlimited types, custom fields, statuses | Same | **Use native CE** |
| **Custom Workflows** | Granular status transitions by role | Same | **Use native CE** |
| **Standard Gantt & Timelines** | Full support | Full support | **Use native CE** |
| **Basic Kanban & Status Boards** | Available (Status-filtered lists) | Advanced Action Boards (Assignee/Subproject/Version) | **Use CE boards; plan UI enhancement** |
| **Backlog & Sprints** | Available in Backlogs module | Extended Agile Boards | **Configure CE Backlogs module** |
| **Story Points & Estimation** | Available via Backlogs / Custom fields | Same | **Configure custom fields / Backlogs** |
| **Issue Dependencies & Blockers** | Relations: "blocks", "precedes", "follows" | Same | **Use native CE** |
| **Attachments & Comments** | Rich text (Markdown/CKEditor) | Same | **Use native CE** |
| **REST API v3** | 100% feature-complete | Same | **Foundation for Curatal integrations** |
| **Webhooks** | Core events supported | Same | **Foundation for CI/CD integrations** |
| **SSO / SAML / OIDC** | Basic Omniauth plugins | Enterprise SSO configuration UI | **Community Omniauth or reverse proxy auth** |
| **2FA Enforcement** | Available per user | Enforced system-wide | **Configure per-user in CE** |
| **Custom Branding / Colors** | Basic CSS/Theming | Admin UI color picker | **Manage via custom CSS in Curatal build** |

---

## 4. Key Source Code Directories

When Curatal developers modify WorkHub in subsequent phases, changes will touch these primary directories in `curatal-workhub`:

```text
curatal-workhub/
├── app/
│   ├── models/            # Core domain entities (WorkPackage, Status, Type, Project)
│   ├── services/          # Business logic and transaction orchestration
│   ├── contracts/         # Input validations and authorization policies
│   ├── controllers/       # HTTP/REST controllers
│   └── views/             # Server-rendered templates
├── frontend/              # Angular single-page application source
│   ├── src/app/           # Angular components (wp-table, boards, details)
│   └── package.json       # Node.js dependencies
├── config/
│   ├── routes.rb          # URL routing
│   ├── database.yml       # PostgreSQL database connection configuration
│   └── environments/      # Environment-specific Rails settings
├── db/
│   ├── migrate/           # Database schema migrations
│   └── seeds/             # Default seeded data (types, statuses, workflows)
├── docker/
│   ├── prod/              # Production container entrypoint scripts (web, worker, cron, seeder)
│   └── dev/               # Development container scripts
└── lib/                   # Extensible modules and utilities
```

---

## 5. Architectural Verdict & Recommendations

1. **Scalability:** OpenProject's decoupled process model (separate Web, Worker, and Cron containers) allows Curatal WorkHub to scale cleanly from 10 users to 30+ users by tuning Puma threads and background worker concurrency without architectural redesign.
2. **Resource Predictability:** Using Memcached for caching and PostgreSQL 16/17 for persistence provides rock-solid stability within a modest ~2.5 GB RAM footprint.
3. **Extensibility:** The REST API v3, webhook system, and Angular frontend provide clean integration surfaces for connecting Curatal's existing portals (recruiter, candidate, interviewer, assessment) and GitHub CI/CD pipelines.
