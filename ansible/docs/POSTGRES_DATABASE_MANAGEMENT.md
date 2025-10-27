# PostgreSQL Database Management Guide

## Overview

This guide explains how to manage PostgreSQL databases, users, and credentials in your Ansible homelab environment. The database-management role provides a declarative way to define and manage multiple databases and users.

---

## Current Configuration

### Admin Credentials

Your PostgreSQL instance has the following admin accounts:

| User | Password Variable | Purpose | Defined In |
|------|------------------|---------|------------|
| `postgres` | `POSTGRES_ROOT_PASSWORD` | PostgreSQL superuser (root) | `secrets.yml` |
| `db_admin` | `POSTGRES_ADMIN_PASSWORD` | Admin role with CREATEDB/CREATEROLE | `secrets.yml` |

**Access these from Ansible Vault:**
```bash
# View encrypted secrets
ansible-vault view group_vars/secrets.yml

# Edit secrets
ansible-vault edit group_vars/secrets.yml
```

### Currently Defined Databases

**Status:** No databases are currently defined.

The role includes commented examples in `group_vars/homelab.yml` (lines 93-105) that you can uncomment and customize.

### Currently Defined Users

**Status:** No application users are currently defined.

The role includes commented examples in `group_vars/homelab.yml` (lines 107-116) that you can uncomment and customize.

---

## How to Add a New Database

### Step 1: Define Database Users in `secrets.yml`

Users and their passwords should be stored in the encrypted `secrets.yml` file for security.

**Example: Creating a user for a Langfuse application**

```bash
# Edit the encrypted secrets file
ansible-vault edit group_vars/secrets.yml
```

Add the user password(s):

```yaml
# PostgreSQL Root and Admin Passwords
POSTGRES_ROOT_PASSWORD: "your_existing_root_password"
POSTGRES_ADMIN_PASSWORD: "your_existing_admin_password"

# Application User Passwords
LANGFUSE_DB_PASSWORD: "secure_random_password_here"
```

**Generate Secure Passwords:**
```bash
# Generate a 32-character random password
openssl rand -base64 32

# Or use pwgen
pwgen -s 32 1
```

### Step 2: Define Users in `homelab.yml`

Edit `group_vars/homelab.yml` and add the `POSTGRES_DATABASE_USERS` section:

```yaml
# PostgreSQL Database Users
POSTGRES_DATABASE_USERS:
  - username: langfuse_user
    password: "{{ LANGFUSE_DB_PASSWORD }}"  # Reference from secrets.yml
    is_superuser: false                      # Don't grant superuser privileges
    can_create_db: false                     # Can't create other databases
    can_create_role: false                   # Can't create other users
    connection_limit: 10                     # Max concurrent connections
```

**User Configuration Options:**

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `username` | string | PostgreSQL username | **Required** |
| `password` | string | User password (use Vault variable) | **Required** |
| `is_superuser` | boolean | Grant SUPERUSER privilege | `false` |
| `can_create_db` | boolean | Grant CREATEDB privilege | `false` |
| `can_create_role` | boolean | Grant CREATEROLE privilege | `false` |
| `connection_limit` | integer | Max simultaneous connections | `-1` (unlimited) |

### Step 3: Define Databases in `homelab.yml`

Add the `POSTGRES_DATABASES` section:

```yaml
# PostgreSQL Databases
POSTGRES_DATABASES:
  - name: langfuse_db
    owner: langfuse_user                     # User created in Step 2
    description: "Langfuse AI observability database"
    encoding: UTF8                           # Character encoding
    locale: en_US.UTF-8                      # Database locale
    connection_limit: 50                     # Max connections to this DB
    allowed_users: []                        # Additional users with full access
    readonly_users: []                       # Users with read-only access
    enable_pgvector: true                    # Enable vector similarity search
    enable_uuid: true                        # Enable UUID generation
    enable_audit_log: false                  # Create audit_log table
```

**Database Configuration Options:**

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `name` | string | Database name | **Required** |
| `owner` | string | Database owner (must exist in POSTGRES_DATABASE_USERS) | **Required** |
| `description` | string | Human-readable description | `""` |
| `encoding` | string | Character encoding | `UTF8` |
| `locale` | string | Locale for sorting/formatting | `en_US.UTF-8` |
| `connection_limit` | integer | Max connections to database | `-1` (unlimited) |
| `allowed_users` | list | Usernames with CONNECT, CREATE, and full permissions | `[]` |
| `readonly_users` | list | Usernames with read-only SELECT permissions | `[]` |
| `enable_pgvector` | boolean | Enable pgvector extension for AI embeddings | `false` |
| `enable_uuid` | boolean | Enable uuid-ossp extension | `true` |
| `enable_audit_log` | boolean | Create audit_log table for compliance | `false` |

### Step 4: Deploy the Changes

Run the Ansible playbook to apply your changes:

```bash
# Check mode (dry run) to preview changes
ansible-playbook server_playbook.yml --tags database --check

# Apply changes
ansible-playbook server_playbook.yml --tags database

# Or just PostgreSQL tasks
ansible-playbook server_playbook.yml --tags postgres
```

### Step 5: Verify the Database

Connect from any device on your Tailnet:

```bash
# Connect as the database owner
psql "postgresql://langfuse_user:PASSWORD@100.64.x.x:5432/langfuse_db"

# Verify pgvector extension (if enabled)
\dx vector

# Check database info
\l langfuse_db
```

---

## Complete Example: Adding a New Application Database

Let's walk through a complete example of adding a database for a hypothetical "n8n" workflow automation tool.

### Scenario Requirements

- **Application:** n8n (workflow automation)
- **Database Name:** `n8n_production`
- **Owner:** `n8n_app`
- **Additional Access:** 
  - `analytics_user` (read-only access for reporting)
  - `backup_user` (full access for backup jobs)
- **Features:** Enable UUID support, no pgvector needed

### Implementation

**1. Edit `group_vars/secrets.yml`**

```bash
ansible-vault edit group_vars/secrets.yml
```

Add passwords:

```yaml
# ... existing passwords ...

# n8n Application Passwords
N8N_APP_PASSWORD: "n8n_secure_password_123"
N8N_ANALYTICS_PASSWORD: "analytics_readonly_456"
N8N_BACKUP_PASSWORD: "backup_secure_789"
```

**2. Edit `group_vars/homelab.yml`**

Add users:

```yaml
POSTGRES_DATABASE_USERS:
  # n8n Application User (owner)
  - username: n8n_app
    password: "{{ N8N_APP_PASSWORD }}"
    is_superuser: false
    can_create_db: false
    can_create_role: false
    connection_limit: 20

  # Analytics User (read-only)
  - username: analytics_user
    password: "{{ N8N_ANALYTICS_PASSWORD }}"
    is_superuser: false
    can_create_db: false
    can_create_role: false
    connection_limit: 5

  # Backup User (full access to multiple DBs)
  - username: backup_user
    password: "{{ N8N_BACKUP_PASSWORD }}"
    is_superuser: false
    can_create_db: false
    can_create_role: false
    connection_limit: 2
```

Add database:

```yaml
POSTGRES_DATABASES:
  - name: n8n_production
    owner: n8n_app
    description: "n8n workflow automation production database"
    encoding: UTF8
    locale: en_US.UTF-8
    connection_limit: 50
    allowed_users:
      - backup_user      # Full access for backups
    readonly_users:
      - analytics_user   # Read-only for reports
    enable_pgvector: false
    enable_uuid: true
    enable_audit_log: true
```

**3. Deploy**

```bash
# Preview changes
ansible-playbook server_playbook.yml --tags postgres --check

# Apply
ansible-playbook server_playbook.yml --tags postgres
```

**4. Test Connections**

```bash
# Test as owner (full access)
psql "postgresql://n8n_app:PASSWORD@100.64.x.x:5432/n8n_production"
n8n_production=> CREATE TABLE test (id serial primary key);
-- Success!

# Test as analytics user (read-only)
psql "postgresql://analytics_user:PASSWORD@100.64.x.x:5432/n8n_production"
n8n_production=> SELECT * FROM test;
-- Success!
n8n_production=> INSERT INTO test VALUES (1);
-- Error: permission denied (expected!)

# Test as backup user (full access via allowed_users)
psql "postgresql://backup_user:PASSWORD@100.64.x.x:5432/n8n_production"
n8n_production=> SELECT * FROM test;
-- Success!
n8n_production=> INSERT INTO test VALUES (2);
-- Success!
```

---

## Adding Databases to Existing Deployments

If PostgreSQL is already running, the role will:

1. ✅ Create new users (if they don't exist)
2. ✅ Update user passwords (if changed)
3. ✅ Create new databases (if they don't exist)
4. ✅ Enable extensions (if not already enabled)
5. ✅ Grant permissions to new users
6. ❌ **Does NOT** drop existing databases or users

**Safe to Re-run:** You can safely re-run the playbook multiple times. It's idempotent and won't break existing databases.

### Updating Existing Database Permissions

To add a user to an existing database:

```yaml
POSTGRES_DATABASES:
  - name: existing_database
    owner: original_owner
    allowed_users:
      - new_user_with_full_access  # Add this user
    readonly_users:
      - new_readonly_user           # Or add read-only
```

Re-run the playbook to apply permission changes:

```bash
ansible-playbook server_playbook.yml --tags postgres
```

---

## Advanced Configuration

### Multiple Databases for One Application

```yaml
POSTGRES_DATABASE_USERS:
  - username: myapp_user
    password: "{{ MYAPP_PASSWORD }}"
    connection_limit: 20

POSTGRES_DATABASES:
  - name: myapp_production
    owner: myapp_user
    enable_audit_log: true

  - name: myapp_staging
    owner: myapp_user
    enable_audit_log: false

  - name: myapp_testing
    owner: myapp_user
    connection_limit: 10
```

### Shared Database with Multiple Owners

```yaml
POSTGRES_DATABASE_USERS:
  - username: service_a_user
    password: "{{ SERVICE_A_PASSWORD }}"
  
  - username: service_b_user
    password: "{{ SERVICE_B_PASSWORD }}"

POSTGRES_DATABASES:
  - name: shared_data
    owner: service_a_user
    allowed_users:
      - service_b_user  # Full access to service B
```

### AI Application with pgvector

```yaml
POSTGRES_DATABASE_USERS:
  - username: ai_app_user
    password: "{{ AI_APP_PASSWORD }}"
    connection_limit: 30

POSTGRES_DATABASES:
  - name: ai_embeddings
    owner: ai_app_user
    description: "Vector embeddings for semantic search"
    enable_pgvector: true      # Enable vector extension
    enable_uuid: true
    enable_audit_log: false
```

After deployment, test pgvector:

```sql
-- Connect to the database
\c ai_embeddings

-- Create a table with vector column
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT,
    embedding VECTOR(1536)  -- OpenAI ada-002 dimensions
);

-- Insert test data
INSERT INTO documents (content, embedding)
VALUES ('test', '[0,0,0,...]'::vector);

-- Find similar vectors (cosine similarity)
SELECT content, embedding <=> '[0,0,0,...]'::vector AS distance
FROM documents
ORDER BY distance
LIMIT 5;
```

---

## Troubleshooting

### Can't Connect to Database

**1. Check PostgreSQL is running:**
```bash
ansible-playbook server_playbook.yml --tags database -e "DEBUG_MODE=true"
```

**2. Verify from the server:**
```bash
ssh homelab-server
docker ps | grep postgres
docker logs database-management-postgres-1
```

**3. Test connection from server localhost:**
```bash
psql "postgresql://postgres:PASSWORD@localhost:5432/postgres"
```

**4. Check Tailscale IP:**
```bash
tailscale ip -4
# Should show 100.64.x.x address
```

### Permission Denied Errors

**Problem:** User can't access database

**Solution:** Check that user is in `allowed_users` or `readonly_users`:

```yaml
POSTGRES_DATABASES:
  - name: my_database
    owner: app_user
    allowed_users:
      - additional_user  # Add this
```

### Password Changes Not Applied

The role uses `ALTER USER` which updates passwords on every run. If passwords aren't updating:

1. Verify the password in `secrets.yml`:
   ```bash
   ansible-vault view group_vars/secrets.yml
   ```

2. Force password update:
   ```bash
   ansible-playbook server_playbook.yml --tags postgres -e "FORCE_PASSWORD_UPDATE=true"
   ```

### Database Already Exists with Different Owner

If you need to change ownership:

```sql
-- Connect as postgres superuser
psql "postgresql://postgres:PASSWORD@100.64.x.x:5432/postgres"

-- Change database owner
ALTER DATABASE database_name OWNER TO new_owner;

-- Reassign object ownership within database
\c database_name
REASSIGN OWNED BY old_owner TO new_owner;
```

---

## Security Best Practices

### 1. Use Strong Passwords

```bash
# Generate secure passwords
openssl rand -base64 32 > password.txt

# Or use a password manager
```

### 2. Limit Connection Counts

```yaml
POSTGRES_DATABASE_USERS:
  - username: webapp_user
    connection_limit: 20  # Prevent connection exhaustion
```

### 3. Use Read-Only Users for Analytics

```yaml
POSTGRES_DATABASES:
  - name: production_db
    owner: app_user
    readonly_users:
      - analyst_user      # Can only SELECT
      - reporting_tool    # Dashboard access
```

### 4. Enable Audit Logging for Compliance

```yaml
POSTGRES_DATABASES:
  - name: sensitive_data
    owner: app_user
    enable_audit_log: true  # Creates audit_log table
```

### 5. Rotate Passwords Regularly

```bash
# 1. Edit secrets
ansible-vault edit group_vars/secrets.yml

# 2. Update password
MYAPP_PASSWORD: "new_secure_password"

# 3. Deploy changes
ansible-playbook server_playbook.yml --tags postgres

# 4. Restart application to use new password
```

---

## Backup and Recovery

### Automated Backups

Backups are configured in `group_vars/homelab.yml`:

```yaml
POSTGRES_BACKUP_SCHEDULE: "0 2 * * *"  # Daily at 2 AM
POSTGRES_BACKUP_RETENTION_DAILY: 7      # Keep 7 daily backups
POSTGRES_BACKUP_RETENTION_WEEKLY: 30    # Keep 30 weekly backups
POSTGRES_BACKUP_RETENTION_MONTHLY: 365  # Keep 365 monthly backups
```

**Backup Location:**
```
/cold-storage/database/postgresql/backups/
├── daily/
│   ├── postgres_all_2025-10-25.sql.gz
│   └── langfuse_db_2025-10-25.sql.gz
├── weekly/
└── monthly/
```

### Manual Backup

```bash
# Run backup script manually
ssh homelab-server
sudo /usr/local/bin/postgresql-backup.sh

# Or trigger via Ansible
ansible-playbook server_playbook.yml --tags database-bkp
```

### Restore a Database

```bash
# 1. Copy backup to local machine
scp homelab-server:/cold-storage/database/postgresql/backups/daily/mydb_2025-10-25.sql.gz .

# 2. Decompress
gunzip mydb_2025-10-25.sql.gz

# 3. Restore
psql "postgresql://postgres:PASSWORD@100.64.x.x:5432/postgres" -f mydb_2025-10-25.sql
```

---

## Connection Strings for Applications

### Standard Connection String

```
postgresql://username:password@100.64.x.x:5432/database_name
```

### Environment Variables

```bash
# For Docker Compose applications
DATABASE_URL="postgresql://langfuse_user:${LANGFUSE_DB_PASSWORD}@${TAILSCALE_IP_ADDRESS}:5432/langfuse_db"

# Or split format
DB_HOST=100.64.x.x
DB_PORT=5432
DB_NAME=langfuse_db
DB_USER=langfuse_user
DB_PASSWORD=${LANGFUSE_DB_PASSWORD}
```

### Connection Pooling (PgBouncer)

For high-traffic applications, consider adding PgBouncer in a future enhancement:

```yaml
# Future enhancement idea
POSTGRES_ENABLE_PGBOUNCER: true
PGBOUNCER_POOL_MODE: "transaction"
PGBOUNCER_MAX_CLIENT_CONN: 1000
PGBOUNCER_DEFAULT_POOL_SIZE: 20
```

---

## Quick Reference

### View Current Databases

```sql
-- Connect as admin
psql "postgresql://db_admin:PASSWORD@100.64.x.x:5432/postgres"

-- List databases
\l

-- List users
\du

-- Show database sizes
SELECT pg_database.datname, pg_size_pretty(pg_database_size(pg_database.datname))
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;
```

### Common Ansible Commands

```bash
# Deploy database changes
ansible-playbook server_playbook.yml --tags postgres

# Run backup only
ansible-playbook server_playbook.yml --tags database-bkp

# Check configuration (dry run)
ansible-playbook server_playbook.yml --tags database --check

# Debug mode
ansible-playbook server_playbook.yml --tags postgres -vvv

# Skip backup configuration
ansible-playbook server_playbook.yml --tags postgres --skip-tags database-bkp
```

---

## Next Steps

1. **Populate `secrets.yml`** with secure passwords for your databases
2. **Define your first database** in `homelab.yml` using the examples above
3. **Deploy** with `ansible-playbook server_playbook.yml --tags postgres`
4. **Test connection** from your local machine on Tailnet
5. **Configure your application** to use the database connection string

For more information, see:
- [ENVIRONMENT_VARIABLES.md](ENVIRONMENT_VARIABLES.md) - Complete environment variable reference
- [DATABASE_MANAGEMENT_PLAN.md](DATABASE_MANAGEMENT_PLAN.md) - Architecture overview
- [ARCHITECTURE.md](ARCHITECTURE.md) - Overall homelab design
- [roles/database-management/tasks/README.md](../roles/database-management/tasks/README.md) - Adding new database engines
