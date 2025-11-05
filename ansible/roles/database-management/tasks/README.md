# Database Management Tasks Structure

This directory contains database engine implementations organized by engine type.

## Directory Structure

```
tasks/
├── main.yml                    # Main orchestration (engine-agnostic)
├── validate.yml                # Pre-flight validation
├── setup-storage.yml           # Storage directory creation
├── configure-backups.yml       # Backup automation (engine-specific logic needed)
├── verify.yml                  # Post-deployment verification
└── postgres/                   # PostgreSQL implementation
    ├── main.yml                # PostgreSQL orchestration
    ├── deploy-postgresql.yml   # Container deployment
    ├── initialize-databases.yml # Database/user initialization
    └── deploy-pgadmin.yml      # pgAdmin management UI
```

## Adding New Database Engines

To add support for a new database engine (e.g., SQL Server, Oracle):

### 1. Create Engine Directory

```bash
mkdir -p tasks/sqlserver/
# or
mkdir -p tasks/oracle/
```

### 2. Create Engine Main File

**tasks/sqlserver/main.yml:**
```yaml
---
# SQL Server Database Engine Tasks
- name: Deploy SQL Server container
  ansible.builtin.include_tasks: deploy-sqlserver.yml
  tags:
    - sqlserver
    - deploy

- name: Initialize SQL Server databases
  ansible.builtin.include_tasks: initialize-databases.yml
  tags:
    - sqlserver
    - init
```

### 3. Implement Engine-Specific Tasks

Create files like:
- `deploy-sqlserver.yml` - Container deployment
- `initialize-databases.yml` - Database creation
- `configure-users.yml` - User management
- `deploy-management-ui.yml` - Management tools (e.g., SQL Server Management Studio)

### 4. Update Main Orchestration

Add the engine block in `tasks/main.yml`:

```yaml
- name: Deploy SQL Server database engine
  ansible.builtin.include_tasks: sqlserver/main.yml
  tags:
    - database
    - database-management
    - sqlserver
    - deploy
```

### 5. Add Engine Variables

Add engine-specific variables to `group_vars/homelab.yml`:

```yaml
# SQL Server Configuration
SQLSERVER_VERSION: "2022-latest"
SQLSERVER_DATA_PATH: "{{ COLD_STORAGE_PATH }}/database/sqlserver/data"
SQLSERVER_BACKUP_PATH: "{{ COLD_STORAGE_PATH }}/database/sqlserver/backup"
SQLSERVER_SA_PASSWORD: "{{ SQLSERVER_SA_PASSWORD }}"  # Define in secrets.yml
```

### 6. Update Backup Scripts

Modify `configure-backups.yml` to support engine-specific backup methods:

```yaml
- name: Deploy SQL Server backup script
  ansible.builtin.template:
    src: backup-script-sqlserver.sh.j2
    dest: /usr/local/bin/sqlserver-backup.sh
```

## Multiple Database Engines

The role supports deploying multiple database engines simultaneously. Each engine's tasks are executed independently when the role runs. To deploy a specific engine, use tags:

```bash
# Deploy only PostgreSQL
ansible-playbook server_playbook.yml --tags postgres

# Deploy only SQL Server (when implemented)
ansible-playbook server_playbook.yml --tags sqlserver

# Deploy all database engines
ansible-playbook server_playbook.yml --tags database
```

## Engine-Specific Tags

Each engine has dedicated tags for targeted deployment:

```bash
# Deploy only PostgreSQL
ansible-playbook server_playbook.yml --tags postgres

# Deploy only SQL Server (future)
ansible-playbook server_playbook.yml --tags sqlserver

# Deploy only Oracle (future)
ansible-playbook server_playbook.yml --tags oracle
```

## Current Engine Status

| Engine | Status | Version | Extensions |
|--------|--------|---------|------------|
| PostgreSQL | ✅ Implemented | 16-alpine | pgvector, uuid-ossp, pg_trgm |
| SQL Server | 🔜 Planned | - | - |
| Oracle | 🔜 Planned | - | - |
| MySQL/MariaDB | 🔜 Planned | - | - |

## Best Practices

1. **Isolation**: Keep engine-specific logic in dedicated subfolders
2. **Consistency**: Follow the same task naming pattern across engines
3. **Variables**: Use `ENGINE_PROPERTY` naming (e.g., `POSTGRES_PORT`, `SQLSERVER_PORT`)
4. **Templates**: Store in `templates/engine/` subfolders (e.g., `templates/postgres/`, `templates/sqlserver/`)
5. **Validation**: Add engine-specific validation in `validate.yml`
6. **Documentation**: Update role README.md with engine-specific usage

## Example: Adding SQL Server

```bash
# 1. Create directory
mkdir -p tasks/sqlserver

# 2. Create main orchestration
cat > tasks/sqlserver/main.yml << 'EOF'
---
- name: Deploy SQL Server container
  ansible.builtin.include_tasks: deploy-sqlserver.yml
- name: Initialize databases
  ansible.builtin.include_tasks: initialize-databases.yml
EOF

# 3. Create deployment task
cat > tasks/sqlserver/deploy-sqlserver.yml << 'EOF'
---
- name: Pull SQL Server image
  community.docker.docker_image:
    name: "mcr.microsoft.com/mssql/server:{{ SQLSERVER_VERSION }}"
    source: pull
EOF

# 4. Deploy
ansible-playbook server_playbook.yml --tags database
```
