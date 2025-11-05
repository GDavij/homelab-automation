# Database Management Role - Complete Technical Documentation

## Overview

The `database-management` role is a modular, production-ready Ansible role for deploying and managing database engines in a homelab environment. It currently implements **PostgreSQL 16 with pgvector** and includes an optional **pgAdmin 4** web interface.

**Version**: 1.0  
**Status**: Production Ready  
**Last Updated**: October 25, 2025

---

## Table of Contents

1. [Architecture](#architecture)
2. [Task Execution Flow](#task-execution-flow)
3. [Directory Structure](#directory-structure)
4. [Task Files Explained](#task-files-explained)
5. [Variables Reference](#variables-reference)
6. [Templates Reference](#templates-reference)
7. [Deployment Process](#deployment-process)
8. [Troubleshooting](#troubleshooting)
9. [Security Considerations](#security-considerations)
10. [Maintenance](#maintenance)

---

## Architecture

### Design Principles

1. **Modular Engine Support**: Each database engine (PostgreSQL, SQL Server, Oracle) has its own subdirectory
2. **Idempotent Operations**: All tasks can be run multiple times safely
3. **Security First**: Zero-trust networking via Tailscale, encrypted secrets, least-privilege users
4. **Production Ready**: Automated backups, health checks, audit logging, performance tuning
5. **Container Native**: Docker Compose orchestration with proper volume management

### Component Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    database-management Role                  │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  PostgreSQL  │  │  SQL Server  │  │    Oracle    │      │
│  │   (Active)   │  │   (Future)   │  │   (Future)   │      │
│  └──────┬───────┘  └──────────────┘  └──────────────┘      │
│         │                                                     │
│         ├─ Validate prerequisites                            │
│         ├─ Setup storage directories                         │
│         ├─ Deploy containers (docker-compose)                │
│         ├─ Initialize databases & users                      │
│         ├─ Configure backups                                 │
│         └─ Verify deployment                                 │
│                                                               │
│  ┌──────────────────────────────────────┐                   │
│  │         pgAdmin Sub-Role              │                   │
│  │  (Optional Web UI for PostgreSQL)    │                   │
│  ├──────────────────────────────────────┤                   │
│  │  - Deploy pgAdmin container          │                   │
│  │  - Fix directory permissions         │                   │
│  │  - Verify web UI accessibility       │                   │
│  └──────────────────────────────────────┘                   │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Network Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Tailscale VPN Network                   │
│                      (100.64.0.0/10)                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐         ┌──────────────────────┐          │
│  │   Any Device │         │  Homelab Server      │          │
│  │  on Tailnet  │────────▶│  100.84.146.121      │          │
│  └──────────────┘         └──────────┬───────────┘          │
│                                       │                       │
│                                       │                       │
│                      ┌────────────────┴────────────────┐     │
│                      │    Docker Bridge Network        │     │
│                      │    "self-hosted"                │     │
│                      ├─────────────────────────────────┤     │
│                      │                                 │     │
│                      │  ┌─────────────────┐            │     │
│                      │  │   PostgreSQL    │            │     │
│                      │  │   Container     │            │     │
│                      │  │  Port: 5432     │            │     │
│                      │  └─────────────────┘            │     │
│                      │                                 │     │
│                      │  ┌─────────────────┐            │     │
│                      │  │    pgAdmin      │            │     │
│                      │  │   Container     │            │     │
│                      │  │  Port: 5050     │            │     │
│                      │  └────────┬────────┘            │     │
│                      │           │                     │     │
│                      │           │ Accessed via:       │     │
│                      │           │ https://pg.homelab  │     │
│                      │           ▼                     │     │
│                      │  ┌─────────────────┐            │     │
│                      │  │ Nginx Proxy Mgr │            │     │
│                      │  │   (Reverse)     │            │     │
│                      │  └─────────────────┘            │     │
│                      │                                 │     │
│                      └─────────────────────────────────┘     │
│                                                               │
└─────────────────────────────────────────────────────────────┘

Access Patterns:
- PostgreSQL: Direct connection via Tailscale IP:5432
- pgAdmin: HTTPS via Nginx Proxy Manager at pg.homelab
```

---

## Task Execution Flow

### High-Level Execution Order

```
ansible-playbook server_playbook.yml --tags database
    │
    ├─▶ [1] Include database-management/tasks/main.yml
    │       │
    │       ├─▶ [2] Include postgres/main.yml
    │       │       │
    │       │       ├─▶ [2.1] validate.yml
    │       │       │     └─ Check passwords, validate config
    │       │       │
    │       │       ├─▶ [2.2] setup-storage.yml
    │       │       │     └─ Create all persistent directories
    │       │       │
    │       │       ├─▶ [2.3] deploy-postgresql.yml
    │       │       │     ├─ Pull images
    │       │       │     ├─ Template configs & env files
    │       │       │     ├─ Deploy SQL scripts
    │       │       │     └─ Launch docker-compose stack
    │       │       │
    │       │       ├─▶ [2.4] initialize-databases.yml
    │       │       │     └─ Execute SQL initialization scripts
    │       │       │
    │       │       ├─▶ [2.5] configure-backups.yml
    │       │       │     └─ Setup backup script & cron job
    │       │       │
    │       │       └─▶ [2.6] verify.yml
    │       │             └─ Health checks & connectivity tests
    │       │
    │       ├─▶ [3] Include pgadmin/deploy.yml (if enabled)
    │       │     ├─ Create directory with correct ownership
    │       │     ├─ Fix existing permissions (idempotent)
    │       │     └─ Restart container if needed
    │       │
    │       └─▶ [4] Include pgadmin/verify.yml (if enabled)
    │             └─ Verify pgAdmin is accessible
    │
    └─▶ Deployment Complete ✓
```

### Detailed Task Flow

#### Phase 1: Validation (`validate.yml`)
```yaml
Purpose: Pre-flight checks before deployment
Tasks:
  1. Assert POSTGRES_ROOT_PASSWORD is defined
  2. Assert POSTGRES_ADMIN_PASSWORD is defined
  3. Validate POSTGRES_DATABASES structure
  4. Validate POSTGRES_DATABASE_USERS structure
  5. Verify each user has a password
  6. Warn if backup encryption enabled without GPG key

Exit Condition: Fails if any required variable is missing
Idempotent: Yes (read-only checks)
```

#### Phase 2: Storage Setup (`setup-storage.yml`)
```yaml
Purpose: Create all persistent directories with correct ownership
Tasks:
  1. Create PostgreSQL directories:
     - data/      → 70:70 (postgres user), mode 0700
     - init/      → 70:70, mode 0755
     - conf/      → 70:70, mode 0750
     - backup/    → root:root, mode 0750
     - logs/      → 70:70, mode 0755
     - certs/     → 70:70, mode 0750
  
  2. Create backup subdirectories:
     - backup/daily/   → root:root, mode 0750
     - backup/weekly/  → root:root, mode 0750
     - backup/monthly/ → root:root, mode 0750
  
  3. Create pgAdmin directory (if enabled):
     - data/ → 5050:0 (pgadmin user), mode 0750

Exit Condition: All directories created successfully
Idempotent: Yes (checks existing directories)
Critical Fix: pgAdmin ownership corrected to 5050:0 (was root:root)
```

#### Phase 3: Container Deployment (`deploy-postgresql.yml`)
```yaml
Purpose: Deploy PostgreSQL and pgAdmin containers via Docker Compose
Tasks:
  1. Ensure Docker is running
  2. Pull container images:
     - postgres:16-alpine
     - dpage/pgadmin4:latest (if enabled)
  
  3. Create docker-compose project directory
  4. Copy docker-compose.yml
  5. Template environment files:
     - .env (main)
     - .postgresql.env
     - .pgadmin.env (if enabled)
  
  6. Template configuration files:
     - postgresql.conf (performance tuning)
     - pg_hba.conf (access control)
  
  7. Template SQL initialization scripts:
     - 01-enable-extensions.sql (uuid, pgvector)
     - 02-create-admin-role.sql (db_admin user)
     - 03-create-users.sql (application users)
     - 04-create-databases.sql (databases)
     - 05-setup-database-extensions.sql (per-db extensions)
     - 06-grant-permissions.sql (user permissions)
     - 07-pgvector-examples.sql (example queries)
     - 08-audit-log.sql (audit tables if enabled)
  
  8. Launch stack with docker-compose:
     - Profiles: ['pgadmin'] if PGADMIN_ENABLED
     - Recreate: auto (only if config changed)
  
  9. Wait for PostgreSQL health check (up to 50 seconds)
  10. Display deployment status

Exit Condition: PostgreSQL container healthy
Idempotent: Yes (docker-compose manages state)
Triggers: Restart if config files changed (via handler)
```

#### Phase 4: Database Initialization (`initialize-databases.yml`)
```yaml
Purpose: Execute SQL scripts to create databases, users, permissions
Tasks:
  1. Execute each SQL script inside container:
     - docker exec postgresql psql -U postgres -f /docker-entrypoint-initdb.d/<script>
  
  Scripts executed:
    01-enable-extensions.sql    → Enable uuid-ossp, pgvector
    02-create-admin-role.sql    → Create db_admin role
    03-create-users.sql         → Create application users
    04-create-databases.sql     → Create databases with owners
    05-setup-database-extensions.sql → Enable extensions per-db
    06-grant-permissions.sql    → Grant permissions to users
    08-audit-log.sql            → Create audit tables (if enabled)

Exit Condition: All scripts execute successfully
Idempotent: Yes (SQL uses IF NOT EXISTS, DO $$ blocks)
Changed When: Always marked as unchanged (informational only)
```

#### Phase 5: Backup Configuration (`configure-backups.yml`)
```yaml
Purpose: Setup automated backup system with retention policy
Tasks:
  1. Template backup script → /cold-storage/database/postgresql/backup/backup-script.sh
  2. Make script executable (mode 0750)
  3. Create systemd timer unit (optional)
  4. Create cron job:
     - Daily: 2:00 AM
     - Backup all databases
     - Rotate: 7 daily, 4 weekly, 6 monthly
     - Optional: GPG encryption

Exit Condition: Backup script and cron job created
Idempotent: Yes (cron module manages entries)
```

#### Phase 6: Verification (`verify.yml`)
```yaml
Purpose: Health checks and connectivity tests
Tasks:
  1. Check PostgreSQL container is running
  2. Verify PostgreSQL is accepting connections
  3. Test authentication with postgres user
  4. Test authentication with db_admin user
  5. List created databases
  6. Display connection information

Exit Condition: All checks pass
Idempotent: Yes (read-only queries)
```

#### Phase 7: pgAdmin Deployment (`pgadmin/deploy.yml`)
```yaml
Purpose: Deploy pgAdmin with correct permissions
Tasks:
  1. Check if pgAdmin is enabled
  2. Create pgAdmin data directory:
     - Owner: 5050 (pgadmin user)
     - Group: 0 (root group)
     - Mode: 0750
  
  3. Fix ownership of existing files (if needed):
     - Check current ownership
     - Only run chown if ownership is wrong
     - Mark as changed only if actually fixed
  
  4. Check pgAdmin container status
  5. Restart container ONLY if:
     - pgAdmin is enabled
     - Permissions were fixed
     - Container is running
  
  6. Wait 5 seconds after restart (if restarted)

Exit Condition: pgAdmin running with correct permissions
Idempotent: Yes (conditional checks prevent unnecessary changes)
Critical: Only restarts if permissions were actually fixed
```

#### Phase 8: pgAdmin Verification (`pgadmin/verify.yml`)
```yaml
Purpose: Verify pgAdmin is accessible
Tasks:
  1. Check pgAdmin container health
  2. Verify no permission errors in logs
  3. Display access information:
     - URL: https://pg.homelab
     - Login: PGADMIN_EMAIL
     - Password: PGADMIN_PASSWORD

Exit Condition: pgAdmin accessible
Idempotent: Yes (read-only checks)
```

---

## Directory Structure

```
roles/database-management/
├── README.md                          # Role overview
├── meta/
│   └── main.yml                       # Role metadata & dependencies
├── handlers/
│   └── main.yml                       # Event handlers (restart containers)
├── tasks/
│   ├── main.yml                       # Main orchestration file
│   ├── README.md                      # Task documentation
│   ├── postgres/                      # PostgreSQL engine tasks
│   │   ├── main.yml                   # PostgreSQL orchestration
│   │   ├── validate.yml               # Pre-flight checks
│   │   ├── setup-storage.yml          # Directory creation
│   │   ├── deploy-postgresql.yml      # Container deployment
│   │   ├── initialize-databases.yml   # SQL execution
│   │   ├── configure-backups.yml      # Backup system setup
│   │   └── verify.yml                 # Health checks
│   └── pgadmin/                       # pgAdmin sub-role tasks
│       ├── deploy.yml                 # pgAdmin deployment & permissions
│       └── verify.yml                 # pgAdmin verification
├── files/
│   ├── docker-compose.yml             # Docker Compose stack definition
│   └── .dockerignore                  # Docker build ignore file
└── templates/
    └── postgres/
        ├── .env.j2                    # Main environment variables
        ├── .postgresql.env.j2         # PostgreSQL environment
        ├── .pgadmin.env.j2            # pgAdmin environment
        ├── postgresql.conf.j2         # PostgreSQL config (performance)
        ├── pg_hba.conf.j2             # Access control config
        ├── backup-script.sh.j2        # Backup automation script
        └── *.sql.j2                   # SQL initialization scripts
```

---

## Task Files Explained

### `tasks/main.yml`

**Purpose**: Role entry point and orchestrator  
**Responsibilities**:
- Deploy all implemented database engines
- Include engine-specific task files
- Include pgAdmin tasks if enabled
- Apply tags for selective execution

**Key Logic**:
```yaml
# Deploy PostgreSQL
- include_tasks: postgres/main.yml
  
# Deploy pgAdmin (optional)
- include_tasks: pgadmin/deploy.yml
  when: PGADMIN_ENABLED | bool
```

**Tags**: `database`, `database-management`, `postgres`, `pgadmin`, `database-bkp`

---

### `tasks/postgres/validate.yml`

**Purpose**: Pre-flight validation before deployment  
**When It Runs**: First task in PostgreSQL deployment  
**Can Fail**: Yes - will abort deployment if checks fail

**Validations**:
1. ✓ `POSTGRES_ROOT_PASSWORD` is defined and not empty
2. ✓ `POSTGRES_ADMIN_PASSWORD` is defined and not empty
3. ✓ `POSTGRES_DATABASES` is a list (can be empty)
4. ✓ `POSTGRES_DATABASE_USERS` is a list (can be empty)
5. ✓ Each user in `POSTGRES_DATABASE_USERS` has a password
6. ⚠ Warns if encryption enabled but no GPG key configured

**Example Error**:
```
TASK [Ensure PostgreSQL root password is defined]
fatal: [100.84.146.121]: FAILED! => {
    "msg": "POSTGRES_ROOT_PASSWORD must be defined in group_vars/database-management/postgres/secrets.yml (use ansible-vault)"
}
```

**Idempotency**: Perfect (read-only assertions)

---

### `tasks/postgres/setup-storage.yml`

**Purpose**: Create all persistent storage directories with correct ownership  
**When It Runs**: After validation, before deployment  
**Can Fail**: Yes - will abort if directory creation fails (rare)

**Directories Created**:

| Path | Owner:Group | Mode | Purpose |
|------|-------------|------|---------|
| `{{ POSTGRESQL_DATA_PATH }}` | 70:70 | 0700 | PostgreSQL data files |
| `{{ POSTGRESQL_INIT_PATH }}` | 70:70 | 0755 | SQL init scripts |
| `{{ POSTGRESQL_CONF_PATH }}` | 70:70 | 0750 | Config files |
| `{{ POSTGRESQL_BACKUP_PATH }}` | root:root | 0750 | Backup storage |
| `{{ POSTGRESQL_LOGS_PATH }}` | 70:70 | 0755 | PostgreSQL logs |
| `{{ POSTGRESQL_CERTS_PATH }}` | 70:70 | 0750 | SSL certificates |
| `{{ POSTGRESQL_BACKUP_PATH }}/daily` | root:root | 0750 | Daily backups |
| `{{ POSTGRESQL_BACKUP_PATH }}/weekly` | root:root | 0750 | Weekly backups |
| `{{ POSTGRESQL_BACKUP_PATH }}/monthly` | root:root | 0750 | Monthly backups |
| `{{ PGADMIN_DATA_PATH }}` | 5050:0 | 0750 | pgAdmin data (if enabled) |

**Critical Fix Applied** (Oct 25, 2025):
- **Before**: pgAdmin directory created as `root:root`
- **After**: pgAdmin directory created as `5050:0`
- **Reason**: pgAdmin container runs as uid=5050, needs write access

**Idempotency**: Perfect (Ansible `file` module checks existing state)

**Tags**: `postgres`, `setup`, `storage`, `pgadmin`

---

### `tasks/postgres/deploy-postgresql.yml`

**Purpose**: Deploy PostgreSQL and pgAdmin containers using Docker Compose  
**When It Runs**: After storage setup  
**Can Fail**: Yes - will abort if containers fail to start

**Steps**:

1. **Ensure Docker Running** (`include_tasks: ../../common/tasks/ensure_docker.yml`)
   - Validates Docker daemon is active
   - Ensures docker-compose binary exists

2. **Pull Images**
   ```yaml
   - postgres:16-alpine          (PostgreSQL)
   - dpage/pgadmin4:latest       (pgAdmin, if enabled)
   ```

3. **Create Project Directory**
   - Location: `{{ DATABASE_MANAGEMENT_PROJECT_DIR }}`
   - Default: `/cold-storage/docker/database-management`

4. **Deploy Configuration Files**
   - `docker-compose.yml` (copied from files/)
   - `.env` (templated from .env.j2)
   - `.postgresql.env` (templated)
   - `.pgadmin.env` (templated, if enabled)
   - `postgresql.conf` (performance tuning)
   - `pg_hba.conf` (access control)

5. **Deploy SQL Scripts**
   - Templates 8 SQL files to `{{ POSTGRESQL_INIT_PATH }}`
   - Scripts are idempotent (safe to re-run)

6. **Launch Docker Compose Stack**
   ```yaml
   community.docker.docker_compose_v2:
     project_src: "{{ DATABASE_MANAGEMENT_PROJECT_DIR }}"
     project_name: database_management
     profiles: ['pgadmin'] if PGADMIN_ENABLED
     state: present
     pull: policy
     recreate: auto  # Only recreates if config changed
   ```

7. **Wait for Health Check**
   - Retries: 10 times (default)
   - Delay: 5 seconds between retries
   - Max Wait: 50 seconds
   - Checks: PostgreSQL container health status

8. **Display Status**
   - Container health
   - Tailscale access URL
   - Internal Docker network URL

**Handlers Triggered**:
- `Restart PostgreSQL stack` - if `postgresql.conf` or `pg_hba.conf` changed

**Idempotency**: Excellent (docker-compose manages state)
- First run: Creates containers
- Subsequent runs: No changes if config unchanged
- Config change: Recreates only affected containers

**Tags**: `postgres`, `deploy`, `containers`

---

### `tasks/postgres/initialize-databases.yml`

**Purpose**: Execute SQL initialization scripts to create databases, users, and permissions  
**When It Runs**: After containers are healthy  
**Can Fail**: Yes - will abort if SQL execution fails

**Scripts Executed (in order)**:

1. **01-enable-extensions.sql**
   - Enables `uuid-ossp` extension (UUID generation)
   - Enables `pgvector` extension (vector similarity search for AI)

2. **02-create-admin-role.sql**
   - Creates `db_admin` role with CREATEDB and CREATEROLE privileges
   - Sets password from `POSTGRES_ADMIN_PASSWORD`

3. **03-create-users.sql**
   - Creates all users defined in `POSTGRES_DATABASE_USERS`
   - Sets passwords, connection limits, privileges

4. **04-create-databases.sql**
   - Creates all databases defined in `POSTGRES_DATABASES`
   - Sets owners, encoding, locale

5. **05-setup-database-extensions.sql**
   - Enables per-database extensions (uuid, pgvector)
   - Based on `enable_uuid`, `enable_pgvector` flags

6. **06-grant-permissions.sql**
   - Grants CONNECT, CREATE, full access to `allowed_users`
   - Grants SELECT-only access to `readonly_users`

7. **08-audit-log.sql**
   - Creates audit_log tables (if `enable_audit_log: true`)

**Execution Method**:
```bash
docker exec postgresql psql -U postgres -f /docker-entrypoint-initdb.d/<script>
```

**Idempotency**: Excellent (all SQL uses `IF NOT EXISTS` or `DO $$ BEGIN ... EXCEPTION` blocks)
- First run: Creates all objects
- Subsequent runs: Skips existing objects
- Always marked as "ok" (not "changed")

**Tags**: `postgres`, `database`, `init`

---

### `tasks/postgres/configure-backups.yml`

**Purpose**: Setup automated backup system with retention policy  
**When It Runs**: After database initialization  
**Can Fail**: No (marked as optional)

**Backup Strategy**:
- **Daily backups**: 7 days retention
- **Weekly backups**: 4 weeks retention
- **Monthly backups**: 6 months retention
- **Method**: pg_dumpall (full cluster backup)
- **Optional**: GPG encryption

**Tasks**:

1. **Deploy Backup Script**
   - Template: `backup-script.sh.j2`
   - Destination: `{{ POSTGRESQL_BACKUP_PATH }}/backup-script.sh`
   - Mode: 0750 (executable)
   - Features:
     - Rotation logic
     - Error handling
     - Logging
     - Optional GPG encryption

2. **Create Cron Job**
   ```yaml
   cron:
     name: "PostgreSQL automated backup"
     minute: "0"
     hour: "2"
     job: "{{ POSTGRESQL_BACKUP_PATH }}/backup-script.sh >> /var/log/postgresql-backup.log 2>&1"
   ```

**Backup Script Logic**:
```bash
# 1. Create timestamp
DATE=$(date +%Y-%m-%d_%H-%M-%S)

# 2. Dump all databases
docker exec postgresql pg_dumpall -U postgres > backup_${DATE}.sql

# 3. Optionally encrypt
gpg --encrypt --recipient $GPG_RECIPIENT backup_${DATE}.sql

# 4. Move to retention folder (daily/weekly/monthly)
# Daily: Keep 7 days
# Weekly: Keep 4 weeks (created every Sunday)
# Monthly: Keep 6 months (created on 1st of month)

# 5. Delete old backups
find daily/ -mtime +7 -delete
find weekly/ -mtime +28 -delete
find monthly/ -mtime +180 -delete
```

**Idempotency**: Perfect (cron module manages entries)

**Tags**: `postgres`, `backup`, `database-bkp`

---

### `tasks/postgres/verify.yml`

**Purpose**: Post-deployment health checks and connectivity tests  
**When It Runs**: After backup configuration  
**Can Fail**: No (informational only)

**Verification Steps**:

1. **Container Status Check**
   ```bash
   docker ps --filter name=postgresql
   ```

2. **Connection Test (postgres user)**
   ```bash
   docker exec postgresql psql -U postgres -c "SELECT version();"
   ```

3. **Connection Test (db_admin user)**
   ```bash
   docker exec postgresql psql -U db_admin -c "SELECT current_user;"
   ```

4. **List Databases**
   ```bash
   docker exec postgresql psql -U postgres -c "\l"
   ```

5. **Display Connection Info**
   - Tailscale URL: `postgresql://100.84.146.121:5432`
   - Internal URL: `postgresql://postgresql:5432`
   - Available databases
   - Available users

**Idempotency**: Perfect (read-only queries)

**Tags**: `postgres`, `verify`

---

### `tasks/pgadmin/deploy.yml`

**Purpose**: Deploy pgAdmin with correct permissions (IDEMPOTENT FIX)  
**When It Runs**: After PostgreSQL deployment (if `PGADMIN_ENABLED: true`)  
**Can Fail**: No (uses conditional logic)

**Critical Feature**: This task was fixed on Oct 25, 2025 to be truly idempotent and only restart when needed.

**Tasks**:

1. **Check if Enabled**
   ```yaml
   debug: "pgAdmin deployment: ENABLED/DISABLED"
   ```

2. **Create Directory with Correct Ownership**
   ```yaml
   file:
     path: "{{ PGADMIN_DATA_PATH }}"
     state: directory
     owner: "5050"     # pgadmin user
     group: "0"        # root group
     mode: '0750'
   register: pgadmin_dir_created
   ```
   - **Idempotent**: Yes (checks existing directory)
   - **Changed**: Only if directory didn't exist or wrong ownership

3. **Fix Existing File Ownership (Smart Check)**
   ```bash
   # Only fix if needed
   current_owner=$(stat -c '%u' "$PGADMIN_DATA_PATH")
   if [ "$current_owner" != "5050" ] || \
      [ -n "$(find $PGADMIN_DATA_PATH ! -user 5050)" ]; then
     chown -R 5050:0 "$PGADMIN_DATA_PATH"
     echo "changed"
   else
     echo "ok"
   fi
   ```
   - **Idempotent**: Yes (checks before changing)
   - **Changed**: Only if ownership was wrong

4. **Check Container Status**
   ```bash
   docker inspect -f '{{.State.Running}}' pgadmin
   ```
   - Returns: "true", "false", or "not_found"
   - **Idempotent**: Yes (read-only, `changed_when: false`)

5. **Conditional Restart**
   ```yaml
   when:
     - PGADMIN_ENABLED | bool
     - pgadmin_ownership_fix.changed | bool
     - pgadmin_container_status.stdout == "true"
   ```
   - **Only restarts if ALL conditions true**:
     - pgAdmin is enabled
     - Permissions were actually fixed
     - Container is running
   - **Idempotent**: Yes (only runs when needed)

6. **Wait After Restart**
   ```yaml
   pause: seconds=5
   when: pgadmin_restart_result is changed
   ```
   - Only waits if restart actually happened

**Idempotency Proof**:
```
Run 1 (wrong permissions):
  - Create directory: ok (existed)
  - Fix ownership: CHANGED (5050:0)
  - Check container: ok (running)
  - Restart: CHANGED (restarted)
  - Wait: RUN (5 seconds)
  Result: 2 changes

Run 2 (correct permissions):
  - Create directory: ok (correct)
  - Fix ownership: ok (already correct)
  - Check container: ok (running)
  - Restart: SKIPPED (no ownership change)
  - Wait: SKIPPED (no restart)
  Result: 0 changes ✓

Run 3:
  Same as Run 2 ✓
```

**Tags**: `database`, `database-management`, `pgadmin`

---

### `tasks/pgadmin/verify.yml`

**Purpose**: Verify pgAdmin is accessible and working  
**When It Runs**: After pgAdmin deployment (if enabled)  
**Can Fail**: No (informational only)

**Verification Steps**:

1. **Check Container Health**
   ```bash
   docker ps --filter name=pgadmin
   ```

2. **Check for Permission Errors**
   ```bash
   docker logs pgadmin 2>&1 | grep -i "permission denied" | wc -l
   ```
   - Expected: 0 errors

3. **Display Access Information**
   ```
   ============================================================
   pgAdmin Web Interface
   ============================================================
   URL: https://pg.homelab (via Nginx Proxy Manager)
   Login Email: {{ PGADMIN_EMAIL }}
   Password: {{ PGADMIN_PASSWORD }}
   
   To add PostgreSQL server in pgAdmin:
     Host: postgresql
     Port: 5432
     Username: postgres or db_admin
     Password: (from secrets.yml)
   ============================================================
   ```

**Idempotency**: Perfect (read-only)

**Tags**: `database`, `database-management`, `pgadmin`, `verify`

---

## Variables Reference

### Required Variables

These **must** be defined in `group_vars/database-management/postgres/secrets.yml` (encrypted with ansible-vault):

```yaml
POSTGRES_ROOT_PASSWORD: "your_secure_password"
POSTGRES_ADMIN_PASSWORD: "your_admin_password"
PGADMIN_PASSWORD: "your_pgadmin_password"  # If PGADMIN_ENABLED: true
```

### PostgreSQL Configuration

Defined in `group_vars/database-management/postgres/vars.yml`:

```yaml
# Version
POSTGRESQL_VERSION: "16-alpine"

# Storage Paths
POSTGRESQL_DATA_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/data"
POSTGRESQL_INIT_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/init"
POSTGRESQL_CONF_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/conf"
POSTGRESQL_BACKUP_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/backup"
POSTGRESQL_LOGS_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/logs"
POSTGRESQL_CERTS_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/conf/certs"

# Network
POSTGRESQL_PORT: 5432

# Performance (tuned for 16GB RAM server)
POSTGRES_SHARED_BUFFERS: "512MB"
POSTGRES_EFFECTIVE_CACHE_SIZE: "4GB"
POSTGRES_MAX_CONNECTIONS: 50
POSTGRES_WORK_MEM: "16MB"

# Backup
POSTGRES_BACKUP_ENABLE: true
POSTGRES_BACKUP_ENABLE_ENCRYPTION: false
POSTGRES_BACKUP_GPG_RECIPIENT: ""

# Database & User Configuration
POSTGRES_DATABASES: []
POSTGRES_DATABASE_USERS: []
```

### pgAdmin Configuration

Defined in `group_vars/database-management/pgadmin/vars.yml`:

```yaml
PGADMIN_ENABLED: true
PGADMIN_VERSION: "latest"
PGADMIN_PORT: 5050
PGADMIN_DATA_PATH: "{{ COLD_STORAGE_PATH }}/database/pgadmin/data"
PGADMIN_EMAIL: "admin@homelab.local"
PGADMIN_PROXY_HOST: "pg.homelab"
```

### Database & User Definition Examples

```yaml
POSTGRES_DATABASE_USERS:
  - username: myapp_user
    password: "{{ MYAPP_DB_PASSWORD }}"  # From secrets.yml
    connection_limit: 10
    can_create_db: false
    can_create_role: false
    is_superuser: false

POSTGRES_DATABASES:
  - name: myapp_production
    owner: myapp_user
    description: "Production database for myapp"
    encoding: "UTF8"
    locale: "en_US.UTF-8"
    connection_limit: -1  # Unlimited
    allowed_users:
      - backup_user
    readonly_users:
      - analytics_user
    enable_pgvector: true
    enable_uuid: true
    enable_audit_log: false
```

For complete variable reference, see: [`docs/ENVIRONMENT_VARIABLES.md`](./ENVIRONMENT_VARIABLES.md)

---

## Templates Reference

### Configuration Files

#### `templates/postgres/postgresql.conf.j2`
**Purpose**: PostgreSQL server configuration  
**Tuned for**: 16GB RAM homelab server with multiple services  
**Key Settings**:
```ini
# Memory
shared_buffers = {{ POSTGRES_SHARED_BUFFERS }}        # 512MB
effective_cache_size = {{ POSTGRES_EFFECTIVE_CACHE_SIZE }}  # 4GB
work_mem = {{ POSTGRES_WORK_MEM }}                    # 16MB

# Connections
max_connections = {{ POSTGRES_MAX_CONNECTIONS }}      # 50

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d.log'
log_statement = '{{ POSTGRES_LOG_STATEMENT }}'        # all, ddl, mod, none

# Performance
effective_io_concurrency = 200
random_page_cost = 1.1
```

#### `templates/postgres/pg_hba.conf.j2`
**Purpose**: Client authentication configuration (zero-trust)  
**Security**: Restricts access to Tailscale VPN only  
**Rules**:
```conf
# Type  Database  User      Address           Method
local   all       postgres                    peer
local   all       all                         peer
host    all       all       127.0.0.1/32      scram-sha-256
host    all       all       ::1/128           scram-sha-256
host    all       all       100.64.0.0/10     scram-sha-256  # Tailscale only
```

### Environment Files

#### `templates/postgres/.env.j2`
**Purpose**: Main environment variables for docker-compose  
**Variables**:
```bash
# Container images
POSTGRESQL_IMAGE={{ POSTGRESQL_IMAGE }}
PGADMIN_IMAGE={{ PGADMIN_IMAGE }}

# Storage paths
POSTGRESQL_DATA_PATH={{ POSTGRESQL_DATA_PATH }}
POSTGRESQL_INIT_PATH={{ POSTGRESQL_INIT_PATH }}
# ... etc

# Network
TAILSCALE_BIND_ADDRESS={{ TAILSCALE_IP_ADDRESS }}
POSTGRESQL_PORT={{ POSTGRESQL_PORT }}

# pgAdmin profile
PGADMIN_PROFILE={{ 'pgadmin' if PGADMIN_ENABLED else 'disabled' }}
```

#### `templates/postgres/.postgresql.env.j2`
**Purpose**: PostgreSQL container environment variables  
**Variables**:
```bash
POSTGRES_USER=postgres
POSTGRES_PASSWORD='{{ POSTGRES_ROOT_PASSWORD }}'
POSTGRES_DB=postgres
POSTGRES_SSLMODE={{ POSTGRES_SSL_MODE }}
POSTGRES_LOG_STATEMENT={{ POSTGRES_LOG_STATEMENT }}
POSTGRES_LOG_DURATION={{ POSTGRES_LOG_DURATION }}
```

#### `templates/postgres/.pgadmin.env.j2`
**Purpose**: pgAdmin container environment variables  
**Variables**:
```bash
PGADMIN_DEFAULT_EMAIL={{ PGADMIN_EMAIL }}
PGADMIN_DEFAULT_PASSWORD='{{ PGADMIN_PASSWORD }}'
PGADMIN_LISTEN_PORT={{ PGADMIN_PORT }}
PGADMIN_CONFIG_SERVER_MODE=True
PGADMIN_CONFIG_MASTER_PASSWORD_REQUIRED=True
```

### SQL Initialization Scripts

All scripts use idempotent patterns (`IF NOT EXISTS`, `DO $$ BEGIN ... EXCEPTION`).

#### `01-enable-extensions.sql.j2`
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS vector;
```

#### `02-create-admin-role.sql.j2`
```sql
DO $$
BEGIN
    IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'db_admin') THEN
        CREATE ROLE db_admin WITH LOGIN PASSWORD '{{ POSTGRES_ADMIN_PASSWORD }}'
            CREATEDB CREATEROLE;
    END IF;
END
$$;
```

#### `03-create-users.sql.j2`
```sql
{% for user in POSTGRES_DATABASE_USERS %}
DO $$
BEGIN
    IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = '{{ user.username }}') THEN
        CREATE ROLE {{ user.username }} WITH LOGIN PASSWORD '{{ user.password }}'
            {% if user.connection_limit is defined %}CONNECTION LIMIT {{ user.connection_limit }}{% endif %}
            {% if user.can_create_db %}CREATEDB{% endif %}
            {% if user.can_create_role %}CREATEROLE{% endif %}
            {% if user.is_superuser %}SUPERUSER{% endif %};
    END IF;
END
$$;
{% endfor %}
```

#### `04-create-databases.sql.j2`
```sql
{% for db in POSTGRES_DATABASES %}
SELECT 'CREATE DATABASE {{ db.name }} OWNER {{ db.owner }}'
WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = '{{ db.name }}')
\gexec
{% endfor %}
```

#### `06-grant-permissions.sql.j2`
```sql
{% for db in POSTGRES_DATABASES %}
-- Connect permission
{% for user in db.allowed_users %}
GRANT CONNECT ON DATABASE {{ db.name }} TO {{ user }};
{% endfor %}

-- Full permissions
\c {{ db.name }}
{% for user in db.allowed_users %}
GRANT ALL PRIVILEGES ON DATABASE {{ db.name }} TO {{ user }};
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO {{ user }};
{% endfor %}

-- Read-only permissions
{% for user in db.readonly_users %}
GRANT CONNECT ON DATABASE {{ db.name }} TO {{ user }};
GRANT USAGE ON SCHEMA public TO {{ user }};
GRANT SELECT ON ALL TABLES IN SCHEMA public TO {{ user }};
{% endfor %}
{% endfor %}
```

---

## Deployment Process

### Prerequisites

1. **Roles Deployed First**:
   ```bash
   ansible-playbook server_playbook.yml --tags docker,networking
   ```

2. **Variables Configured**:
   - `group_vars/database-management/engine.yml`
   - `group_vars/database-management/postgres/vars.yml`
   - `group_vars/database-management/postgres/secrets.yml` (encrypted)
   - `group_vars/database-management/pgadmin/vars.yml`
   - `group_vars/database-management/pgadmin/secrets.yml` (encrypted)

3. **Nginx Proxy Manager** (if using pgAdmin):
   ```bash
   ansible-playbook server_playbook.yml --tags nginx-manager
   ```

### Full Deployment

```bash
# Deploy complete database stack
ansible-playbook server_playbook.yml --tags database
```

**Expected Output**:
```
TASK [database-management : Validate PostgreSQL prerequisites] ✓
TASK [database-management : Create PostgreSQL storage directories] changed: 9
TASK [database-management : Deploy docker-compose.yml] changed: 1
TASK [database-management : Launch PostgreSQL stack] changed: 1
TASK [database-management : Wait for PostgreSQL health check] ✓
TASK [database-management : Execute initialization SQL scripts] ✓
TASK [database-management : Create backup cron job] changed: 1
TASK [database-management : Verify PostgreSQL deployment] ✓
TASK [database-management : Deploy pgAdmin] changed: 1
TASK [database-management : Verify pgAdmin] ✓

PLAY RECAP:
100.84.146.121: ok=45 changed=13 unreachable=0 failed=0
```

### Selective Deployment

```bash
# PostgreSQL only
ansible-playbook server_playbook.yml --tags postgres

# pgAdmin only
ansible-playbook server_playbook.yml --tags pgadmin

# Backup configuration only
ansible-playbook server_playbook.yml --tags database-bkp

# Verification only
ansible-playbook server_playbook.yml --tags verify
```

### Re-deployment (Idempotent)

```bash
# Run again - should show 0 changes
ansible-playbook server_playbook.yml --tags database
```

**Expected Output**:
```
PLAY RECAP:
100.84.146.121: ok=45 changed=0 unreachable=0 failed=0
```

### Fix pgAdmin Permissions (Standalone)

```bash
# If pgAdmin has permission issues
ansible-playbook fix_pgadmin_permissions.yml -i inventory.ini
```

---

## Troubleshooting

### PostgreSQL Won't Start

**Symptom**: Container exits immediately or health check fails

**Diagnosis**:
```bash
# Check logs
ansible homelab -i inventory.ini -m shell -a "docker logs postgresql --tail 50"

# Check container status
ansible homelab -i inventory.ini -m shell -a "docker ps -a --filter name=postgresql"
```

**Common Issues**:

1. **Permission Denied on Data Directory**
   ```
   Fix: Ensure directory owned by uid=70
   ansible homelab -i inventory.ini -m shell -a "chown -R 70:70 /cold-storage/database/postgresql/data" --become
   ```

2. **Port Already in Use**
   ```
   Fix: Check if another PostgreSQL instance is running
   ansible homelab -i inventory.ini -m shell -a "ss -tlnp | grep 5432"
   ```

3. **Invalid Configuration**
   ```
   Fix: Validate postgresql.conf syntax
   ansible homelab -i inventory.ini -m shell -a "docker exec postgresql postgres -C config_file -c 'SHOW ALL'"
   ```

### pgAdmin Won't Start

**Symptom**: Container crashes with "Permission denied" errors

**Diagnosis**:
```bash
# Check logs
ansible homelab -i inventory.ini -m shell -a "docker logs pgadmin --tail 50"

# Check directory ownership
ansible homelab -i inventory.ini -m shell -a "stat -c '%u:%g %a' /cold-storage/database/pgadmin/data" --become
```

**Fix**:
```bash
# Run the fix playbook
ansible-playbook fix_pgadmin_permissions.yml -i inventory.ini

# Or manually
ansible homelab -i inventory.ini -m shell -a "chown -R 5050:0 /cold-storage/database/pgadmin/data" --become
ansible homelab -i inventory.ini -m shell -a "docker restart pgadmin"
```

### Can't Connect to PostgreSQL

**Symptom**: Connection refused or authentication failed

**Diagnosis**:
```bash
# Test from server
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql psql -U postgres -c 'SELECT version();'"

# Test from Tailscale network
psql "postgresql://postgres:PASSWORD@100.84.146.121:5432/postgres"
```

**Common Issues**:

1. **pg_hba.conf Blocking Connection**
   ```
   Fix: Verify Tailscale CIDR is allowed
   ansible homelab -i inventory.ini -m shell -a "docker exec postgresql cat /etc/postgresql/pg_hba.conf"
   ```

2. **Wrong Password**
   ```
   Fix: Verify password in secrets.yml
   ansible-vault view group_vars/database-management/postgres/secrets.yml
   ```

3. **Firewall Blocking**
   ```
   Fix: Check firewall rules (should allow Tailscale)
   ansible homelab -i inventory.ini -m shell -a "firewall-cmd --list-all"
   ```

### Database Not Created

**Symptom**: Database defined in vars but doesn't exist

**Diagnosis**:
```bash
# List databases
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql psql -U postgres -c '\l'"

# Check init script logs
ansible homelab -i inventory.ini -m shell -a "docker logs postgresql | grep 'CREATE DATABASE'"
```

**Fix**:
```bash
# Re-run initialization
ansible-playbook server_playbook.yml --tags postgres -e "FORCE_REINIT=true"
```

### Backup Not Running

**Symptom**: No backups in `/cold-storage/database/postgresql/backup`

**Diagnosis**:
```bash
# Check cron job
ansible homelab -i inventory.ini -m shell -a "crontab -l | grep postgresql"

# Check backup script exists
ansible homelab -i inventory.ini -m shell -a "ls -lh /cold-storage/database/postgresql/backup/backup-script.sh" --become

# Test backup script manually
ansible homelab -i inventory.ini -m shell -a "/cold-storage/database/postgresql/backup/backup-script.sh" --become
```

**Fix**:
```bash
# Re-deploy backup configuration
ansible-playbook server_playbook.yml --tags database-bkp
```

---

## Security Considerations

### Network Security

✅ **Zero-Trust Networking**
- PostgreSQL only accessible via Tailscale VPN (100.64.0.0/10)
- pg_hba.conf explicitly denies all other networks
- No public internet exposure

✅ **Container Isolation**
- Containers run in isolated `self-hosted` Docker network
- No host network mode
- Minimal exposed ports

### Authentication

✅ **Strong Password Requirements**
- Root and admin passwords stored in ansible-vault
- SCRAM-SHA-256 authentication (modern, secure)
- No plaintext passwords in configs

✅ **User Segregation**
- Superuser (postgres): Emergency use only
- Admin (db_admin): Database creation/management
- Application users: Database-specific access
- Read-only users: SELECT permission only

### File Permissions

✅ **Least Privilege Principle**
- PostgreSQL data: 0700 (owner only)
- Config files: 0640 (owner read/write, group read)
- Backup directory: 0750 (root only)
- pgAdmin data: 0750 (pgadmin user only)

✅ **Correct Ownership**
- PostgreSQL: uid=70 (postgres user)
- pgAdmin: uid=5050 (pgadmin user)
- Backups: root (additional protection)

### Backup Security

✅ **Optional GPG Encryption**
```yaml
POSTGRES_BACKUP_ENABLE_ENCRYPTION: true
POSTGRES_BACKUP_GPG_RECIPIENT: "your.email@domain.com"
```

✅ **Secure Storage**
- Backups stored on cold storage mount
- Separate from live data
- Root-only access

### Audit Logging

✅ **Optional Per-Database Audit**
```yaml
POSTGRES_DATABASES:
  - name: sensitive_db
    enable_audit_log: true
```

Creates audit_log table tracking:
- INSERT, UPDATE, DELETE operations
- User who performed action
- Timestamp
- Old and new values

---

## Maintenance

### Regular Tasks

#### Weekly
```bash
# Check disk usage
ansible homelab -i inventory.ini -m shell -a "df -h /cold-storage/database"

# Verify backups are running
ansible homelab -i inventory.ini -m shell -a "ls -lh /cold-storage/database/postgresql/backup/daily" --become

# Check for errors in logs
ansible homelab -i inventory.ini -m shell -a "docker logs postgresql --since 7d | grep -i error"
```

#### Monthly
```bash
# Update container images
ansible-playbook server_playbook.yml --tags postgres

# Vacuum databases
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql vacuumdb -U postgres --all --analyze"

# Test backup restoration (in test environment)
```

#### Quarterly
```bash
# Review user permissions
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql psql -U postgres -c '\du'"

# Review database sizes
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql psql -U postgres -c 'SELECT pg_database.datname, pg_size_pretty(pg_database_size(pg_database.datname)) AS size FROM pg_database ORDER BY pg_database_size(pg_database.datname) DESC;'"

# Rotate old backups manually if needed
```

### Updating PostgreSQL Version

```yaml
# Edit group_vars/database-management/postgres/vars.yml
POSTGRESQL_VERSION: "17-alpine"  # From 16-alpine
```

```bash
# Deploy (will pull new image and recreate container)
ansible-playbook server_playbook.yml --tags postgres

# Verify
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql psql -U postgres -c 'SELECT version();'"
```

### Adding a New Database

See: [`docs/POSTGRES_DATABASE_MANAGEMENT.md`](./POSTGRES_DATABASE_MANAGEMENT.md#how-to-add-a-new-database)

**Quick Steps**:
1. Add password to `secrets.yml`
2. Add user to `POSTGRES_DATABASE_USERS` in `vars.yml`
3. Add database to `POSTGRES_DATABASES` in `vars.yml`
4. Run: `ansible-playbook server_playbook.yml --tags postgres`

### Removing a Database

```bash
# Connect to PostgreSQL
psql "postgresql://postgres:PASSWORD@100.84.146.121:5432/postgres"

# Drop database
DROP DATABASE database_name;

# Drop user
DROP ROLE username;

# Remove from vars.yml
# Re-run playbook
ansible-playbook server_playbook.yml --tags postgres
```

---

## Related Documentation

- **[PostgreSQL Management Guide](./POSTGRES_DATABASE_MANAGEMENT.md)** - Complete usage guide
- **[Environment Variables Reference](./ENVIRONMENT_VARIABLES.md#database-management-role)** - All variables
- **[Container User IDs Guide](./CONTAINER_USER_IDS.md)** - User ID reference
- **[pgAdmin Permission Fix Guide](./PGADMIN_PERMISSION_FIX.md)** - Troubleshooting permissions
- **[Database Management Plan](./DATABASE_MANAGEMENT_PLAN.md)** - Architecture decisions
- **[Deployment Guide](./DEPLOYMENT_GUIDE.md)** - Full deployment walkthrough

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-10-25 | - Initial production release<br>- Fixed pgAdmin permission issue<br>- Implemented idempotent pgAdmin deployment<br>- Added comprehensive documentation |

---

## Support

For issues, questions, or contributions:

1. Check existing documentation in `docs/` folder
2. Review task files for implementation details
3. Check logs: `docker logs postgresql` or `docker logs pgadmin`
4. Run with verbose output: `ansible-playbook -vvv`

---

## License

This role is part of the homelab Ansible project.

---

**Generated**: October 25, 2025  
**Maintainer**: Database Management Role  
**Status**: ✅ Production Ready
