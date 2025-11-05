# Environment Variables Reference

## Overview

This document provides a comprehensive reference of all environment variables used across the homelab Ansible automation. Variables are organized by role and sub-role for easy navigation.

**Files:**
- **`group_vars/homelab.yml`** - User-configurable variables (edit this)
- **`group_vars/secrets.yml`** - Encrypted passwords (ansible-vault)
- **`group_vars/database-management-internal.yml`** - Internal computed variables (DO NOT edit)

---

## Table of Contents

1. [Global Variables](#global-variables)
2. [Docker Role](#docker-role)
3. [Networking Role](#networking-role)
4. [NGINX Proxy Manager Role](#nginx-proxy-manager-role)
5. [Pi-hole Role](#pi-hole-role)
6. [AI Services](#ai-services)
   - [Ollama (LLM Server)](#ollama-llm-server)
   - [ComfyUI (Stable Diffusion)](#comfyui-stable-diffusion)
   - [Open WebUI (Chat Interface)](#open-webui-chat-interface)
7. [Database Management Role](#database-management-role)
   - [PostgreSQL Sub-Role](#postgresql-sub-role)
   - [pgAdmin Sub-Role](#pgadmin-sub-role)
   - [Internal Variables](#database-internal-variables)
8. [Development Role](#development-role)
   - [Python Sub-Task](#python-sub-task)
   - [Node.js Sub-Task](#nodejs-sub-task)
   - [.NET Sub-Task](#dotnet-sub-task)

---

## Global Variables

**File:** `group_vars/homelab.yml`

### Storage Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `COLD_STORAGE_PATH` | string | `/cold-storage` | Base path for persistent storage on HDD |

### Network Configuration

| Variable | Type | Source | Description |
|----------|------|--------|-------------|
| `TAILSCALE_IP_ADDRESS` | string | Auto-detected | Tailscale VPN IP (100.64.x.x) |

---

## Docker Role

**File:** `group_vars/homelab.yml`

No user-configurable variables. Docker CE is installed with standard configuration.

**See:** `roles/docker/README.md` for implementation details.

---

## Networking Role

**File:** `group_vars/homelab.yml`

### Tailscale VPN

Tailscale is configured via the networking role. The `TAILSCALE_IP_ADDRESS` variable is auto-detected during deployment.

**Network:** `self-hosted` - Internal Docker network for service communication

**See:** `roles/networking/README.md` for Tailscale setup.

---

## NGINX Proxy Manager Role

**File:** `group_vars/homelab.yml`

### Version Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `NGINX_PROXY_MANAGER_VERSION` | string | `"2"` | Docker image version |

### Storage Paths

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `NGINX_MANAGER_DATA_STORE_PATH` | string | `{{ COLD_STORAGE_PATH }}/services/nginx-proxy/` | NGINX configuration and data |
| `NGINX_MANAGER_CERTIFICATES_STORE_PATH` | string | `{{ COLD_STORAGE_PATH }}/services/nginx-proxy/certificates` | SSL/TLS certificates |

### Network Ports

| Service | Port | Access |
|---------|------|--------|
| Admin UI | 81 | Tailnet only |
| HTTP | 80 | Public |
| HTTPS | 443 | Public |

**See:** `roles/nginx-manager/README.md` for proxy configuration.

---

## Pi-hole Role

**File:** `group_vars/homelab.yml`

### Version Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `PIHOLE_VERSION` | string | `"latest"` | Docker image version |

### Storage Paths

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `PI_HOLE_CONFIG_STORAGE_PATH` | string | `{{ COLD_STORAGE_PATH }}/services/pihole/config` | Pi-hole configuration |
| `PI_HOLE_LOGS_STORAGE_PATH` | string | `{{ COLD_STORAGE_PATH }}/services/pihole/logs` | DNS query logs |

### Security

**File:** `group_vars/secrets.yml`

| Variable | Type | Description |
|----------|------|-------------|
| `PI_HOLE_SECURE_WEBPASSWORD` | string | Pi-hole admin UI password |

**Note:** Currently stored in `homelab.yml` - should be moved to `secrets.yml`.

**See:** `roles/pihole/README.md` for DNS configuration.

---

## AI Services

All AI services use NVIDIA RTX 3060 12GB GPU for acceleration.

---

### Ollama (LLM Server)

**File:** `group_vars/homelab.yml`

#### Version Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `OLLAMA_VERSION` | string | `"latest"` | Ollama Docker image version |

#### Storage Paths

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `OLLAMA_MODELS_PATH` | string | `{{ COLD_STORAGE_PATH }}/ai/ollama/models` | Downloaded LLM models |

#### Network Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `OLLAMA_SUBDOMAIN` | string | `"ollama"` | Subdomain for NGINX proxy (ollama.domain.local) |

#### Service Endpoints

| Endpoint | Port | Access |
|----------|------|--------|
| API | 11434 | Tailnet only |

**See:** `docs/AI_SERVICES_IMPLEMENTATION.md` for model management.

---

### ComfyUI (Stable Diffusion)

**File:** `group_vars/homelab.yml`

#### Version Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `COMFYUI_VERSION` | string | `"cu128-slim"` | yanwk/comfyui-boot image with CUDA 12.8 |

#### Storage Paths

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `COMFYUI_DATA_PATH` | string | `{{ COLD_STORAGE_PATH }}/ai/comfyui/data` | Generated images and workflows |
| `COMFYUI_MODELS_PATH` | string | `{{ COLD_STORAGE_PATH }}/ai/comfyui/models` | Stable Diffusion checkpoints |

#### Network Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `COMFYUI_SUBDOMAIN` | string | `"comfy"` | Subdomain for NGINX proxy (comfy.domain.local) |

#### Service Endpoints

| Endpoint | Port | Access |
|----------|------|--------|
| Web UI | 8188 | Tailnet only |

**See:** `docs/AI_SERVICES_IMPLEMENTATION.md` for workflow examples.

---

### Open WebUI (Chat Interface)

**File:** `group_vars/homelab.yml`

#### Version Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `OPEN_WEBUI_VERSION` | string | `"latest"` | Open WebUI Docker image version |

#### Storage Paths

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `OPEN_WEBUI_DATA_PATH` | string | `{{ COLD_STORAGE_PATH }}/ai/open-webui/data` | Chat history and settings |

#### Network Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `OPEN_WEBUI_SUBDOMAIN` | string | `"chat"` | Subdomain for NGINX proxy (chat.domain.local) |

#### Service Endpoints

| Endpoint | Port | Access |
|----------|------|--------|
| Web UI | 3000 | Tailnet only |

**See:** `docs/AI_SERVICES_IMPLEMENTATION.md` for usage guide.

---

## Database Management Role

**File:** `group_vars/homelab.yml`

### Overview

The database-management role supports multiple database engines simultaneously. PostgreSQL 16 with pgvector is currently implemented. Future support for SQL Server and Oracle is planned (architecture ready).

---

### PostgreSQL Sub-Role

All PostgreSQL-specific configuration for the database-management role.

#### Version Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `POSTGRESQL_VERSION` | string | `"16-alpine"` | PostgreSQL image tag (ankane/pgvector) |
| `PGADMIN_VERSION` | string | `"latest"` | pgAdmin 4 web UI version |

#### Storage Paths

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `POSTGRESQL_DATA_PATH` | string | `{{ COLD_STORAGE_PATH }}/database/postgresql/data` | Database files (PGDATA) |
| `POSTGRESQL_INIT_PATH` | string | `{{ COLD_STORAGE_PATH }}/database/postgresql/init` | Initialization SQL scripts |
| `POSTGRESQL_CONF_PATH` | string | `{{ COLD_STORAGE_PATH }}/database/postgresql/conf` | postgresql.conf, pg_hba.conf |
| `POSTGRESQL_BACKUP_PATH` | string | `{{ COLD_STORAGE_PATH }}/database/postgresql/backup` | Automated backups |
| `POSTGRESQL_LOGS_PATH` | string | `{{ COLD_STORAGE_PATH }}/database/postgresql/logs` | PostgreSQL logs |
| `PGADMIN_DATA_PATH` | string | `{{ COLD_STORAGE_PATH }}/database/pgadmin/data` | pgAdmin settings |

#### Network Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `POSTGRESQL_PORT` | integer | `5432` | PostgreSQL server port |
| `PGADMIN_PORT` | integer | `5050` | pgAdmin web UI port |
| `PGADMIN_ENABLED` | boolean | `false` | Enable/disable pgAdmin deployment |

#### Performance Tuning

Optimized for 16GB RAM server running multiple services.

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `POSTGRES_SHARED_BUFFERS` | string | `"512MB"` | RAM allocated to PostgreSQL cache (~3% of 16GB) |
| `POSTGRES_EFFECTIVE_CACHE_SIZE` | string | `"4GB"` | OS cache hint (not allocated, ~25% of RAM) |
| `POSTGRES_MAX_CONNECTIONS` | integer | `50` | Maximum concurrent connections |
| `POSTGRES_WORK_MEM` | string | `"8MB"` | Memory per query operation (sort, hash) |
| `POSTGRES_MAINTENANCE_WORK_MEM` | string | `"256MB"` | Memory for VACUUM, CREATE INDEX |

**RAM Usage Estimate:**
- Idle: ~600MB
- Light load: ~800-1000MB
- Heavy load: ~1.5GB (50 connections)

#### Admin User Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `POSTGRES_ADMIN_USER` | string | `"db_admin"` | Admin username (CREATEDB, CREATEROLE privileges) |

**Passwords (in `secrets.yml`):**

| Variable | User | Privileges |
|----------|------|------------|
| `POSTGRES_ROOT_PASSWORD` | `postgres` | SUPERUSER (root) |
| `POSTGRES_ADMIN_PASSWORD` | `db_admin` | CREATEDB, CREATEROLE |

#### Backup Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `POSTGRES_BACKUP_SCHEDULE` | string | `"0 2 * * *"` | Cron schedule (daily at 2 AM) |
| `POSTGRES_BACKUP_RETENTION_DAILY` | integer | `7` | Keep daily backups for 7 days |
| `POSTGRES_BACKUP_RETENTION_WEEKLY` | integer | `30` | Keep weekly backups for 30 days |
| `POSTGRES_BACKUP_RETENTION_MONTHLY` | integer | `365` | Keep monthly backups for 1 year |
| `POSTGRES_BACKUP_ENABLE_ENCRYPTION` | boolean | `false` | Encrypt backups with GPG |
| `POSTGRES_BACKUP_GPG_RECIPIENT` | string | `""` | GPG key ID for encryption |

**Backup Method:** `pg_dump` with gzip compression
**Backup Location:** `{{ POSTGRESQL_BACKUP_PATH }}/daily/`, `weekly/`, `monthly/`

#### SSL/TLS Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `POSTGRES_ENABLE_SSL` | boolean | `false` | Enable SSL/TLS connections |

**Note:** SSL disabled by default since Tailscale provides encrypted transport.

#### Database Definitions

**File:** `group_vars/homelab.yml` or `group_vars/secrets.yml`

Define databases as a list under `POSTGRES_DATABASES`:

```yaml
POSTGRES_DATABASES:
  - name: myapp_db
    owner: myapp_user
    description: "My Application Database"
    encoding: UTF8
    locale: en_US.UTF-8
    connection_limit: 50
    allowed_users: []        # Additional users with full access
    readonly_users: []       # Users with SELECT-only access
    enable_pgvector: true    # Enable vector similarity search
    enable_uuid: true        # Enable UUID generation
    enable_audit_log: false  # Create audit_log table
```

**Fields:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | ✅ | - | Database name |
| `owner` | string | ✅ | - | Database owner (must exist in POSTGRES_DATABASE_USERS) |
| `description` | string | ❌ | `""` | Human-readable description |
| `encoding` | string | ❌ | `UTF8` | Character encoding |
| `locale` | string | ❌ | `en_US.UTF-8` | Locale for sorting/formatting |
| `connection_limit` | integer | ❌ | `-1` | Max connections (-1 = unlimited) |
| `allowed_users` | list | ❌ | `[]` | Users with CONNECT, CREATE, full permissions |
| `readonly_users` | list | ❌ | `[]` | Users with read-only SELECT permissions |
| `enable_pgvector` | boolean | ❌ | `false` | Enable pgvector extension (AI embeddings) |
| `enable_uuid` | boolean | ❌ | `true` | Enable uuid-ossp extension |
| `enable_audit_log` | boolean | ❌ | `false` | Create audit_log table for compliance |

#### User Definitions

**File:** `group_vars/secrets.yml` (recommended) or `group_vars/homelab.yml`

Define users as a list under `POSTGRES_DATABASE_USERS`:

```yaml
POSTGRES_DATABASE_USERS:
  - username: myapp_user
    password: "{{ MYAPP_DB_PASSWORD }}"  # Reference from secrets.yml
    is_superuser: false
    can_create_db: false
    can_create_role: false
    connection_limit: 10
```

**Fields:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `username` | string | ✅ | - | PostgreSQL username |
| `password` | string | ✅ | - | User password (use Vault variable) |
| `is_superuser` | boolean | ❌ | `false` | Grant SUPERUSER privilege |
| `can_create_db` | boolean | ❌ | `false` | Grant CREATEDB privilege |
| `can_create_role` | boolean | ❌ | `false` | Grant CREATEROLE privilege |
| `connection_limit` | integer | ❌ | `-1` | Max simultaneous connections (-1 = unlimited) |

---

### pgAdmin Sub-Role

Web-based PostgreSQL administration tool (optional).

#### Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `PGADMIN_ENABLED` | boolean | `false` | Deploy pgAdmin container |
| `PGADMIN_EMAIL` | string | `"admin@homelab.local"` | pgAdmin login email |
| `PGADMIN_PORT` | integer | `5050` | Web UI port |

**Password (in `secrets.yml`):**

| Variable | Description |
|----------|-------------|
| `PGADMIN_PASSWORD` | pgAdmin login password |

**Access:** `http://{{ TAILSCALE_IP_ADDRESS }}:5050`

---

### Database Internal Variables

**File:** `group_vars/database-management-internal.yml`

⚠️ **DO NOT EDIT** - These are computed automatically from user variables.

#### Docker Images

| Variable | Computed From | Description |
|----------|---------------|-------------|
| `POSTGRESQL_IMAGE` | `POSTGRESQL_VERSION` | `ankane/pgvector:{{ POSTGRESQL_VERSION }}` |
| `PGADMIN_IMAGE` | `PGADMIN_VERSION` | `dpage/pgadmin4:{{ PGADMIN_VERSION }}` |

#### Project Configuration

| Variable | Value | Description |
|----------|-------|-------------|
| `DATABASE_MANAGEMENT_PROJECT_DIR` | `/tmp/database-management` | Temporary build directory |
| `DATABASE_MANAGEMENT_ENV_FILE` | `.env` | Main environment file |
| `DATABASE_MANAGEMENT_POSTGRES_ENV_FILE` | `.postgresql.env` | PostgreSQL bootstrap credentials |
| `DATABASE_MANAGEMENT_PGADMIN_ENV_FILE` | `.pgadmin.env` | pgAdmin credentials |

#### Health Check Configuration

| Variable | Value | Description |
|----------|-------|-------------|
| `DATABASE_MANAGEMENT_HEALTHCHECK_RETRIES` | `12` | Max health check attempts |
| `DATABASE_MANAGEMENT_HEALTHCHECK_DELAY` | `5` | Seconds between attempts |

#### Backup Configuration

| Variable | Computed From | Description |
|----------|---------------|-------------|
| `DATABASE_MANAGEMENT_BACKUP_SCRIPT` | - | `/usr/local/bin/postgresql-backup.sh` |
| `POSTGRES_BACKUP_CRON_MINUTE` | `POSTGRES_BACKUP_SCHEDULE` | Cron minute field |
| `POSTGRES_BACKUP_CRON_HOUR` | `POSTGRES_BACKUP_SCHEDULE` | Cron hour field |
| `POSTGRES_BACKUP_CRON_USER` | - | `root` |

#### Network Configuration

| Variable | Value | Description |
|----------|-------|-------------|
| `DATABASE_MANAGEMENT_INTERNAL_NETWORK` | `self-hosted` | Docker network name |

#### Computed Paths

| Variable | Computed From | Description |
|----------|---------------|-------------|
| `POSTGRESQL_CERTS_PATH` | `POSTGRESQL_CONF_PATH` | `{{ POSTGRESQL_CONF_PATH }}/certs` |

---

## Security Best Practices

### Password Management

1. **Use Ansible Vault** for all passwords:
   ```bash
   ansible-vault edit group_vars/secrets.yml
   ```

2. **Generate Strong Passwords:**
   ```bash
   openssl rand -base64 32
   ```

3. **Rotate Passwords Regularly:**
   - Edit `secrets.yml`
   - Re-run playbook to update
   - Restart applications with new credentials

### Network Security

- **Tailscale VPN:** All services accessible only via Tailnet (100.64.0.0/10)
- **PostgreSQL:** `pg_hba.conf` restricts to Tailnet CIDR, rejects public internet
- **NGINX Proxy Manager:** Admin UI on port 81 (Tailnet only)

### File Permissions

All sensitive files use restricted permissions:
- Configuration: `0640` (owner read/write, group read)
- Data directories: `0700` (owner only)
- Backup encryption: GPG with recipient key

---

## Variable Precedence

Ansible loads variables in this order (last wins):

1. `group_vars/homelab.yml` - User defaults
2. `group_vars/database-management-internal.yml` - Computed values
3. `group_vars/secrets.yml` - Encrypted secrets
4. Command-line: `-e` extra vars (highest priority)

---

## Quick Reference

### View All Variables

```bash
# View unencrypted variables
cat group_vars/homelab.yml
cat group_vars/database-management-internal.yml

# View encrypted secrets
ansible-vault view group_vars/secrets.yml
```

### Edit Configuration

```bash
# Edit user configuration
nano group_vars/homelab.yml

# Edit secrets (requires vault password)
ansible-vault edit group_vars/secrets.yml
```

### Deploy Changes

```bash
# Apply all changes
ansible-playbook server_playbook.yml

# Apply specific role
ansible-playbook server_playbook.yml --tags database

# Check what would change (dry run)
ansible-playbook server_playbook.yml --check
```

---

## Related Documentation

- **[DATABASE_MANAGEMENT_PLAN.md](DATABASE_MANAGEMENT_PLAN.md)** - Database architecture
- **[POSTGRES_DATABASE_MANAGEMENT.md](roles/database-management/docs/POSTGRES_DATABASE_MANAGEMENT.md)** - PostgreSQL usage guide

---

## Development Role

**Purpose:** Install host-level development tools for remote IDE support (VS Code Remote-SSH, JetBrains Gateway)

**File:** `group_vars/development/vars.yml`

**Documentation:** 
- `roles/development/README.md` - Complete implementation guide
- [DEVELOPMENT_QUICK_REFERENCE.md](DEVELOPMENT_QUICK_REFERENCE.md) - Quick command reference

### Version Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `NVM_VERSION` | string | `"0.40.3"` | Node Version Manager version |
| `NODEJS_VERSION` | string | `"22"` | Node.js major version (installed via nvm) |
| `DOTNET_VERSION` | string | `"9.0"` | .NET SDK version |

### Python Sub-Task

**File:** `group_vars/development/vars.yml`

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `PYTHON_PACKAGES` | list | See below | DNF packages to install for Python development |
| `DEVELOPMENT_SKIP_PYTHON` | bool | `false` | Skip Python installation if true |

**Default Python Packages:**
```yaml
PYTHON_PACKAGES:
  - python3
  - python3-pip
  - python3-devel
  - python3-virtualenv
  - python3-setuptools
```

**Verification:**
- Python 3.9+ required
- pip3 functional test
- Import test (`import sys`)
- Symlink `/usr/bin/python3` → `/usr/bin/python3.x`

### Node.js Sub-Task

**File:** `group_vars/development/vars.yml`

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `NVM_VERSION` | string | `"0.40.3"` | nvm version to install |
| `NODEJS_VERSION` | string | `"22"` | Node.js major version |
| `DEVELOPMENT_SKIP_NODEJS` | bool | `false` | Skip Node.js installation if true |

**Installation Method:**
- nvm installed from official script: `https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh`
- Node.js installed via nvm: `nvm install 22`
- PNPM enabled via Corepack: `corepack enable pnpm`

**Installation Locations:**
- nvm: `~/.nvm/`
- Node.js: `~/.nvm/versions/node/v22.x.x/`
- Global packages: `~/.nvm/versions/node/v22.x.x/lib/node_modules/`

**Verification:**
- Node.js v22.x required (>= 18.0.0)
- npm, pnpm functional tests
- Version assertions for all four tools (nvm, node, npm, pnpm)

**Important:** nvm must be sourced in each shell session:
```bash
source ~/.nvm/nvm.sh
```

### .NET Sub-Task

**File:** `group_vars/development/vars.yml`

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `DOTNET_VERSION` | string | `"9.0"` | .NET SDK version |
| `DEVELOPMENT_SKIP_DOTNET` | bool | `false` | Skip .NET installation if true |

**Installation Method:**
- DNF package: `dotnet-sdk-9.0`
- Available in Fedora default repositories
- Telemetry automatically disabled (`DOTNET_CLI_TELEMETRY_OPTOUT=1` in `~/.bashrc`)

**Installation Location:**
- SDK: `/usr/lib64/dotnet/`
- Tools: `/usr/bin/dotnet`

**Verification:**
- .NET 9.0+ required
- Functional test: `dotnet new console` project creation
- Version assertion (>= 9.0.0)

### Control Flags

All development sub-tasks support skip flags for selective installation:

```yaml
# group_vars/development/vars.yml
DEVELOPMENT_SKIP_PYTHON: false   # Set to true to skip Python
DEVELOPMENT_SKIP_NODEJS: false   # Set to true to skip Node.js
DEVELOPMENT_SKIP_DOTNET: false   # Set to true to skip .NET
```

### Deployment Tags

| Tag | Description |
|-----|-------------|
| `development` | Deploy all development tools |
| `dev-tools` | Alias for development |
| `python` | Deploy Python only |
| `nodejs` | Deploy Node.js only |
| `dotnet` | Deploy .NET only |
| `setup` | Includes development (full setup) |

**Examples:**
```bash
# Deploy all development tools
ansible-playbook server_playbook.yml --tags development

# Deploy Python only
ansible-playbook server_playbook.yml --tags python

# Deploy Node.js and .NET (skip Python)
ansible-playbook server_playbook.yml --tags nodejs,dotnet

# Skip development during full deployment
ansible-playbook server_playbook.yml --skip-tags development
```

### IDE Configuration

**VS Code Remote-SSH:**
```json
// .vscode/settings.json
{
  "remote.SSH.defaultExtensions": [
    "ms-python.python",
    "ms-dotnettools.csharp"
  ],
  "python.defaultInterpreterPath": "/usr/bin/python3",
  "dotnet.server.path": "/usr/bin/dotnet"
}
```

**JetBrains Gateway:**
- Python (PyCharm): Automatically detects `/usr/bin/python3`
- .NET (Rider): Automatically detects `/usr/bin/dotnet`
- Node.js (WebStorm/IntelliJ): Configure Node.js interpreter → `~/.nvm/versions/node/v22.x.x/bin/node`

### Quick Reference

**See:** [DEVELOPMENT_QUICK_REFERENCE.md](DEVELOPMENT_QUICK_REFERENCE.md) for:
- Quick deploy commands
- Verification steps
- Common tasks (venv, npm/pnpm, dotnet projects)
- Troubleshooting
- Version management

---
- **[AI_SERVICES_IMPLEMENTATION.md](AI_SERVICES_IMPLEMENTATION.md)** - AI services configuration
- **[DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)** - Full deployment guide
- **[ARCHITECTURE.md](ARCHITECTURE.md)** - Overall system architecture
