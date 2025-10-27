# Database Management Role - Implementation Plan
**Role Name:** `database-management`  
**Purpose:** Centralized PostgreSQL database service for all homelab applications  
**Version:** 1.0.0  
**Date:** October 24, 2025

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Architecture Design](#architecture-design)
3. [Role Structure](#role-structure)
4. [PostgreSQL Configuration](#postgresql-configuration)
5. [Security Considerations](#security-considerations)
6. [Database Management Features](#database-management-features)
7. [Integration with Existing Services](#integration-with-existing-services)
8. [Backup & Recovery Strategy](#backup--recovery-strategy)
9. [Implementation Steps](#implementation-steps)
10. [Testing & Validation](#testing--validation)
11. [Future Enhancements](#future-enhancements)

---

## Overview

### Purpose
Create a centralized PostgreSQL database service that:
- Provides database hosting for all homelab services
- Implements proper security and access control
- Includes automated backup and recovery
- Supports multiple databases and users
- Integrates with existing zero-trust architecture
- Follows established role patterns

### Benefits
- ✅ **Centralized Management** - Single PostgreSQL service for every app
- ✅ **Resource Efficiency** - Shared instance with role-based isolation
- ✅ **Consistent Backups** - Automated, encrypted, and verified backups
- ✅ **Secure Tailnet Access** - Bound to Tailscale IP with optional TLS
- ✅ **AI Ready** - Built-in pgvector extension for embeddings
- ✅ **Scalability** - Simple workflows for new databases and users

### Current Services That Will Use It
1. **Langfuse** (planned) - LLM observability platform
2. **Open WebUI** (migration) - Currently uses SQLite, can migrate to PostgreSQL
3. **Future Services** - Any service requiring relational database

---

## Architecture Design

### Network Architecture

```
┌──────────────────────────────────────────────────────────────┐
│          Tailscale VPN Layer (Remote Access)                 │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  PostgreSQL Accessible on Tailnet                      │ │
│  │  ${TAILSCALE_IP}:5432 (Secure, VPN-only)              │ │
│  │                                                         │ │
│  │  Access Control:                                        │ │
│  │  ├─ Admin users (full access)                          │ │
│  │  ├─ Application users (database-specific)              │ │
│  │  └─ Read-only users (monitoring/reporting)             │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│               Docker Network: self-hosted                    │
│                                                              │
│  ┌────────────────┐         ┌──────────────────────┐       │
│  │   PostgreSQL   │◄────────│  Application Layer   │       │
│  │   Container    │         │                      │       │
│  │                │         │  ├─ Langfuse         │       │
│  │  Port: 5432    │         │  ├─ Open WebUI       │       │
│  │  Bound to:     │         │  └─ Future Apps      │       │
│  │  ${TAILSCALE}  │         │                      │       │
│  │                │         │  Connections:        │       │
│  │  Extensions:   │         │  ├─ Internal:        │       │
│  │  ├─ pgvector   │         │  │  postgresql:5432 │       │
│  │  ├─ pg_trgm    │         │  └─ External:        │       │
│  │  └─ uuid-ossp  │         │     ${TAILSCALE}     │       │
│  │                │         │                      │       │
│  │  Volumes:      │         │                      │       │
│  │  ├─ Data       │         │                      │       │
│  │  ├─ Backups    │         │                      │       │
│  │  └─ Init SQL   │         │                      │       │
│  └────────────────┘         └──────────────────────┘       │
│         ▲                                                    │
│         │ (Admin access: Tailscale + pgAdmin)               │
│         │                                                    │
│  ┌────────────────┐                                         │
│  │   pgAdmin      │  Optional Web UI                        │
│  │   :5050        │  (via NGINX proxy)                      │
│  └────────────────┘                                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘

Security Model:
├─ ✅ PostgreSQL exposed on Tailscale IP (VPN-only access)
├─ ✅ Role-based access control (admin, app users, read-only)
├─ ✅ SSL/TLS encryption (optional, easy to enable)
├─ ✅ Password authentication with strong passwords (Vault)
├─ ✅ pgvector extension for AI embeddings
└─ ✅ Optional pgAdmin for administration (proxied via NGINX)
```

### Storage Architecture

```
/cold-storage/
└── database/
    ├── postgresql/
    │   ├── data/                    # PostgreSQL data directory
    │   │   ├── base/               # Database files
    │   │   ├── global/             # Cluster-wide tables
    │   │   ├── pg_wal/             # Write-Ahead Logs
    │   │   └── pg_stat/            # Statistics
    │   │
    │   ├── backups/                # Automated backups
    │   │   ├── daily/              # Daily backups (7 days)
    │   │   ├── weekly/             # Weekly backups (4 weeks)
    │   │   └── monthly/            # Monthly backups (3 months)
    │   │
    │   ├── init/                   # Initialization SQL scripts
    │   │   ├── 01-create-users.sql
    │   │   ├── 02-create-databases.sql
    │   │   └── 03-grant-permissions.sql
    │   │
    │   └── logs/                   # PostgreSQL logs
    │       └── postgresql.log
    │
    └── pgadmin/                    # pgAdmin data (optional)
        └── data/

Estimated Storage:
├─ PostgreSQL data: 5-50GB (depends on usage)
├─ Backups: 10-100GB (depends on retention)
└─ Logs: 1-5GB
```

---

## Role Structure

### Directory Layout

```
roles/database-management/
├── README.md                        # Complete role documentation
│
├── defaults/
│   └── main.yml                    # Default variables
│
├── vars/
│   └── main.yml                    # Role-specific variables
│
├── tasks/
│   ├── main.yml                    # Main task orchestration
│   ├── validate.yml                # Pre-deployment validation
│   ├── setup-storage.yml           # Create storage directories
│   ├── deploy-postgresql.yml       # Deploy PostgreSQL container
│   ├── initialize-databases.yml    # Create databases and users
│   ├── deploy-pgadmin.yml          # Deploy pgAdmin (optional)
│   ├── configure-backups.yml       # Setup backup automation
│   └── verify.yml                  # Post-deployment verification
│
├── templates/
│   ├── .postgresql.env.j2          # PostgreSQL environment variables
│   ├── .pgadmin.env.j2             # pgAdmin environment variables
│   ├── 01-create-users.sql.j2      # User creation script
│   ├── 02-create-databases.sql.j2  # Database creation script
│   ├── 03-grant-permissions.sql.j2 # Permission grants
│   ├── postgresql.conf.j2          # PostgreSQL configuration
│   ├── pg_hba.conf.j2              # Host-based authentication
│   └── backup-script.sh.j2         # Automated backup script
│
├── files/
│   ├── docker-compose.yml          # PostgreSQL + pgAdmin compose
│   └── .dockerignore
│
├── handlers/
│   └── main.yml                    # Service restart handlers
│
└── meta/
    └── main.yml                    # Role dependencies
```

---

## PostgreSQL Configuration

### Docker Compose Configuration

```yaml
# roles/database-management/files/docker-compose.yml

services:
  postgresql:
    # Using official PostgreSQL image with pgvector support
    image: ankane/pgvector:latest
    container_name: postgresql
    restart: unless-stopped
    
    environment:
      # Core Configuration
      POSTGRES_PASSWORD: ${POSTGRES_ROOT_PASSWORD}
      POSTGRES_USER: postgres
      POSTGRES_DB: postgres
      
      # Performance Tuning
      POSTGRES_SHARED_BUFFERS: ${POSTGRES_SHARED_BUFFERS:-256MB}
      POSTGRES_EFFECTIVE_CACHE_SIZE: ${POSTGRES_EFFECTIVE_CACHE_SIZE:-1GB}
      POSTGRES_MAX_CONNECTIONS: ${POSTGRES_MAX_CONNECTIONS:-100}
      POSTGRES_WORK_MEM: ${POSTGRES_WORK_MEM:-4MB}
      POSTGRES_MAINTENANCE_WORK_MEM: ${POSTGRES_MAINTENANCE_WORK_MEM:-64MB}
      
      # Logging
      POSTGRES_LOG_STATEMENT: ${POSTGRES_LOG_STATEMENT:-all}
      POSTGRES_LOG_DURATION: ${POSTGRES_LOG_DURATION:-on}
      
      # SSL/TLS Configuration (optional)
      POSTGRES_SSL_MODE: ${POSTGRES_SSL_MODE:-prefer}
    
    volumes:
      # Data persistence
      - ${POSTGRESQL_DATA_PATH}:/var/lib/postgresql/data
      
      # Initialization scripts (run on first start)
      - ${POSTGRESQL_INIT_PATH}:/docker-entrypoint-initdb.d
      
      # Custom configuration
      - ${POSTGRESQL_CONF_PATH}/postgresql.conf:/etc/postgresql/postgresql.conf
      - ${POSTGRESQL_CONF_PATH}/pg_hba.conf:/etc/postgresql/pg_hba.conf
      
      # SSL certificates (if encryption enabled)
      - ${POSTGRESQL_CONF_PATH}/certs:/etc/postgresql/certs
      
      # Backups
      - ${POSTGRESQL_BACKUP_PATH}:/backups
      
      # Logs
      - ${POSTGRESQL_LOG_PATH}:/var/log/postgresql
    
    # EXPOSED on Tailscale IP for remote access
    ports:
      - "${TAILSCALE_IP_ADDRESS}:5432:5432"
    
    # Also available internally via Docker network
    networks:
      - self-hosted
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    
    # Security
    security_opt:
      - no-new-privileges:true
    
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin
    restart: unless-stopped
    
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_PASSWORD}
      PGADMIN_LISTEN_PORT: 5050
      PGADMIN_CONFIG_SERVER_MODE: 'True'
      PGADMIN_CONFIG_MASTER_PASSWORD_REQUIRED: 'True'
    
    volumes:
      - ${PGADMIN_DATA_PATH}:/var/lib/pgadmin
    
    # Internal only
    expose:
      - "5050"
    
    networks:
      - self-hosted
    
    depends_on:
      postgresql:
        condition: service_healthy
    
    # Optional - can be disabled via variable
    profiles:
      - ${PGADMIN_ENABLED:-enabled}

networks:
  self-hosted:
    external: true
```

### Environment Variables Template

```jinja2
# roles/database-management/templates/.postgresql.env.j2

# PostgreSQL Root Credentials
POSTGRES_ROOT_PASSWORD={{ POSTGRES_ROOT_PASSWORD }}

# Storage Paths
POSTGRESQL_DATA_PATH={{ POSTGRESQL_DATA_PATH }}
POSTGRESQL_INIT_PATH={{ POSTGRESQL_INIT_PATH }}
POSTGRESQL_CONF_PATH={{ POSTGRESQL_CONF_PATH }}
POSTGRESQL_BACKUP_PATH={{ POSTGRESQL_BACKUP_PATH }}
POSTGRESQL_LOG_PATH={{ POSTGRESQL_LOG_PATH }}

# Performance Settings
POSTGRES_SHARED_BUFFERS={{ POSTGRES_SHARED_BUFFERS | default('256MB') }}
POSTGRES_EFFECTIVE_CACHE_SIZE={{ POSTGRES_EFFECTIVE_CACHE_SIZE | default('1GB') }}
POSTGRES_MAX_CONNECTIONS={{ POSTGRES_MAX_CONNECTIONS | default('100') }}

# Logging Settings
POSTGRES_LOG_STATEMENT={{ POSTGRES_LOG_STATEMENT | default('all') }}
POSTGRES_LOG_DURATION={{ POSTGRES_LOG_DURATION | default('on') }}

# pgAdmin Configuration
PGADMIN_EMAIL={{ PGADMIN_EMAIL }}
PGADMIN_PASSWORD={{ PGADMIN_PASSWORD }}
PGADMIN_DATA_PATH={{ PGADMIN_DATA_PATH }}
PGADMIN_ENABLED={{ PGADMIN_ENABLED | default('enabled') }}
```

### Database Initialization Scripts

```sql
-- roles/database-management/templates/01-enable-extensions.sql.j2

-- =============================================================================
-- EXTENSION INSTALLATION
-- =============================================================================

-- Enable pgvector extension for AI embeddings (vector similarity search)
CREATE EXTENSION IF NOT EXISTS vector;

-- Enable other useful extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";           -- UUID generation
CREATE EXTENSION IF NOT EXISTS "pg_trgm";             -- Trigram matching for text search
CREATE EXTENSION IF NOT EXISTS "btree_gin";           -- GIN index support for btree types
CREATE EXTENSION IF NOT EXISTS "btree_gist";          -- GIST index support for btree types
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";  -- Query statistics

-- Log successful extension installation
SELECT 'Extensions installed: vector, uuid-ossp, pg_trgm, btree_gin, btree_gist, pg_stat_statements' AS status;
```

```sql
-- roles/database-management/templates/02-create-admin-role.sql.j2

-- =============================================================================
-- ADMIN ROLE CREATION (separate from postgres superuser)
-- =============================================================================

-- Create dedicated admin role for database management
CREATE ROLE db_admin WITH
    LOGIN
    PASSWORD '{{ postgres_admin_password }}'
    CREATEDB
    CREATEROLE
    REPLICATION
    BYPASSRLS
    CONNECTION LIMIT -1;

COMMENT ON ROLE db_admin IS 'Administrative role for database management operations';

-- Grant necessary privileges to admin
GRANT ALL PRIVILEGES ON DATABASE postgres TO db_admin;
GRANT ALL PRIVILEGES ON SCHEMA public TO db_admin;

SELECT 'Admin role db_admin created successfully' AS status;
```

```sql
-- roles/database-management/templates/03-create-users.sql.j2

-- =============================================================================
-- APPLICATION-SPECIFIC USER CREATION
-- =============================================================================

{% for db_user in POSTGRES_DATABASE_USERS %}
-- Create user: {{ db_user.username }}
CREATE ROLE {{ db_user.username }} WITH
    LOGIN
    PASSWORD '{{ db_user.password }}'
    {% if db_user.createdb | default(false) %}CREATEDB{% else %}NOCREATEDB{% endif %}
    NOCREATEROLE
    NOREPLICATION
    NOBYPASSRLS
    CONNECTION LIMIT {{ db_user.connection_limit | default(-1) }};

COMMENT ON ROLE {{ db_user.username }} IS '{{ db_user.description | default("Application user") }}';

{% endfor %}

SELECT 'Application users created successfully' AS status;
```

```sql
-- roles/database-management/templates/04-create-databases.sql.j2

-- =============================================================================
-- DATABASE CREATION FOR EACH APPLICATION
-- =============================================================================

{% for database in POSTGRES_DATABASES %}
-- Create database: {{ database.name }}
CREATE DATABASE {{ database.name }}
    WITH 
    OWNER = {{ database.owner }}
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.utf8'
    LC_CTYPE = 'en_US.utf8'
    TEMPLATE = template0
    CONNECTION LIMIT {{ database.connection_limit | default(-1) }};

COMMENT ON DATABASE {{ database.name }} IS '{{ database.description | default("Application database") }}';

{% endfor %}

SELECT 'Application databases created successfully' AS status;
```

```sql
-- roles/database-management/templates/05-setup-database-extensions.sql.j2

-- =============================================================================
-- ENABLE EXTENSIONS IN EACH DATABASE
-- =============================================================================

{% for database in POSTGRES_DATABASES %}
-- Connect to database: {{ database.name }}
\c {{ database.name }}

-- Enable pgvector in this database (for AI embeddings)
CREATE EXTENSION IF NOT EXISTS vector;

-- Enable other extensions as configured
{% if database.enable_uuid | default(true) %}
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
{% endif %}

{% if database.enable_trgm | default(false) %}
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
{% endif %}

{% if database.enable_pg_stat_statements | default(false) %}
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";
{% endif %}

-- Create schema if specified
{% if database.schema is defined %}
CREATE SCHEMA IF NOT EXISTS {{ database.schema }};
ALTER SCHEMA {{ database.schema }} OWNER TO {{ database.owner }};
{% endif %}

SELECT 'Extensions enabled in {{ database.name }}' AS status;

{% endfor %}

-- Return to postgres database
\c postgres
```

```sql
-- roles/database-management/templates/06-grant-permissions.sql.j2

-- =============================================================================
-- USER PERMISSIONS AND SEGREGATION
-- =============================================================================

{% for database in POSTGRES_DATABASES %}
\c {{ database.name }}

-- Grant ownership privileges to database owner
GRANT ALL PRIVILEGES ON DATABASE {{ database.name }} TO {{ database.owner }};
GRANT ALL PRIVILEGES ON SCHEMA public TO {{ database.owner }};

{% if database.schema is defined %}
GRANT ALL PRIVILEGES ON SCHEMA {{ database.schema }} TO {{ database.owner }};
{% endif %}

-- Set default privileges for future objects
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO {{ database.owner }};
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO {{ database.owner }};
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON FUNCTIONS TO {{ database.owner }};

{% if database.schema is defined %}
ALTER DEFAULT PRIVILEGES IN SCHEMA {{ database.schema }} GRANT ALL ON TABLES TO {{ database.owner }};
ALTER DEFAULT PRIVILEGES IN SCHEMA {{ database.schema }} GRANT ALL ON SEQUENCES TO {{ database.owner }};
ALTER DEFAULT PRIVILEGES IN SCHEMA {{ database.schema }} GRANT ALL ON FUNCTIONS TO {{ database.owner }};
{% endif %}

-- Grant privileges to allowed users
{% for user in database.allowed_users | default([]) %}
GRANT CONNECT ON DATABASE {{ database.name }} TO {{ user }};
GRANT USAGE ON SCHEMA public TO {{ user }};
-- Full access users (read/write)
{% if database.readonly_users is not defined or user not in database.readonly_users %}
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO {{ user }};
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO {{ user }};
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO {{ user }};
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO {{ user }};
{% endif %}
{% endfor %}

-- Grant read-only access to specific users
{% if database.readonly_users is defined %}
{% for user in database.readonly_users %}
GRANT CONNECT ON DATABASE {{ database.name }} TO {{ user }};
GRANT USAGE ON SCHEMA public TO {{ user }};
GRANT SELECT ON ALL TABLES IN SCHEMA public TO {{ user }};
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO {{ user }};
{% endfor %}
{% endif %}

SELECT 'Permissions granted for {{ database.name }}' AS status;

{% endfor %}

-- Return to postgres database
\c postgres
```

```sql
-- roles/database-management/templates/07-pgvector-examples.sql.j2

-- =============================================================================
-- PGVECTOR USAGE EXAMPLES (for AI applications)
-- =============================================================================

-- Example embeddings table structure for AI vector search
-- This can be used by Langfuse, RAG applications, or custom AI services
-- Uncomment and customize as needed for your AI workloads

/*
-- Connect to your AI application database
\c your_ai_database

-- Create embeddings table
CREATE TABLE IF NOT EXISTS embeddings (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    content TEXT NOT NULL,
    embedding VECTOR(1536),  -- OpenAI ada-002 embedding size (adjust as needed)
    metadata JSONB,
    source TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

COMMENT ON TABLE embeddings IS 'Vector embeddings for AI similarity search';
COMMENT ON COLUMN embeddings.embedding IS 'Vector dimension: 1536 (OpenAI ada-002). Adjust for other models.';

-- Create indexes for fast vector similarity search
CREATE INDEX ON embeddings USING hnsw (embedding vector_cosine_ops);    -- HNSW for cosine similarity
CREATE INDEX ON embeddings USING ivfflat (embedding vector_l2_ops);     -- IVFFlat for L2 distance
CREATE INDEX ON embeddings USING gin (metadata jsonb_path_ops);         -- GIN for metadata queries

-- Example vector search queries:

-- 1. Find similar vectors using cosine similarity
-- SELECT 
--     id, 
--     content,
--     1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
-- FROM embeddings
-- ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
-- LIMIT 10;

-- 2. Find similar vectors using L2 distance
-- SELECT 
--     id,
--     content,
--     embedding <-> '[0.1, 0.2, ...]'::vector AS distance
-- FROM embeddings
-- ORDER BY embedding <-> '[0.1, 0.2, ...]'::vector
-- LIMIT 10;

-- 3. Combine vector search with metadata filtering
-- SELECT 
--     id,
--     content,
--     metadata,
--     1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
-- FROM embeddings
-- WHERE metadata->>'category' = 'documentation'
-- ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
-- LIMIT 10;

-- Vector similarity operators:
-- <-> : L2 distance (Euclidean)
-- <#> : Inner product (dot product)
-- <=> : Cosine distance (1 - cosine similarity)

*/

SELECT 'pgvector examples documented in comments' AS status;
```

```sql
-- roles/database-management/templates/08-audit-log.sql.j2

-- =============================================================================
-- AUDIT LOG SETUP (optional but recommended)
-- =============================================================================

-- Create audit log table for tracking database operations
CREATE TABLE IF NOT EXISTS audit_log (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    username TEXT,
    database_name TEXT,
    action TEXT,
    object_type TEXT,
    object_name TEXT,
    details JSONB,
    client_addr INET,
    application_name TEXT
);

COMMENT ON TABLE audit_log IS 'Audit log for tracking database operations';

CREATE INDEX idx_audit_log_timestamp ON audit_log (timestamp DESC);
CREATE INDEX idx_audit_log_username ON audit_log (username);
CREATE INDEX idx_audit_log_database ON audit_log (database_name);
CREATE INDEX idx_audit_log_action ON audit_log (action);

-- Create audit logging function
CREATE OR REPLACE FUNCTION log_database_operation()
RETURNS event_trigger AS $$
BEGIN
    INSERT INTO audit_log (username, database_name, action, object_type, object_name, details, client_addr, application_name)
    VALUES (
        session_user,
        current_database(),
        tg_tag,
        tg_event,
        '',
        jsonb_build_object('tag', tg_tag, 'event', tg_event),
        inet_client_addr(),
        current_setting('application_name', true)
    );
END;
$$ LANGUAGE plpgsql;

-- Create event triggers for DDL operations (optional - can be resource intensive)
-- CREATE EVENT TRIGGER log_ddl_operations ON ddl_command_end
--     EXECUTE FUNCTION log_database_operation();

SELECT 'Audit log infrastructure created successfully' AS status;
```

---

## Security Considerations

### Security Features

```yaml
Security Layers:

Layer 1: Network Security
├─ PostgreSQL port 5432 EXPOSED on Tailscale IP ONLY
├─ ${TAILSCALE_IP}:5432 binding (not 0.0.0.0)
├─ Tailscale authentication required for external access
├─ Internal applications connect via Docker network (postgresql:5432)
├─ Firewalld rules restrict access to Tailnet only
└─ No public internet exposure

Layer 2: Authentication & User Segregation
├─ postgres superuser (emergency use only, restricted)
├─ db_admin role (dedicated admin, separate from superuser)
├─ Per-application database users (isolated credentials)
├─ Strong passwords (generated via Ansible Vault)
├─ Host-based authentication (pg_hba.conf)
├─ Connection limits per user
└─ Optional SSL/TLS connections (configurable)

Layer 3: Authorization & Role-Based Access Control
├─ Database-level permissions (CONNECT grants)
├─ Schema-level permissions (USAGE, ALL)
├─ Table-level permissions (SELECT, INSERT, UPDATE, DELETE)
├─ Read-only users for analytics/reporting
├─ Default privileges for future objects
└─ No cross-database access (except for admin)

Layer 4: Data Protection & Encryption (Optional but Easy)
├─ SSL/TLS connection encryption (client-server)
│  └─ Configurable via POSTGRES_SSL_MODE variable
├─ At-rest encryption (host-level LUKS/dm-crypt)
│  └─ Applied to /cold-storage mount point
├─ Backup encryption (GPG/age encryption of dumps)
│  └─ Automated in backup scripts
└─ Secret management via Ansible Vault
   └─ All passwords encrypted in secrets.yml

Layer 5: Audit & Logging
├─ Connection logging (all connections tracked)
├─ Statement logging (all queries logged)
├─ Duration logging (slow query detection)
├─ Error logging (troubleshooting)
├─ Audit table (DDL/DML operations)
└─ Log rotation and retention

Layer 6: Backup & Recovery
├─ Automated daily backups (Ansible cron)
├─ Backup encryption (GPG)
├─ Multiple retention windows (daily/weekly/monthly)
├─ Point-in-time recovery capability
├─ Backup verification (automated restore tests)
└─ Off-site backup support (optional)

Layer 7: Container Security
├─ Non-root user (postgres user)
├─ Read-only root filesystem (where possible)
├─ No new privileges (security_opt)
├─ Resource limits (CPU, memory)
├─ Security scanning of images
└─ Minimal attack surface (Alpine base)

Layer 8: Extension Security
├─ pgvector extension (trusted, community-vetted)
├─ Limited extension installation (admin only)
├─ Extension updates via container image updates
└─ Extension audit trail
```

### Encryption Configuration (Optional but Easy to Enable)

#### Connection Encryption (SSL/TLS)

PostgreSQL supports SSL/TLS encryption for client-server connections. This is **optional** but can be enabled easily:

```yaml
# group_vars/homelab.yml - Enable SSL/TLS
POSTGRES_SSL_MODE: "require"  # Options: disable, allow, prefer, require, verify-ca, verify-full

# When enabled, generate certificates:
# - Server certificate (postgresql.crt)
# - Server private key (postgresql.key)
# - CA certificate (root.crt) for client verification
```

**Configuration Steps:**
1. Generate SSL certificates (self-signed or Let's Encrypt)
2. Place certificates in `{{ POSTGRESQL_CONF_PATH }}/certs/`
3. Set `POSTGRES_SSL_MODE: "require"` in group_vars
4. Update pg_hba.conf to enforce SSL: `hostssl` instead of `host`
5. Clients connect with `sslmode=require` in connection string

#### At-Rest Encryption (Storage Level)

Data at-rest encryption protects database files on disk. This is **optional** and configured at the host level:

**Option 1: LUKS/dm-crypt (Full Disk Encryption)**
```bash
# Encrypt the /cold-storage partition
sudo cryptsetup luksFormat /dev/sdX
sudo cryptsetup luksOpen /dev/sdX cold-storage-encrypted
sudo mkfs.ext4 /dev/mapper/cold-storage-encrypted
sudo mount /dev/mapper/cold-storage-encrypted /cold-storage
```

**Option 2: eCryptfs (Directory-Level Encryption)**
```bash
# Encrypt specific database directories
sudo mount -t ecryptfs /cold-storage/database /cold-storage/database
```

**Option 3: Transparent Data Encryption (TDE)**
PostgreSQL 16+ supports transparent data encryption via extensions (e.g., `pgcrypto`, `pg_tde`). This is **advanced** and requires careful planning.

#### Backup Encryption (Always Recommended)

Backup encryption protects database dumps from unauthorized access:

```bash
# Encrypt backups with GPG (automated in backup script)
pg_dump dbname | gzip | gpg --encrypt --recipient admin@example.com > backup.sql.gz.gpg

# Decrypt backup for restore
gpg --decrypt backup.sql.gz.gpg | gunzip | psql dbname
```

**Configuration:**
```yaml
# group_vars/homelab.yml
POSTGRES_BACKUP_ENCRYPTION: true
POSTGRES_BACKUP_GPG_KEY: "admin@example.com"  # GPG key for encryption
```

### Variables Configuration

```yaml
# group_vars/homelab.yml (public variables)

# PostgreSQL Version (using pgvector-enabled image)
POSTGRESQL_IMAGE: "ankane/pgvector:latest"  # PostgreSQL 16 + pgvector
PGADMIN_VERSION: "latest"

# Network Configuration
TAILSCALE_IP_ADDRESS: "{{ TAILSCALE_IP }}"  # Bound to Tailscale interface

# Storage Paths
POSTGRESQL_DATA_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/data"
POSTGRESQL_INIT_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/init"
POSTGRESQL_CONF_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/conf"
POSTGRESQL_BACKUP_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/backups"
POSTGRESQL_LOG_PATH: "{{ COLD_STORAGE_PATH }}/database/postgresql/logs"
PGADMIN_DATA_PATH: "{{ COLD_STORAGE_PATH }}/database/pgadmin/data"

# Performance Settings
POSTGRES_SHARED_BUFFERS: "512MB"
POSTGRES_EFFECTIVE_CACHE_SIZE: "2GB"
POSTGRES_MAX_CONNECTIONS: "100"
POSTGRES_WORK_MEM: "8MB"
POSTGRES_MAINTENANCE_WORK_MEM: "128MB"

# Logging
POSTGRES_LOG_STATEMENT: "all"
POSTGRES_LOG_DURATION: "on"

# SSL/TLS Configuration (optional - set to "disable" to skip encryption)
POSTGRES_SSL_MODE: "prefer"  # Options: disable, allow, prefer, require, verify-ca, verify-full

# Backup Configuration
POSTGRES_BACKUP_RETENTION_DAYS: 7
POSTGRES_BACKUP_RETENTION_WEEKS: 4
POSTGRES_BACKUP_RETENTION_MONTHS: 3
POSTGRES_BACKUP_ENCRYPTION: true
POSTGRES_BACKUP_GPG_KEY: "{{ ADMIN_EMAIL }}"

# pgAdmin
PGADMIN_ENABLED: "enabled"  # or "disabled"
PGADMIN_SUBDOMAIN: "pgadmin"

# Database Definitions (example with pgvector-enabled databases)
POSTGRES_DATABASES:
  - name: "langfuse"
    owner: "langfuse_user"
    description: "Langfuse LLM observability database"
    connection_limit: 50
    enable_uuid: true
    enable_trgm: false
    enable_pg_stat_statements: true
    allowed_users:
      - "langfuse_user"
    readonly_users: []
  
  - name: "openwebui"
    owner: "openwebui_user"
    description: "Open WebUI chat history and embeddings"
    connection_limit: 30
    enable_uuid: true
    enable_trgm: true
    enable_pg_stat_statements: false
    allowed_users:
      - "openwebui_user"
    readonly_users: []
  
  # Future databases (example with vector embeddings)
  - name: "rag_system"
    owner: "rag_user"
    description: "RAG system with document embeddings"
    connection_limit: 20
    enable_uuid: true
    enable_trgm: true
    enable_pg_stat_statements: true
    allowed_users:
      - "rag_user"
      - "search_api_user"
    readonly_users:
      - "analytics_user"
```

```yaml
# group_vars/secrets.yml (encrypted with Ansible Vault)

# PostgreSQL Root Password (superuser - emergency use only)
POSTGRES_ROOT_PASSWORD: "{{ vault_postgres_root_password }}"

# PostgreSQL Admin Password (db_admin role)
POSTGRES_ADMIN_PASSWORD: "{{ vault_postgres_admin_password }}"

# Application Database Users
POSTGRES_DATABASE_USERS:
  - username: "langfuse_user"
    password: "{{ vault_langfuse_db_password }}"
    description: "Langfuse application user"
    connection_limit: 50
    createdb: false
  
  - username: "openwebui_user"
    password: "{{ vault_openwebui_db_password }}"
  
  - username: "monitoring_user"
    password: "{{ vault_monitoring_db_password }}"
  
  - username: "grafana_readonly"
    password: "{{ vault_grafana_readonly_password }}"

# pgAdmin Credentials
PGADMIN_EMAIL: "{{ vault_pgadmin_email }}"
PGADMIN_PASSWORD: "{{ vault_pgadmin_password }}"
```

---

## Database Management Features

### Automated Backup Script

```bash
# roles/database-management/templates/backup-script.sh.j2
#!/bin/bash
# PostgreSQL Automated Backup Script

set -euo pipefail

# Configuration
BACKUP_DIR="{{ POSTGRESQL_BACKUP_PATH }}"
CONTAINER_NAME="postgresql"
POSTGRES_USER="postgres"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
DATE=$(date +"%Y%m%d")

# Retention policies
DAILY_RETENTION={{ POSTGRES_BACKUP_RETENTION_DAYS }}
WEEKLY_RETENTION={{ POSTGRES_BACKUP_RETENTION_WEEKS }}
MONTHLY_RETENTION={{ POSTGRES_BACKUP_RETENTION_MONTHS }}

# Logging
LOG_FILE="${BACKUP_DIR}/backup.log"
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "${LOG_FILE}"
}

# Create backup directories
mkdir -p "${BACKUP_DIR}"/{daily,weekly,monthly}

# Determine backup type
DAY_OF_WEEK=$(date +%u)  # 1-7 (Monday-Sunday)
DAY_OF_MONTH=$(date +%d)

if [ "$DAY_OF_MONTH" = "01" ]; then
    BACKUP_TYPE="monthly"
elif [ "$DAY_OF_WEEK" = "7" ]; then
    BACKUP_TYPE="weekly"
else
    BACKUP_TYPE="daily"
fi

log "Starting ${BACKUP_TYPE} backup..."

# Backup all databases
DATABASES=$(docker exec ${CONTAINER_NAME} psql -U ${POSTGRES_USER} -t -c "SELECT datname FROM pg_database WHERE datistemplate = false AND datname != 'postgres';")

for DB in ${DATABASES}; do
    DB=$(echo ${DB} | tr -d '[:space:]')
    BACKUP_FILE="${BACKUP_DIR}/${BACKUP_TYPE}/${DB}_${TIMESTAMP}.sql.gz"
    
    log "Backing up database: ${DB}"
    
    docker exec ${CONTAINER_NAME} pg_dump -U ${POSTGRES_USER} ${DB} | gzip > "${BACKUP_FILE}"
    
    if [ $? -eq 0 ]; then
        log "✓ Successfully backed up ${DB} ($(du -h ${BACKUP_FILE} | cut -f1))"
    else
        log "✗ Failed to backup ${DB}"
    fi
done

# Backup global objects (roles, tablespaces)
GLOBALS_FILE="${BACKUP_DIR}/${BACKUP_TYPE}/globals_${TIMESTAMP}.sql.gz"
log "Backing up global objects..."
docker exec ${CONTAINER_NAME} pg_dumpall -U ${POSTGRES_USER} -g | gzip > "${GLOBALS_FILE}"

# Cleanup old backups
log "Cleaning up old backups..."

# Daily backups (keep last N days)
find "${BACKUP_DIR}/daily" -name "*.sql.gz" -mtime +${DAILY_RETENTION} -delete
log "Removed daily backups older than ${DAILY_RETENTION} days"

# Weekly backups (keep last N weeks)
find "${BACKUP_DIR}/weekly" -name "*.sql.gz" -mtime +$((WEEKLY_RETENTION * 7)) -delete
log "Removed weekly backups older than ${WEEKLY_RETENTION} weeks"

# Monthly backups (keep last N months)
find "${BACKUP_DIR}/monthly" -name "*.sql.gz" -mtime +$((MONTHLY_RETENTION * 30)) -delete
log "Removed monthly backups older than ${MONTHLY_RETENTION} months"

log "Backup completed successfully"

# Summary
log "Backup Summary:"
log "  Daily backups: $(find ${BACKUP_DIR}/daily -name "*.sql.gz" | wc -l) files"
log "  Weekly backups: $(find ${BACKUP_DIR}/weekly -name "*.sql.gz" | wc -l) files"
log "  Monthly backups: $(find ${BACKUP_DIR}/monthly -name "*.sql.gz" | wc -l) files"
log "  Total size: $(du -sh ${BACKUP_DIR} | cut -f1)"
```

### Restore Script

```bash
# roles/database-management/templates/restore-script.sh.j2
#!/bin/bash
# PostgreSQL Restore Script

set -euo pipefail

# Configuration
BACKUP_DIR="{{ POSTGRESQL_BACKUP_PATH }}"
CONTAINER_NAME="postgresql"
POSTGRES_USER="postgres"

# Usage
usage() {
    echo "Usage: $0 -d DATABASE -f BACKUP_FILE"
    echo "  -d: Database name to restore"
    echo "  -f: Backup file path (must be .sql.gz)"
    exit 1
}

# Parse arguments
while getopts "d:f:" opt; do
    case $opt in
        d) DATABASE="$OPTARG" ;;
        f) BACKUP_FILE="$OPTARG" ;;
        *) usage ;;
    esac
done

# Validate arguments
if [ -z "${DATABASE:-}" ] || [ -z "${BACKUP_FILE:-}" ]; then
    usage
fi

if [ ! -f "${BACKUP_FILE}" ]; then
    echo "Error: Backup file not found: ${BACKUP_FILE}"
    exit 1
fi

# Confirmation
echo "WARNING: This will restore database '${DATABASE}' from backup."
echo "Backup file: ${BACKUP_FILE}"
echo "All existing data in '${DATABASE}' will be replaced!"
read -p "Are you sure? (yes/no): " CONFIRM

if [ "${CONFIRM}" != "yes" ]; then
    echo "Restore cancelled."
    exit 0
fi

# Drop existing database (if exists)
echo "Dropping existing database..."
docker exec ${CONTAINER_NAME} psql -U ${POSTGRES_USER} -c "DROP DATABASE IF EXISTS ${DATABASE};"

# Create new database
echo "Creating database..."
docker exec ${CONTAINER_NAME} psql -U ${POSTGRES_USER} -c "CREATE DATABASE ${DATABASE};"

# Restore from backup
echo "Restoring from backup..."
gunzip -c "${BACKUP_FILE}" | docker exec -i ${CONTAINER_NAME} psql -U ${POSTGRES_USER} ${DATABASE}

if [ $? -eq 0 ]; then
    echo "✓ Database '${DATABASE}' restored successfully"
else
    echo "✗ Failed to restore database '${DATABASE}'"
    exit 1
fi
```

---

## Integration with Existing Services

### Langfuse Integration

```yaml
# roles/langfuse/files/docker-compose.yml (excerpt)

services:
  langfuse:
    image: langfuse/langfuse:latest
    environment:
      # Database Connection
      DATABASE_URL: "postgresql://langfuse_user:${LANGFUSE_DB_PASSWORD}@postgresql:5432/langfuse"
      
      # Use connection pooling
      DATABASE_POOL_SIZE: 20
      DATABASE_MAX_OVERFLOW: 10
    
    depends_on:
      postgresql:
        condition: service_healthy
    
    networks:
      - self-hosted
```

### Open WebUI Migration (SQLite to PostgreSQL)

```yaml
# roles/open-webui/files/docker-compose.yml (updated)

services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:latest
    environment:
      # Database Configuration (PostgreSQL instead of SQLite)
      DATABASE_URL: "postgresql://openwebui_user:${OPENWEBUI_DB_PASSWORD}@postgresql:5432/openwebui"
      
      # Or keep SQLite for simplicity (optional)
      # DATABASE_URL: "sqlite:///app/backend/data/webui.db"
    
    depends_on:
      postgresql:
        condition: service_healthy
    
    networks:
      - self-hosted
```

---

## Backup & Recovery Strategy

### Backup Strategy Matrix

| Type | Frequency | Retention | Method | Priority |
|------|-----------|-----------|--------|----------|
| **Daily** | Every day at 2 AM | 7 days | pg_dump (compressed) | High |
| **Weekly** | Sunday at 2 AM | 4 weeks | pg_dump (compressed) | Medium |
| **Monthly** | 1st of month at 2 AM | 3 months | pg_dump (compressed) | Medium |
| **On-Demand** | Manual | Until deleted | pg_dump or pg_basebackup | Critical |

### Cron Job Configuration

```yaml
# roles/database-management/tasks/configure-backups.yml

- name: Create backup script
  ansible.builtin.template:
    src: backup-script.sh.j2
    dest: /usr/local/bin/postgres-backup.sh
    mode: '0750'
    owner: root
    group: root

- name: Create restore script
  ansible.builtin.template:
    src: restore-script.sh.j2
    dest: /usr/local/bin/postgres-restore.sh
    mode: '0750'
    owner: root
    group: root

- name: Setup cron job for automated backups
  ansible.builtin.cron:
    name: "PostgreSQL automated backup"
    minute: "0"
    hour: "2"
    job: "/usr/local/bin/postgres-backup.sh >> /var/log/postgres-backup.log 2>&1"
    user: root
    state: present
```

---

## Implementation Steps

### Phase 1: Role Creation (Day 1)

```bash
# Step 1: Create role structure
ansible-galaxy role init database-management
cd roles/database-management

# Step 2: Create required files
touch tasks/{validate,setup-storage,deploy-postgresql,initialize-databases,deploy-pgadmin,configure-backups,verify}.yml
touch templates/{.postgresql.env.j2,.pgadmin.env.j2,01-create-users.sql.j2,02-create-databases.sql.j2,03-grant-permissions.sql.j2,backup-script.sh.j2,restore-script.sh.j2}
touch files/docker-compose.yml

# Step 3: Implement validation tasks
# Step 4: Implement deployment tasks
# Step 5: Implement verification tasks
```

### Phase 2: Configuration (Day 1)

```bash
# Step 1: Add variables to group_vars/homelab.yml
# Step 2: Add secrets to group_vars/secrets.yml (vault)
# Step 3: Update server_playbook.yml
# Step 4: Update dependencies in meta/main.yml
```

### Phase 3: Testing (Day 2)

```bash
# Step 1: Syntax check
ansible-playbook --syntax-check server_playbook.yml

# Step 2: Dry run
ansible-playbook server_playbook.yml --tags database --check --ask-vault-pass

# Step 3: Deploy to test environment
ansible-playbook server_playbook.yml --tags database --ask-vault-pass

# Step 4: Verify deployment
ansible-playbook verify_services.yml
```

### Phase 4: Integration (Day 2-3)

```bash
# Step 1: Update Langfuse role to use PostgreSQL
# Step 2: Test Langfuse deployment
# Step 3: Create migration guide for Open WebUI (optional)
# Step 4: Update documentation
```

### Phase 5: Documentation (Day 3)

```bash
# Step 1: Create role README.md
# Step 2: Update main ARCHITECTURE.md
# Step 3: Update DEPLOYMENT_GUIDE.md
# Step 4: Create database management guide
```

---

## Testing & Validation

### Validation Checklist

```yaml
Pre-Deployment Validation:
├─ ✓ Docker service running
├─ ✓ Docker network 'self-hosted' exists
├─ ✓ Storage paths writable
├─ ✓ Sufficient disk space (>20GB)
├─ ✓ PostgreSQL variables defined
└─ ✓ Database users/passwords in vault

Deployment Validation:
├─ ✓ PostgreSQL container running
├─ ✓ Health check passing
├─ ✓ Port 5432 listening (internal)
├─ ✓ Initialization scripts executed
├─ ✓ Databases created
├─ ✓ Users created with correct permissions
└─ ✓ pgAdmin accessible (if enabled)

Post-Deployment Validation:
├─ ✓ Can connect from application containers
├─ ✓ Can create/read/write data
├─ ✓ Backup script executable
├─ ✓ Cron job configured
├─ ✓ Logs being written
└─ ✓ Performance metrics acceptable
```

### Test Scenarios

```bash
# 1. Connection Test
docker exec postgresql psql -U postgres -c "SELECT version();"

# 2. Database List
docker exec postgresql psql -U postgres -c "\l"

# 3. User List
docker exec postgresql psql -U postgres -c "\du"

# 4. Create Test Table
docker exec postgresql psql -U langfuse_user -d langfuse -c "
CREATE TABLE test (id SERIAL PRIMARY KEY, name VARCHAR(50));
INSERT INTO test (name) VALUES ('test1'), ('test2');
SELECT * FROM test;
DROP TABLE test;
"

# 5. Connection from Application Container
docker exec open-webui psql -h postgresql -U openwebui_user -d openwebui -c "SELECT 1;"

# 6. Backup Test
/usr/local/bin/postgres-backup.sh

# 7. Restore Test
/usr/local/bin/postgres-restore.sh -d langfuse -f /path/to/backup.sql.gz

# 8. Performance Test
docker exec postgresql psql -U postgres -c "
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
"
```

---

## Future Enhancements

### Phase 2 Features (Future)

1. **High Availability**
   - PostgreSQL replication (primary + replica)
   - Automatic failover
   - Read replicas for scaling

2. **Advanced Monitoring**
   - PostgreSQL Exporter for Prometheus
   - Grafana dashboards
   - Query performance monitoring
   - Connection pooling metrics

3. **Connection Pooling**
   - PgBouncer container
   - Connection pooling for all applications
   - Reduced connection overhead

4. **Enhanced Security**
   - SSL/TLS encryption for all connections
   - Certificate-based authentication
   - Audit logging with pgaudit
   - Automated security patches

5. **Automated Migrations**
   - Flyway or Liquibase integration
   - Version-controlled schema changes
   - Automated rollback capability

6. **Performance Optimization**
   - Automated query optimization
   - Index recommendations
   - Vacuum automation
   - Table partitioning

---

## Summary

### What Will Be Delivered

✅ **Complete Role Implementation**
- Fully functional database-management role
- Follows all existing patterns and conventions
- Comprehensive validation and verification

✅ **PostgreSQL Service**
- PostgreSQL 16 container
- Optimized configuration
- Internal-only access (secure)

✅ **Management Tools**
- pgAdmin web interface (optional)
- Automated backup scripts
- Restore procedures
- Database initialization

✅ **Security**
- Zero-trust architecture maintained
- Encrypted secrets (Ansible Vault)
- Per-application database isolation
- Comprehensive logging

✅ **Documentation**
- Complete role README
- Integration guides
- Backup/restore procedures
- Troubleshooting guide

✅ **Integration**
- Ready for Langfuse
- Compatible with Open WebUI
- Easy to add new applications

### Timeline

- **Day 1**: Role creation + configuration (4-6 hours)
- **Day 2**: Testing + integration (3-4 hours)
- **Day 3**: Documentation + refinement (2-3 hours)
- **Total**: 2-3 days for complete implementation

### Next Steps

1. ✅ Review this plan
2. ⏭️ Create role structure
3. ⏭️ Implement validation tasks
4. ⏭️ Implement deployment tasks
5. ⏭️ Add to server_playbook.yml
6. ⏭️ Test deployment
7. ⏭️ Integrate with Langfuse
8. ⏭️ Complete documentation

---

**Ready to proceed with implementation?** 🚀

Let me know if you'd like me to start creating the actual role files!
