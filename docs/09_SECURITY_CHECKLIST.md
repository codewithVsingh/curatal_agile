# Curatal WorkHub — Phase 0: Security & Hardening Checklist

**Document ID:** `docs/09_SECURITY_CHECKLIST.md`  
**Classification:** Internal DevOps & Security Governance  
**Organization:** Curatal  
**Author:** Infrastructure Security Engineer & DevOps Architect  

---

## 1. Executive Security Summary

Curatal WorkHub hosts internal software development roadmaps, bug reports, architectural documentation, and team communications. Securing the WorkHub deployment without introducing unmanageable operational friction is paramount. This checklist validates the defense-in-depth posture implemented during Phase 0 and defines mandatory security gates for staging and production.

---

## 2. Phase 0 Implemented Security Controls

### 2.1 Credential & Secret Management
- [x] **Zero Plaintext Secrets in Version Control:** Both `curatal-workhub` and `curatal-workhub-infra` Git repositories are completely free of plaintext secrets, tokens, or live passwords.
- [x] **Strict `.gitignore` Boundaries:** `.env`, `.env.*` (excluding `.env.example`), `*.dump`, `*.sql`, `*.tar.gz`, and runtime logs are explicitly excluded across all repository boundaries.
- [x] **Cryptographic Secret Generation:** `scripts/workhub.ps1` and `scripts/workhub.sh` dynamically generate cryptographically strong pseudo-random keys (32-byte / 64-character hex strings) for `SECRET_KEY_BASE`, `COLLABORATIVE_SERVER_SECRET`, and `POSTGRES_PASSWORD`.

### 2.2 Network Isolation & Port Exposure
- [x] **Host Port Collision Elimination:** WorkHub web edge is restricted to `127.0.0.1:8088` on the local loopback interface, completely preventing exposure on public interfaces (`0.0.0.0`).
- [x] **Unexposed Database Port:** PostgreSQL port `5432` is strictly unmapped from the host system. It is accessible exclusively within the private Docker bridge network (`curatal_workhub_backend`).
- [x] **Separate Network Namespaces:** Services are isolated into `curatal_workhub_frontend` (accessible by proxy, web, and hocuspocus) and `curatal_workhub_backend` (accessible by web, worker, cron, db, and cache).
- [x] **Zero Interference with Existing Business Services:** WorkHub has no access to or shared state with existing host containers (`jobnavigator`, `livekit`).

### 2.3 Container & Runtime Hardening
- [x] **Non-Root Execution:** OpenProject web and worker services execute under non-privileged service user `app` (UID/GID 1000).
- [x] **Graceful Shutdown Traps:** Database container configured with `stop_grace_period: 10s` to prevent write buffer corruption during host shutdowns.
- [x] **Health Check Gates:** Cascading startup health checks ensure web and worker containers do not boot until PostgreSQL reports ready (`pg_isready`).

---

## 3. Staging & Production Security Readiness Checklist

Prior to promoting Curatal WorkHub to shared staging or production environments, the following gates must be satisfied:

| Category | Security Control | Verification Procedure | Status |
| :--- | :--- | :--- | :--- |
| **Transport Security** | Enforce HTTPS (TLS 1.3) at Reverse Proxy | Confirm HTTP redirects to HTTPS; verify valid TLS certificate | Mandatory for Staging |
| **HTTP Headers** | Enable HSTS (`OPENPROJECT_HSTS=true`) | Verify `Strict-Transport-Security` header in curl response | Configured in staging template |
| **Host Header Protection** | Restrict `OPENPROJECT_HOST__NAME` | Verify requests with arbitrary Host headers are rejected (HTTP 400) | Validated in Phase 0 |
| **Admin Credential** | Reset Default Admin Credentials | Rotate default initial password (`admin / admin`) on first login | Mandatory on launch |
| **User Registration** | Disable Public Self-Registration | Set Admin → Authentication → Self-registration to `disabled` | Phase 1 Admin Config |
| **Session Security** | Secure & HttpOnly Session Cookies | Inspect Set-Cookie response header for `Secure` and `HttpOnly` flags | Active when HTTPS=true |
| **Database Encryption** | Encrypt Persistent Volumes at Rest | Mount `curatal_workhub_pgdata` on BitLocker / LUKS encrypted block storage | Infrastructure prerequisite |
| **Automated Backups** | Encrypted Offsite Backup Shipping | Ensure `database.dump` archives are encrypted (GPG/AES-256) prior to S3 upload | Detailed in Backup Runbook |
| **Vulnerability Scanning** | Container CVE Scanning | Run `docker scout cves` or `trivy image` against pinned release | Phase 0 baseline: 0 critical |

---

## 4. Security Incident Response Protocol

In the event of suspected credential exposure or security compromise:
1. Immediately quiesce public access: `docker compose stop proxy web`.
2. Rotate `SECRET_KEY_BASE` in `.env` to invalidate all active user sessions immediately.
3. Rotate `POSTGRES_PASSWORD` and update `DATABASE_URL`.
4. Rotate `COLLABORATIVE_SERVER_SECRET`.
5. Restart containers: `docker compose up -d`.
6. Inspect Rails production audit log (`log/production.log`) for unauthorized data exfiltration.
