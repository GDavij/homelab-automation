# Database Management Role

Ansible role for deploying and managing database engines with PostgreSQL 16 + pgvector currently implemented. Designed for secure, production-ready database infrastructure with automated backups, user segregation, and AI-ready vector search capabilities.

## 📚 Documentation

**Complete documentation is available in the `docs/` folder:**

- **[Environment Variables Reference](../../docs/ENVIRONMENT_VARIABLES.md#database-management-role)** - All variables organized by role and sub-role
- **[PostgreSQL Management Guide](../../docs/POSTGRES_DATABASE_MANAGEMENT.md)** - Complete PostgreSQL usage guide
- **[Database Management Plan](../../docs/DATABASE_MANAGEMENT_PLAN.md)** - Architecture and design
- **[Deployment Guide](../../docs/DEPLOYMENT_GUIDE.md)** - Deployment instructions

## Features

- **Multiple Database Engine Support** (modular architecture)
  - **PostgreSQL 16** with **pgvector extension** (currently implemented)
  - SQL Server support (architecture ready)
  - Oracle Database support (architecture ready)
- **Tailscale-first networking**: Secure VPN-only database access (100.64.0.0/10)
- **pgAdmin via Nginx Proxy**: pgAdmin accessible at `pg.homelab` through Nginx Proxy Manager (internal network only)
- **User segregation**: Superuser → Admin → Application users → Read-only users
- **Automated backups**: Daily/weekly/monthly with optional GPG encryption
- **pgAdmin 4**: Optional web UI for PostgreSQL management (nginx proxy required)
- **Zero-trust security**: pg_hba.conf restricted to Tailscale CIDR
- **Performance tuning**: Optimized for 16GB RAM server with multiple services (512MB shared_buffers)
- **Idempotent SQL**: Safe to re-run initialization scripts
- **Audit logging**: Optional per-database audit trail

## Quick Start

### 1. Configure Variables

**Edit `group_vars/homelab.yml`:**
```yaml
# Storage paths
POSTGRESQL_DATA_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/data"

# Performance tuning (optimized for 16GB RAM)
POSTGRES_SHARED_BUFFERS: "512MB"
POSTGRES_MAX_CONNECTIONS: 50
```

**See:** [Environment Variables Reference](../../docs/ENVIRONMENT_VARIABLES.md#postgresql-sub-role) for all options.

### 2. Set Passwords

**Edit `group_vars/secrets.yml` (ansible-vault):**
```bash
ansible-vault edit group_vars/secrets.yml
```

```yaml
POSTGRES_ROOT_PASSWORD: "your_secure_password"
POSTGRES_ADMIN_PASSWORD: "your_admin_password"
```

### 3. Define Databases (Optional)

**Edit `group_vars/homelab.yml`:**
```yaml
POSTGRES_DATABASE_USERS:
  - username: myapp_user
    password: "{{ MYAPP_DB_PASSWORD }}"
    connection_limit: 10

POSTGRES_DATABASES:
  - name: myapp_db
    owner: myapp_user
    enable_pgvector: true
    allowed_users: []
    readonly_users: []
```

**See:** [PostgreSQL Management Guide](../../docs/POSTGRES_DATABASE_MANAGEMENT.md#how-to-add-a-new-database) for detailed instructions.

### 4. Deploy

```bash
# Deploy database
ansible-playbook server_playbook.yml --tags database

# Deploy PostgreSQL only
ansible-playbook server_playbook.yml --tags postgres
```

## Requirements

### Dependencies
- **Ansible**: 2.10 or higher
- **Collections**: `community.docker` v3.4.0+
- **Roles**: `docker`, `networking` (must run first)

### Target System
- **OS**: Fedora Server (or RHEL-based)
- **Docker**: Docker CE with Docker Compose v2
- **Storage**: `/cold-storage` path configured
- **Tailscale**: Active VPN connection
- **RAM**: 16GB recommended (512MB allocated to PostgreSQL by default)

Install dependencies:
```bash
ansible-galaxy collection install -r requirements.yml
```

## Environment Variables

All variables are organized in `group_vars/` following project standards:

### User-Configurable Variables
**File:** `group_vars/homelab.yml`

| Category | Variables |
|----------|-----------|
| **PostgreSQL Version** | `POSTGRESQL_VERSION`, `PGADMIN_VERSION` |
| **Storage Paths** | `POSTGRESQL_DATA_PATH`, `POSTGRESQL_BACKUP_PATH`, etc. |
| **Network** | `POSTGRESQL_PORT`, `PGADMIN_PORT`, `PGADMIN_ENABLED` |
| **Performance** | `POSTGRES_SHARED_BUFFERS`, `POSTGRES_MAX_CONNECTIONS`, etc. |
| **Backup** | `POSTGRES_BACKUP_SCHEDULE`, retention settings |
| **Databases** | `POSTGRES_DATABASES` (list) |
| **Users** | `POSTGRES_DATABASE_USERS` (list) |

### Secrets
**File:** `group_vars/secrets.yml` (encrypted)

| Variable | Purpose |
|----------|---------|
| `POSTGRES_ROOT_PASSWORD` | PostgreSQL superuser password |
| `POSTGRES_ADMIN_PASSWORD` | db_admin user password |
| `PGADMIN_PASSWORD` | pgAdmin UI password |
| Application user passwords | e.g., `MYAPP_DB_PASSWORD` |

### Internal Variables
**File:** `group_vars/database-management-internal.yml` (auto-computed - DO NOT EDIT)

Contains computed values like Docker image names, cron schedules, and internal paths.

**Complete Reference:** [Environment Variables Documentation](../../docs/ENVIRONMENT_VARIABLES.md#database-management-role)

## Usage Examples

### Access pgAdmin Web Interface

**pgAdmin is accessible via Nginx Proxy Manager at `pg.homelab`:**

1. **Configure Nginx Proxy Manager** (if not already done):
   - Host: `pg.homelab`
   - Forward Hostname: `pgadmin` (container name)
   - Forward Port: `5050`
   - Enable SSL (recommended)

2. **Access pgAdmin**:
   ```
   https://pg.homelab
   ```
   - Login Email: From `group_vars/database-management/pgadmin/vars.yml`
   - Password: Set in `group_vars/database-management/pgadmin/secrets.yml`

3. **Add PostgreSQL Server in pgAdmin**:
   - General Tab:
     - Name: `Homelab PostgreSQL`
   - Connection Tab:
     - Host: `postgresql` (container name on docker network)
     - Port: `5432`
     - Username: From `group_vars/database-management/postgres/vars.yml` (or `postgres`)
     - Password: From `group_vars/database-management/postgres/secrets.yml`

**Note**: pgAdmin is only accessible through the nginx proxy. It is NOT exposed directly on any port for security.

### Connect to Database

**From any Tailnet device:**
```bash
# As admin user
psql "postgresql://dbadmin_autogen:PASSWORD@100.64.0.1:5432/postgres"

# As application user
psql "postgresql://myapp_user:PASSWORD@100.64.0.1:5432/myapp_db"
```

### Add New Database

**See:** [Complete guide with examples](../../docs/POSTGRES_DATABASE_MANAGEMENT.md#complete-example-adding-a-new-application-database)

```bash
# 1. Edit secrets
ansible-vault edit group_vars/secrets.yml

# 2. Edit homelab.yml (add user and database)
nano group_vars/homelab.yml

# 3. Deploy
ansible-playbook server_playbook.yml --tags postgres
```

### Backup Management

```bash
# Manual backup
ansible-playbook server_playbook.yml --tags database-bkp

# View backup logs
ssh homelab-server
cat /cold-storage/database/postgresql/logs/backup.log

# Restore database
gunzip < backup.sql.gz | docker exec -i database-management-postgres-1 psql -U postgres -d mydb
```

**See:** [Backup and Recovery Guide](../../docs/POSTGRES_DATABASE_MANAGEMENT.md#backup-and-recovery)

## Architecture

### Role Structure

```
roles/database-management/
├── tasks/
│   ├── main.yml                    # Database-agnostic orchestration
│   ├── README.md                   # Multi-engine implementation guide
│   └── postgres/                   # PostgreSQL-specific tasks
│       ├── main.yml
│       ├── validate.yml
│       ├── setup-storage.yml
│       ├── deploy-postgresql.yml
│       ├── initialize-databases.yml
│       ├── deploy-pgadmin.yml
│       ├── configure-backups.yml
│       └── verify.yml
├── templates/
│   └── postgres/                   # PostgreSQL-specific templates
│       ├── .env.j2
│       ├── postgresql.conf.j2
│       ├── pg_hba.conf.j2
│       ├── backup-script.sh.j2
│       └── *.sql.j2 (8 SQL init scripts)
├── files/
│   ├── docker-compose.yml
│   └── .dockerignore
├── handlers/main.yml
├── meta/main.yml
└── vars/main.yml (empty - uses group_vars)
```

### Multi-Engine Design

The role is designed to support multiple database engines simultaneously:

- **Current:** PostgreSQL 16 with pgvector (`tasks/postgres/`, `templates/postgres/`)
- **Future:** SQL Server (`tasks/sqlserver/`), Oracle (`tasks/oracle/`)

All implemented engines are deployed when the role is executed.

**See:** `tasks/README.md` for adding new engines.

## Security

### Network Security
- **Tailscale VPN only**: PostgreSQL restricted to 100.64.0.0/10 CIDR
- **pg_hba.conf**: Explicit CIDR whitelist, rejects public internet (0.0.0.0/0)
- **scram-sha-256**: Strong password authentication
- **Docker network**: Internal `self-hosted` network isolation

### Access Control
- **Superuser (postgres)**: Root access, restricted to localhost
- **Admin (db_admin)**: CREATEDB, CREATEROLE privileges
- **Application users**: Database-specific permissions
- **Read-only users**: SELECT-only access for analytics

### Data Protection
- **Automated backups**: Daily/weekly/monthly retention
- **Optional GPG encryption**: Secure offsite backups
- **Audit logging**: Per-database audit trail option

**See:** [Security Best Practices](../../docs/POSTGRES_DATABASE_MANAGEMENT.md#security-best-practices)

## Troubleshooting

### Common Issues

**Can't connect:**
```bash
# Check Tailscale
tailscale status

# Check container
docker ps | grep postgres
docker logs database-management-postgres-1
```

**Permission denied:**
```bash
# Verify user exists
docker exec database-management-postgres-1 psql -U postgres -c "\du"

# Check grants
docker exec database-management-postgres-1 psql -U postgres -d mydb -c "\dp"
```

**See:** [Complete Troubleshooting Guide](../../docs/POSTGRES_DATABASE_MANAGEMENT.md#troubleshooting)

## Performance Tuning

### Default Configuration (16GB RAM Server)

Optimized for light database usage with multiple services:

| Setting | Value | Rationale |
|---------|-------|-----------|
| `shared_buffers` | 512MB | Conservative RAM usage |
| `effective_cache_size` | 4GB | OS cache hint |
| `max_connections` | 50 | Sufficient for multiple apps |
| `work_mem` | 8MB | Per-operation memory |
| `maintenance_work_mem` | 256MB | VACUUM, CREATE INDEX |

**RAM Usage:**
- Idle: ~600MB
- Light load: ~800-1000MB
- Heavy load: ~1.5GB (50 connections)

**See:** [Environment Variables - Performance Tuning](../../docs/ENVIRONMENT_VARIABLES.md#performance-tuning)

## Tags

- `database` - All database operations
- `postgres` - PostgreSQL-specific tasks
- `database-bkp` - Backup configuration only

```bash
# Deploy all database services
ansible-playbook server_playbook.yml --tags database

# PostgreSQL only
ansible-playbook server_playbook.yml --tags postgres

# Backup configuration only
ansible-playbook server_playbook.yml --tags database-bkp
```

## Author

Created as part of the Homelab Ansible Infrastructure project.

## License

MIT


## Requirements

### Dependencies
- **Ansible**: 2.10 or higher
- **Collections**: `community.docker` v3.4.0+
- **Roles**: `docker`, `networking` (Tailscale configured)

### Target System
- **OS**: Fedora Server (or RHEL-based)
- **Docker**: Docker CE with Docker Compose v2
- **Storage**: `/cold-storage` with sufficient space for data + backups
- **Tailscale**: Active node with IP configured

Install collection dependencies:
```bash
ansible-galaxy collection install community.docker
```

## Role Variables

### Required Variables

```yaml
# Must be set in group_vars/secrets.yml
POSTGRES_ROOT_PASSWORD: "change_this_superuser_password"
POSTGRES_ADMIN_PASSWORD: "change_this_admin_password"

# Must be set in group_vars/homelab.yml or inventory
TAILSCALE_IP: "100.64.x.x"  # Your Tailscale node IP
```

### Database Configuration

```yaml
# Define databases with per-database settings
POSTGRES_DATABASES:
  - name: langfuse_db
    owner: langfuse_user
    description: "Langfuse AI observability database"
    encoding: UTF8
    locale: en_US.UTF-8
    connection_limit: 50
    
    # User access control
    allowed_users:
      - analytics_user
    readonly_users:
      - reporting_user
    
    # Extension enablement
    enable_pgvector: true
    enable_uuid: true
    enable_trigram: false
    enable_btree_gin: false
    enable_pg_stat_statements: true
    
    # Optional features
    enable_audit_log: true
    enable_ssl: false
```

### User Configuration

```yaml
# Define application users with passwords
POSTGRES_DATABASE_USERS:
  - username: langfuse_user
    password: "{{ LANGFUSE_DB_PASSWORD }}"  # From secrets.yml
    is_superuser: false
    can_create_db: false
    can_create_role: false
    inherit: true
    login: true
    replication: false
    connection_limit: 10
    
  - username: analytics_user
    password: "{{ ANALYTICS_PASSWORD }}"
    is_superuser: false
    can_create_db: false
    connection_limit: 5
    
  - username: reporting_user
    password: "{{ REPORTING_PASSWORD }}"
    is_superuser: false
    can_create_db: false
    connection_limit: 3
```

### Backup Configuration

```yaml
# Backup schedule and retention
POSTGRES_BACKUP_SCHEDULE: "0 2 * * *"  # Daily at 2 AM
POSTGRES_BACKUP_RETENTION_DAILY: 7
POSTGRES_BACKUP_RETENTION_WEEKLY: 30
POSTGRES_BACKUP_RETENTION_MONTHLY: 365

# Optional GPG encryption for backups
POSTGRES_BACKUP_ENABLE_ENCRYPTION: false
POSTGRES_BACKUP_GPG_RECIPIENT: "your-gpg-key-id"
```

### Performance Tuning

```yaml
# Memory settings (adjust based on server RAM)
POSTGRES_SHARED_BUFFERS: "2GB"          # 25% of RAM
POSTGRES_EFFECTIVE_CACHE_SIZE: "6GB"    # 50-75% of RAM
POSTGRES_WORK_MEM: "16MB"               # Per-operation memory
POSTGRES_MAINTENANCE_WORK_MEM: "512MB"  # VACUUM, CREATE INDEX

# Connection limits
POSTGRES_MAX_CONNECTIONS: 100
```

### Optional Features

```yaml
# pgAdmin web UI
PGADMIN_ENABLED: true
PGADMIN_DEFAULT_EMAIL: "admin@example.com"
PGADMIN_DEFAULT_PASSWORD: "{{ PGADMIN_PASSWORD }}"
PGADMIN_PORT: 5050

# SSL/TLS (requires certificates in POSTGRES_CERTS_DIR)
POSTGRES_ENABLE_SSL: false

# Trusted networks (in addition to Tailscale)
POSTGRES_TRUSTED_NETWORKS:
  - cidr: "192.168.1.0/24"
    comment: "Local network"
    database: "all"
    user: "all"
```

## Example Playbook

```yaml
---
- name: Deploy homelab infrastructure
  hosts: homelab
  become: true
  
  roles:
    - role: common
      tags: [common]
    
    - role: networking
      tags: [networking, tailscale]
    
    - role: docker
      tags: [docker]
    
    - role: database-management
      tags: [database, postgres]
```

## Usage

### Initial Deployment

```bash
# Check mode (dry-run)
ansible-playbook server_playbook.yml --tags database --check

# Deploy database
ansible-playbook server_playbook.yml --tags database

# Verify deployment
ansible-playbook verify_services.yml --tags database
```

### Access Database

**Via psql (from any Tailscale device):**
```bash
# As admin user
psql "postgresql://db_admin:PASSWORD@100.64.x.x:5432/postgres"

# As application user
psql "postgresql://langfuse_user:PASSWORD@100.64.x.x:5432/langfuse_db"

# Verify pgvector extension
SELECT * FROM pg_available_extensions WHERE name = 'vector';
```

**Via pgAdmin (if enabled):**
1. Navigate to `http://100.64.x.x:5050`
2. Login with PGADMIN_DEFAULT_EMAIL and password
3. Add server: Host = `postgresql-db`, Port = `5432`

### Manage Backups

```bash
# View backup logs
ssh homelab-server
cat /cold-storage/database/postgresql/logs/backup.log

# List backups
ls -lh /cold-storage/database/postgresql/backup/daily/
ls -lh /cold-storage/database/postgresql/backup/weekly/
ls -lh /cold-storage/database/postgresql/backup/monthly/

# Restore from backup (unencrypted)
gunzip < backup_file.sql.gz | docker exec -i postgresql-db psql -U postgres -d target_db

# Restore from encrypted backup
gpg --decrypt backup_file.sql.gz.gpg | gunzip | docker exec -i postgresql-db psql -U postgres -d target_db
```

### Add New Database

1. Edit `group_vars/homelab.yml`:
```yaml
POSTGRES_DATABASES:
  # ... existing databases ...
  - name: new_app_db
    owner: new_app_user
    enable_pgvector: true
    allowed_users: []
    readonly_users: []
```

2. Add user to `group_vars/secrets.yml`:
```yaml
POSTGRES_DATABASE_USERS:
  # ... existing users ...
  - username: new_app_user
    password: "secure_random_password"
```

3. Re-run playbook:
```bash
ansible-playbook server_playbook.yml --tags database
```

### Monitor Database

```bash
# View container logs
docker logs postgresql-db --tail 100 --follow

# Check database size
docker exec postgresql-db psql -U postgres -c "\l+"

# View connections
docker exec postgresql-db psql -U postgres -c "SELECT * FROM pg_stat_activity;"

# Query statistics (requires pg_stat_statements)
docker exec postgresql-db psql -U postgres -c "SELECT query, calls, total_time, mean_time FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;"
```

## pgvector Usage Examples

**Create embeddings table:**
```sql
CREATE TABLE embeddings (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    content TEXT NOT NULL,
    embedding VECTOR(1536),  -- OpenAI ada-002 dimension
    metadata JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- HNSW index (fast approximate search)
CREATE INDEX ON embeddings USING hnsw (embedding vector_cosine_ops);
```

**Similarity search:**
```sql
-- Find similar vectors (cosine similarity)
SELECT 
    id, 
    content,
    1 - (embedding <=> '[0.1,0.2,...]'::vector) AS similarity
FROM embeddings
ORDER BY embedding <=> '[0.1,0.2,...]'::vector
LIMIT 10;
```

## Security Considerations

- **Network isolation**: PostgreSQL only accessible via Tailscale VPN
- **Strong authentication**: scram-sha-256 password hashing
- **Least privilege**: Application users have minimal required permissions
- **Backup encryption**: Optional GPG encryption for offsite backups
- **SSL/TLS**: Optional in-transit encryption (VPN already encrypts)
- **Audit logging**: Optional per-database audit trail for compliance

## Troubleshooting

**Connection refused:**
- Verify Tailscale is running: `tailscale status`
- Check PostgreSQL container: `docker ps | grep postgresql`
- View logs: `docker logs postgresql-db`

**Permission denied:**
- Verify user exists: `docker exec postgresql-db psql -U postgres -c "\du"`
- Check database grants: `docker exec postgresql-db psql -U postgres -d your_db -c "\dp"`

**pgvector not available:**
- Verify extension: `SELECT * FROM pg_available_extensions WHERE name = 'vector';`
- Enable if needed: `CREATE EXTENSION IF NOT EXISTS vector;`

**Backup failures:**
- Check disk space: `df -h /cold-storage`
- View backup logs: `cat /cold-storage/database/postgresql/logs/backup.log`
- Test GPG encryption: `echo "test" | gpg --encrypt --recipient YOUR_KEY`

## Files Structure

```
roles/database-management/
├── defaults/main.yml          # Default variables with documentation
├── vars/main.yml              # Internal variables (reserved)
├── meta/main.yml              # Role dependencies
├── handlers/main.yml          # Service restart handlers
├── tasks/
│   ├── main.yml               # Main orchestration (engine-agnostic)
│   ├── validate.yml           # Pre-flight checks
│   ├── setup-storage.yml      # Directory creation
│   ├── configure-backups.yml  # Backup automation
│   ├── verify.yml             # Post-deployment checks
│   ├── README.md              # Task structure documentation
│   └── postgres/              # PostgreSQL implementation
│       ├── main.yml           # PostgreSQL orchestration
│       ├── deploy-postgresql.yml   # Container deployment
│       ├── initialize-databases.yml # SQL initialization
│       └── deploy-pgadmin.yml      # pgAdmin UI
├── files/
│   ├── docker-compose.yml     # PostgreSQL + pgAdmin stack
│   └── .dockerignore          # Build context optimization
└── templates/
    ├── .env.j2                # Docker Compose environment
    ├── .postgresql.env.j2     # PostgreSQL credentials
    ├── .pgadmin.env.j2        # pgAdmin credentials
    ├── postgresql.conf.j2     # PostgreSQL configuration
    ├── pg_hba.conf.j2         # Authentication rules
    ├── backup-script.sh.j2    # Automated backup script
    └── *.sql.j2               # Database initialization scripts
```

## Task Structure

The role uses a modular architecture for supporting multiple database engines simultaneously:

- **Engine-agnostic tasks**: `validate.yml`, `setup-storage.yml`, `configure-backups.yml`, `verify.yml`
- **Engine-specific tasks**: Located in `tasks/<engine>/` subfolders (e.g., `tasks/postgres/`)
- **Deployment**: All implemented engine tasks are executed when the role runs

See `tasks/README.md` for instructions on adding new database engines.

## Author

Created as part of the Homelab Ansible Infrastructure project.

## License

MIT
