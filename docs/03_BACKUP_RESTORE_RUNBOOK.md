# Curatal WorkHub — Phase 0: Backup and Disaster Recovery Runbook

**Document ID:** `docs/03_BACKUP_RESTORE_RUNBOOK.md`  
**Organization:** Curatal  
**Product:** Curatal WorkHub  
**Target Environment:** Local Development & Staging  
**Author:** DevOps Engineer & Infrastructure Architect  

---

## 1. Overview and Recovery Objectives

This runbook establishes standard operational procedures for backing up and restoring Curatal WorkHub. Data loss prevention is critical to ensuring development continuity, issue traceability, and release stability.

### Key Recovery Metrics
- **Recovery Point Objective (RPO):** Maximum 24 hours of data loss for normal daily schedules; 0 hours prior to planned system upgrades (mandatory pre-upgrade snapshot).
- **Recovery Time Objective (RTO):** Full system restoration within 45 minutes on verified host infrastructure.

---

## 2. Backup Scope & Assets Inventory

A complete Curatal WorkHub backup consists of three independent components:

| Asset | Storage Location | Backup Mechanism | Format / File |
| :--- | :--- | :--- | :--- |
| **Relational Database** | PostgreSQL volume (`curatal_workhub_pgdata`) | `pg_dump -Fc` (PostgreSQL Custom Format) | `database.dump` |
| **Uploaded Assets** | POSIX asset volume (`curatal_workhub_opdata`) | `tar -czf` archive | `assets.tar.gz` |
| **Configuration Manifest** | Deployment directory | JSON snapshot | `manifest.json` |

> [!CAUTION]
> **Data Security Rule:** Backup files containing production or staging user data, issue descriptions, or attachments must **never** be checked into Git repositories. All backup archives must reside in `curatal-workhub-infra/backups/` which is strictly `.gitignore`-protected.

---

## 3. Automated Backup Procedures

### 3.1 Windows Environment (PowerShell)
Execute the provided management script:
```powershell
# Navigate to infra folder
cd d:\Curatal-Jira\curatal_jira\curatal-workhub-infra

# Run automated backup
.\scripts\workhub.ps1 backup
```

### 3.2 Linux / WSL2 Environment (Bash)
```bash
cd curatal-workhub-infra
./scripts/workhub.sh backup
```

### 3.3 Underlying Manual Commands (DevOps Direct Execution)
If executing outside the helper scripts:
```bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="backups/backup_${TIMESTAMP}"
mkdir -p "$BACKUP_DIR"

# 1. Database snapshot
docker compose exec -T db pg_dump -U postgres -Fc -d openproject > "${BACKUP_DIR}/database.dump"

# 2. Asset files snapshot
docker compose exec -T web tar -czf - -C /var/openproject assets > "${BACKUP_DIR}/assets.tar.gz"

# 3. Record metadata
cat <<EOF > "${BACKUP_DIR}/manifest.json"
{
  "timestamp": "${TIMESTAMP}",
  "version": "v17.9.1",
  "database": "openproject"
}
EOF
```

---

## 4. Disaster Recovery & Restoration Procedures

> [!WARNING]
> Restoring a database overwrites existing database tables. Always ensure a safety backup of the current state is created before executing any restoration against a non-disposable instance.

### 4.1 Step-by-Step Restoration Protocol

1. **Quiesce Application Services:**  
   Stop the web and background workers to prevent database writes or race conditions during the restore:
   ```powershell
   docker compose stop web worker cron
   ```
2. **Execute Database Restore:**  
   Restore tables and schema using `pg_restore`:
   ```powershell
   Get-Content "backups\backup_TIMESTAMP\database.dump" -Raw | docker compose exec -T db pg_restore -U postgres -d openproject --clean --if-exists --no-owner
   ```
3. **Restore Asset Files:**  
   Extract the archived attachments back into `/var/openproject`:
   ```powershell
   Get-Content "backups\backup_TIMESTAMP\assets.tar.gz" -Raw | docker compose exec -T web tar -xzf - -C /var/openproject
   ```
4. **Resume Services & Verify Health:**  
   Start application containers:
   ```powershell
   docker compose start web worker cron
   .\scripts\workhub.ps1 healthcheck
   ```

### 4.2 Restoring via PowerShell Helper Script
```powershell
.\scripts\workhub.ps1 restore -BackupFile "d:\Curatal-Jira\curatal_jira\curatal-workhub-infra\backups\backup_20261009_160000"
```

---

## 5. Retention Policy and Schedule

| Frequency | Retention Period | Target Storage | Pruning Rule |
| :--- | :--- | :--- | :--- |
| **Daily** | 7 Days | Local disk + Encrypted Object Storage | Purge snapshots older than 7 days |
| **Weekly** | 4 Weeks | Offsite / Secondary S3 Bucket | Retain end-of-week snapshot |
| **Pre-Upgrade** | Indefinite until validation | Local + Archival Store | Retained until milestone sign-off |

---

## 6. Disposable Verification Test Plan

Before declaring staging or production readiness, backup and restore **must be tested on a disposable instance**:
1. Seed test work packages with attachments.
2. Trigger `backup.ps1`.
3. Wipe the disposable database (`DROP DATABASE openproject; CREATE DATABASE openproject;`).
4. Execute `restore.ps1`.
5. Verify that all test work packages, relations, and attachments load intact.
