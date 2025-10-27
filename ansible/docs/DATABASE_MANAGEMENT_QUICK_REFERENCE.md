# Database Management Role - Quick Reference

## 🚀 Quick Deployment

```bash
# Full deployment
ansible-playbook server_playbook.yml --tags database

# PostgreSQL only
ansible-playbook server_playbook.yml --tags postgres

# pgAdmin only
ansible-playbook server_playbook.yml --tags pgadmin

# Fix pgAdmin permissions
ansible-playbook fix_pgadmin_permissions.yml -i inventory.ini
```

## 📂 Key Files

| File | Purpose |
|------|---------|
| `group_vars/database-management/engine.yml` | Database engine selection |
| `group_vars/database-management/postgres/vars.yml` | PostgreSQL configuration |
| `group_vars/database-management/postgres/secrets.yml` | PostgreSQL passwords (vault) |
| `group_vars/database-management/pgadmin/vars.yml` | pgAdmin configuration |
| `group_vars/database-management/pgadmin/secrets.yml` | pgAdmin password (vault) |

## 🔐 Required Secrets

```bash
# Edit secrets
ansible-vault edit group_vars/database-management/postgres/secrets.yml

# Required variables:
POSTGRES_ROOT_PASSWORD: "your_secure_password"
POSTGRES_ADMIN_PASSWORD: "your_admin_password"
```

## 📝 Task Files

| Task | File | Purpose |
|------|------|---------|
| **Main** | `tasks/main.yml` | Orchestrates all tasks |
| **Validate** | `tasks/postgres/validate.yml` | Pre-flight checks |
| **Storage** | `tasks/postgres/setup-storage.yml` | Create directories |
| **Deploy** | `tasks/postgres/deploy-postgresql.yml` | Deploy containers |
| **Initialize** | `tasks/postgres/initialize-databases.yml` | Create DBs & users |
| **Backup** | `tasks/postgres/configure-backups.yml` | Setup backups |
| **Verify** | `tasks/postgres/verify.yml` | Health checks |
| **pgAdmin Deploy** | `tasks/pgadmin/deploy.yml` | Deploy pgAdmin |
| **pgAdmin Verify** | `tasks/pgadmin/verify.yml` | Verify pgAdmin |

## 🔧 Common Commands

### Check Status
```bash
# Container status
ansible homelab -i inventory.ini -m shell -a "docker ps --filter name=postgresql"
ansible homelab -i inventory.ini -m shell -a "docker ps --filter name=pgadmin"

# Check logs
ansible homelab -i inventory.ini -m shell -a "docker logs postgresql --tail 50"
ansible homelab -i inventory.ini -m shell -a "docker logs pgadmin --tail 50"

# List databases
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql psql -U postgres -c '\l'"
```

### Connect to PostgreSQL
```bash
# From any Tailscale device
psql "postgresql://postgres:PASSWORD@100.84.146.121:5432/postgres"

# From server
ansible homelab -i inventory.ini -m shell -a "docker exec -it postgresql psql -U postgres"
```

### Directory Ownership
```bash
# PostgreSQL (should be 70:70)
ansible homelab -i inventory.ini -m shell -a "stat -c '%u:%g %a' /cold-storage/database/postgresql/data" --become

# pgAdmin (should be 5050:0)
ansible homelab -i inventory.ini -m shell -a "stat -c '%u:%g %a' /cold-storage/database/pgadmin/data" --become
```

### Fix Permissions
```bash
# PostgreSQL
ansible homelab -i inventory.ini -m shell -a "chown -R 70:70 /cold-storage/database/postgresql/data" --become

# pgAdmin (use playbook for idempotent fix)
ansible-playbook fix_pgadmin_permissions.yml -i inventory.ini
```

## 🗄️ Add New Database

1. **Add password to secrets.yml**:
```bash
ansible-vault edit group_vars/database-management/postgres/secrets.yml
```
```yaml
MYAPP_DB_PASSWORD: "secure_password"
```

2. **Add user to vars.yml**:
```bash
vim group_vars/database-management/postgres/vars.yml
```
```yaml
POSTGRES_DATABASE_USERS:
  - username: myapp_user
    password: "{{ MYAPP_DB_PASSWORD }}"
```

3. **Add database to vars.yml**:
```yaml
POSTGRES_DATABASES:
  - name: myapp_db
    owner: myapp_user
    enable_pgvector: true
```

4. **Deploy**:
```bash
ansible-playbook server_playbook.yml --tags postgres
```

## 🩺 Health Checks

```bash
# PostgreSQL health
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql pg_isready"

# Version check
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql psql -U postgres -c 'SELECT version();'"

# Connection test
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql psql -U postgres -c 'SELECT current_user;'"

# pgAdmin logs for errors
ansible homelab -i inventory.ini -m shell -a "docker logs pgadmin 2>&1 | grep -i 'permission denied' | tail -10"
```

## 🔄 Backup Commands

```bash
# List backups
ansible homelab -i inventory.ini -m shell -a "ls -lh /cold-storage/database/postgresql/backup/daily" --become

# Manual backup
ansible homelab -i inventory.ini -m shell -a "/cold-storage/database/postgresql/backup/backup-script.sh" --become

# Restore backup
ansible homelab -i inventory.ini -m shell -a "docker exec -i postgresql psql -U postgres < /backups/daily/backup_2025-10-25.sql"
```

## 🌐 Access URLs

- **PostgreSQL**: `postgresql://100.84.146.121:5432`
- **pgAdmin**: `https://pg.homelab` (via Nginx Proxy Manager)

## 🏷️ Tags Reference

| Tag | Scope | Use Case |
|-----|-------|----------|
| `database` | All database tasks | Full deployment |
| `database-management` | All tasks | Same as database |
| `postgres` | PostgreSQL only | Deploy PostgreSQL |
| `pgadmin` | pgAdmin only | Deploy pgAdmin |
| `database-bkp` | Backup config | Setup backups |
| `verify` | Verification only | Health checks |
| `setup` | Storage setup | Create directories |
| `deploy` | Container deployment | Deploy containers |
| `init` | Database init | Create DBs/users |
| `storage` | Storage tasks | Directory creation |

## 📊 Storage Paths

| Path | Owner | Mode | Purpose |
|------|-------|------|---------|
| `/cold-storage/database/postgresql/data` | 70:70 | 0700 | PostgreSQL data |
| `/cold-storage/database/postgresql/init` | 70:70 | 0755 | Init scripts |
| `/cold-storage/database/postgresql/conf` | 70:70 | 0750 | Config files |
| `/cold-storage/database/postgresql/backup` | root:root | 0750 | Backups |
| `/cold-storage/database/postgresql/logs` | 70:70 | 0755 | Logs |
| `/cold-storage/database/pgadmin/data` | 5050:0 | 0750 | pgAdmin data |

## 🐛 Troubleshooting Quick Fixes

### PostgreSQL Won't Start
```bash
# Check ownership
ansible homelab -i inventory.ini -m shell -a "chown -R 70:70 /cold-storage/database/postgresql/data" --become
ansible homelab -i inventory.ini -m shell -a "docker restart postgresql"
```

### pgAdmin Permission Errors
```bash
# Use the fix playbook (idempotent)
ansible-playbook fix_pgadmin_permissions.yml -i inventory.ini
```

### Can't Connect
```bash
# Check pg_hba.conf
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql cat /etc/postgresql/pg_hba.conf | grep -v '^#' | grep -v '^$'"

# Verify Tailscale IP
ansible homelab -i inventory.ini -m shell -a "tailscale ip"
```

### Database Not Created
```bash
# Re-run initialization
ansible-playbook server_playbook.yml --tags postgres,init
```

## 📖 Full Documentation

For complete documentation, see:
- **[DATABASE_MANAGEMENT_ROLE.md](./DATABASE_MANAGEMENT_ROLE.md)** - Complete technical documentation
- **[POSTGRES_DATABASE_MANAGEMENT.md](./POSTGRES_DATABASE_MANAGEMENT.md)** - PostgreSQL usage guide
- **[CONTAINER_USER_IDS.md](./CONTAINER_USER_IDS.md)** - User ID reference
- **[PGADMIN_PERMISSION_FIX.md](./PGADMIN_PERMISSION_FIX.md)** - Permission troubleshooting

---

**Last Updated**: October 25, 2025
