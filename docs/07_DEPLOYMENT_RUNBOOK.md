# Curatal WorkHub — Phase 0: Staging & Production Deployment Runbook

**Document ID:** `docs/07_DEPLOYMENT_RUNBOOK.md`  
**Organization:** Curatal  
**Target Environment:** Staging & Self-Hosted Infrastructure  
**Author:** DevOps Engineer & Infrastructure Architect  

---

## 1. Overview & Dual Deployment Strategy

Curatal WorkHub supports two distinct deployment modes to maintain flexibility between upstream updates and Curatal customizations:

- **Mode A: Unmodified Upstream Baseline**  
  Deploys the verified official OpenProject container images (`openproject/openproject:17-slim`). Used for initial infrastructure benchmarking, staging validation, and upstream regression testing.
- **Mode B: Curatal Customized Build**  
  Builds versioned container images directly from `curatal-workhub` source code. Enables deploying Curatal-specific UI themes, Jira workflow extensions, custom fields, and GitHub integrations.

---

## 2. Staging Deployment Prerequisites

Before deploying to staging:
1. Docker Engine 24.0+ and Docker Compose v2.20+ installed.
2. Verified non-colliding host port assigned (default: `127.0.0.1:8088` or edge reverse proxy `80/443`).
3. Domain DNS record configured: `workhub-staging.curatal.internal` pointing to the staging host IP.
4. Independent staging database credentials configured in `.env`.
5. Minimum available host RAM: 4 GB; minimum free disk: 30 GB.

---

## 3. Step-by-Step Staging Deployment

### Step 1: Clone Infrastructure Repository
```bash
git clone -b curatal/infra-develop <CURATAL_INFRA_REPO_URL> curatal-workhub-infra
cd curatal-workhub-infra
```

### Step 2: Configure Environment Secrets
Copy `.env.example` and populate production-grade secrets:
```bash
cp .env.example .env
```
Ensure the following are set:
- `OPENPROJECT_HTTPS=true`
- `OPENPROJECT_HOST__NAME=workhub-staging.curatal.internal`
- `PORT=127.0.0.1:8088`
- `SECRET_KEY_BASE` (64+ character random hex string)
- `POSTGRES_PASSWORD` (32+ character random string)
- `COLLABORATIVE_SERVER_SECRET` (32+ character random string)

### Step 3: Validate Compose Specification
```bash
docker compose -f docker-compose.staging.yml config
```

### Step 4: Build Images (Mode B — Customized Build Only)
```bash
# In curatal-workhub source directory
COMMIT_SHA=$(git rev-parse --short HEAD)
BUILD_TAG="v17.9.1-curatal-${COMMIT_SHA}"

# Build custom application container
docker build -t curatal/workhub:${BUILD_TAG} -f docker/prod/Dockerfile .
```
Update `WORKHUB_IMAGE=curatal/workhub` and `TAG=${BUILD_TAG}` in staging `.env`.

### Step 5: Launch Staging Stack
```bash
docker compose -f docker-compose.staging.yml up -d
```

### Step 6: Post-Deployment Verification
```bash
# Verify container statuses and health
docker compose -f docker-compose.staging.yml ps

# Execute automated health check
curl -fsS http://127.0.0.1:8088/health_checks/default
```

---

## 4. Reverse Proxy & HTTPS Termination

In staging and production, HTTPS must terminate at the edge reverse proxy.

### 4.1 Caddy Edge Configuration
```caddyfile
workhub-staging.curatal.internal {
    reverse_proxy 127.0.0.1:8088 {
        header_up X-Forwarded-Proto {scheme}
        header_up Host {host}
    }
}
```

### 4.2 Nginx Edge Configuration
```nginx
server {
    listen 443 ssl http2;
    server_name workhub-staging.curatal.internal;

    ssl_certificate /etc/ssl/certs/workhub_staging.crt;
    ssl_certificate_key /etc/ssl/private/workhub_staging.key;

    location / {
        proxy_pass http://127.0.0.1:8088;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## 5. Rollback Procedure

If a staging release fails smoke testing:
1. Identify the previous stable image tag:
   ```bash
   docker image ls curatal/workhub
   ```
2. Update `TAG` in `.env` to the previous known-good tag (e.g. `TAG=v17.9.1-curatal-PREVIOUS_SHA`).
3. If database schema was migrated:
   - Restore database backup taken prior to deployment (refer to `docs/03_BACKUP_RESTORE_RUNBOOK.md`).
4. Recreate application containers:
   ```bash
   docker compose -f docker-compose.staging.yml up -d web worker cron
   ```
5. Confirm health check returns HTTP 200.
