# Container User ID Reference - Database Management Stack

## Summary

Different container images run as different users for security and compatibility reasons. This document explains the user ID strategy for the database-management stack.

---

## Container User IDs

### PostgreSQL Container
- **Image**: `postgres:16-alpine`
- **Container Process User**: `root` (uid=0) - Docker default
- **Internal Postgres Process**: Switches to `postgres` user (uid=70) after initialization
- **Why root?**: Official PostgreSQL image requires root to:
  - Initialize the database cluster
  - Set up permissions
  - Switch to postgres user for running the server
  - Manage configuration files

**Host Directory Ownership**: `70:70` (postgres user inside container)

```yaml
Directories owned by uid=70:
- {{ POSTGRESQL_DATA_PATH }}      → /var/lib/postgresql/data
- {{ POSTGRESQL_INIT_PATH }}      → /docker-entrypoint-initdb.d
- {{ POSTGRESQL_CONF_PATH }}      → /etc/postgresql (configs)
- {{ POSTGRESQL_LOGS_PATH }}      → /var/log/postgresql
- {{ POSTGRESQL_CERTS_PATH }}     → /etc/postgresql/certs

Directories owned by root:
- {{ POSTGRESQL_BACKUP_PATH }}    → /backups (managed by root for security)
```

---

### pgAdmin Container
- **Image**: `dpage/pgadmin4:latest`
- **Container Process User**: `pgadmin` (uid=5050, gid=0)
- **Why uid=5050?**: pgAdmin4 image hardcodes this user ID
- **Why gid=0?**: Uses root group for compatibility with OpenShift and Kubernetes

**Host Directory Ownership**: `5050:0` (pgadmin user, root group)

```yaml
Directory owned by uid=5050:
- {{ PGADMIN_DATA_PATH }}         → /var/lib/pgadmin
```

---

## Why Different User IDs?

### PostgreSQL (uid=70)
1. **Historical Standard**: Official PostgreSQL images have used uid=70 since early Docker days
2. **Security**: Runs as non-root user after initialization
3. **Compatibility**: Matches system postgres user on many Linux distributions
4. **Image Design**: The entrypoint script starts as root, then drops privileges to uid=70

### pgAdmin (uid=5050)
1. **Arbitrary Choice**: pgAdmin developers chose 5050 to match the default port
2. **Non-Privileged**: High UID ensures it's not a system user
3. **OpenShift Compatible**: gid=0 (root group) allows running in restricted environments
4. **Image Design**: Runs as non-root from the start

---

## The Root Cause of Permission Issues

### What Happened

**Original Setup** (in `setup-storage.yml`):
```yaml
- name: Create pgAdmin storage directory
  ansible.builtin.file:
    path: "{{ PGADMIN_DATA_PATH }}"
    owner: root         # ❌ WRONG - pgAdmin runs as uid=5050
    group: root
```

**Result**: 
- Host directory: `root:root` (0:0)
- Container process: `pgadmin:root` (5050:0)
- **Mismatch** → Permission denied errors

### The Fix

**Corrected Setup**:
```yaml
- name: Create pgAdmin storage directory
  ansible.builtin.file:
    path: "{{ PGADMIN_DATA_PATH }}"
    owner: '5050'       # ✅ CORRECT - matches container user
    group: '0'          # root group (required by pgAdmin image)
```

**Result**:
- Host directory: `5050:root` (5050:0)
- Container process: `pgadmin:root` (5050:0)
- **Match** → No permission errors ✅

---

## Verification Commands

### Check Container User IDs
```bash
# PostgreSQL
ansible homelab -i inventory.ini -m shell -a "docker exec postgresql id"
# Expected: uid=0(root) gid=0(root) ... (runs as root, but postgres process is uid=70)

# pgAdmin
ansible homelab -i inventory.ini -m shell -a "docker exec pgadmin id"
# Expected: uid=5050(pgadmin) gid=0(root) groups=0(root)
```

### Check Host Directory Ownership
```bash
# PostgreSQL directories (should be 70:70)
ansible homelab -i inventory.ini -m shell -a "stat -c '%u:%g %a %n' /cold-storage/database/postgresql/*" --become

# pgAdmin directory (should be 5050:0)
ansible homelab -i inventory.ini -m shell -a "stat -c '%u:%g %a %n' /cold-storage/database/pgadmin/data" --become
```

### Expected Output
```
PostgreSQL:
70:70 700 /cold-storage/database/postgresql/data
70:70 755 /cold-storage/database/postgresql/init
70:70 750 /cold-storage/database/postgresql/conf
0:0   750 /cold-storage/database/postgresql/backup
70:70 755 /cold-storage/database/postgresql/logs

pgAdmin:
5050:0 750 /cold-storage/database/pgadmin/data
```

---

## Best Practices

### When Creating Directories for Containers

1. **Check the Container's User**:
   ```bash
   docker run --rm <image> id
   ```

2. **Match Host Ownership to Container User**:
   ```yaml
   - name: Create directory for container
     ansible.builtin.file:
       path: /host/path
       owner: '<container-uid>'
       group: '<container-gid>'
   ```

3. **Document the User ID**:
   ```yaml
   owner: '5050'  # pgadmin user (matches container uid)
   ```

### Security Considerations

- ✅ **Use numeric UIDs**: More portable than usernames
- ✅ **Use non-root users**: Containers should run as non-root when possible
- ✅ **Restrict permissions**: Use 0750 or 0700 for data directories
- ✅ **Separate backup ownership**: Backups can be owned by root for additional security
- ❌ **Avoid 777 permissions**: Never needed with correct ownership
- ❌ **Don't run everything as root**: Only when absolutely necessary

---

## Troubleshooting

### Permission Denied Errors

**Symptom**: Container logs show "Permission denied" when accessing mounted volumes

**Diagnosis**:
```bash
# 1. Check what user the container runs as
docker exec <container> id

# 2. Check host directory ownership
stat -c '%u:%g %a' /host/path

# 3. Compare - they should match
```

**Fix**:
```bash
# Correct the ownership
chown -R <container-uid>:<container-gid> /host/path

# Restart the container
docker restart <container>
```

### Container Won't Start

**Symptom**: Container exits immediately or crashes in a loop

**Check**:
```bash
# View recent logs
docker logs <container> --tail 50

# Look for permission errors
docker logs <container> 2>&1 | grep -i "permission denied"
```

---

## References

- PostgreSQL Official Image: https://hub.docker.com/_/postgres
- pgAdmin Official Image: https://hub.docker.com/r/dpage/pgadmin4
- Docker Best Practices: https://docs.docker.com/develop/dev-best-practices/
- OpenShift UID/GID Guidelines: https://www.openshift.com/blog/a-guide-to-openshift-and-uids

---

## Change Log

| Date | Change | Reason |
|------|--------|--------|
| 2025-10-25 | Fixed pgAdmin directory ownership from `root:root` to `5050:0` | Permission denied errors preventing container startup |
| 2025-10-25 | Added idempotent permission fix in `pgadmin/deploy.yml` | Ensure permissions are correct on every deployment |
| 2025-10-25 | Documented all container user IDs and ownership requirements | Prevent future permission issues |
