# Group Variables Organization

## Overview

Variables are organized by **role** and **sub-role** following a hierarchical structure. This makes it easy to find and manage configuration for specific services.

## Structure

```
group_vars/
├── all.yml                               # Global variables (COLD_STORAGE_PATH)
│
├── networking/                           # Networking Role
│   ├── vars.yml                          # Tailscale, Docker network config
│   └── secrets.yml                       # Tailscale IPs (encrypt with ansible-vault)
│
├── nginx-manager/                        # NGINX Proxy Manager Role
│   └── vars.yml                          # Reverse proxy config
│
├── pihole/                               # Pi-hole Role
│   ├── vars.yml                          # DNS and ad-blocking config
│   └── secrets.yml                       # Admin password (encrypt with ansible-vault)
│
├── ollama/                               # Ollama Role (LLM Server)
│   └── vars.yml                          # Model server config
│
├── comfyui/                              # ComfyUI Role (Stable Diffusion)
│   ├── vars.yml                          # Image generation config
│   └── secrets.yml                       # Web UI credentials (encrypt with ansible-vault)
│
├── open-webui/                           # Open WebUI Role (Chat Interface)
│   └── vars.yml                          # Chat UI config
│
└── database-management/                  # Database Management Role
    ├── engine.yml                        # Database engine selection (postgres/sqlserver/oracle)
    ├── internal.yml                      # Computed variables (DO NOT EDIT)
    │
    ├── postgres/                         # PostgreSQL Sub-Role
    │   ├── vars.yml                      # PostgreSQL config (version, storage, performance, backup)
    │   └── secrets.yml                   # PostgreSQL passwords (encrypt with ansible-vault)
    │
    └── pgadmin/                          # pgAdmin Sub-Role
        ├── vars.yml                      # pgAdmin config (port, email, enabled flag)
        └── secrets.yml                   # pgAdmin password (encrypt with ansible-vault)
```

## Variable File Types

### vars.yml
- **Purpose:** Non-sensitive configuration variables
- **Examples:** Versions, paths, ports, performance settings
- **Security:** Can remain unencrypted (but contains no passwords)

### secrets.yml
- **Purpose:** Sensitive data (passwords, API keys, tokens)
- **Examples:** Database passwords, web UI credentials, API tokens
- **Security:** **MUST** be encrypted with ansible-vault
- **Encrypt:** `ansible-vault encrypt group_vars/<role>/secrets.yml`

### engine.yml
- **Purpose:** Top-level engine selection for multi-engine roles
- **Example:** Choose postgres vs sqlserver vs oracle

### internal.yml
- **Purpose:** Computed variables calculated from other variables
- **Security:** DO NOT EDIT - values auto-computed by Ansible

## Role-Based Organization Benefits

### 1. **Clear Ownership**
Each role has its own folder:
```bash
group_vars/ollama/        # Everything for Ollama
group_vars/comfyui/       # Everything for ComfyUI
group_vars/pihole/        # Everything for Pi-hole
```

### 2. **Sub-Role Support**
Complex roles can have sub-roles:
```bash
group_vars/database-management/
├── engine.yml            # Top-level engine choice
├── postgres/             # PostgreSQL sub-role
│   ├── vars.yml
│   └── secrets.yml
└── pgadmin/              # pgAdmin sub-role
    ├── vars.yml
    └── secrets.yml
```

### 3. **Easy Ansible Vault Management**
Encrypt secrets per role:
```bash
# Encrypt PostgreSQL secrets only
ansible-vault encrypt group_vars/database-management/postgres/secrets.yml

# Encrypt all secrets
find group_vars/ -name "secrets.yml" -exec ansible-vault encrypt {} \;
```

### 4. **Scalable for Future Roles**
Adding a new service:
```bash
mkdir -p group_vars/new-service
touch group_vars/new-service/vars.yml
touch group_vars/new-service/secrets.yml
```

### 5. **Clear Documentation Path**
Variables follow the same structure as documentation:
- `docs/ENVIRONMENT_VARIABLES.md#nginx-proxy-manager-role`
- `group_vars/nginx-manager/vars.yml`

## File Naming Convention

| Filename | Purpose | Encrypted? |
|----------|---------|------------|
| `all.yml` | Global variables for all roles | No |
| `vars.yml` | Role-specific configuration | No |
| `secrets.yml` | Role-specific passwords/tokens | **Yes** (ansible-vault) |
| `engine.yml` | Engine selection for multi-engine roles | No |
| `internal.yml` | Auto-computed variables | No |

## Usage Examples

### View Variables for a Role

```bash
# View NGINX Proxy Manager config
cat group_vars/nginx-manager/vars.yml

# View PostgreSQL config
cat group_vars/database-management/postgres/vars.yml

# View encrypted PostgreSQL passwords
ansible-vault view group_vars/database-management/postgres/secrets.yml
```

### Edit Variables

```bash
# Edit non-sensitive config
nano group_vars/ollama/vars.yml

# Edit encrypted secrets
ansible-vault edit group_vars/database-management/postgres/secrets.yml
```

### Add a New Database

1. Edit secrets:
```bash
ansible-vault edit group_vars/database-management/postgres/secrets.yml
```

2. Add password:
```yaml
MYAPP_DB_PASSWORD: "secure_password_here"
```

3. Edit config:
```bash
nano group_vars/database-management/postgres/vars.yml
```

4. Add database definition at the bottom (uncomment example)

## Migration from Old Structure

### Old Structure (Flat)
```
group_vars/
├── homelab.yml           # All variables mixed together
├── secrets.yml           # All secrets mixed together
└── database-management-internal.yml
```

### New Structure (Hierarchical)
```
group_vars/
├── all.yml               # Global only
├── nginx-manager/vars.yml
├── pihole/vars.yml + secrets.yml
├── database-management/postgres/vars.yml + secrets.yml
└── ...
```

### Migration Steps

1. **Keep old files** as backup:
```bash
mv homelab.yml homelab.yml.OLD
mv secrets.yml secrets.yml.OLD
```

2. **Variables now in role-specific files**

3. **Update playbooks** if they reference old variable files

4. **Test deployment** in check mode first:
```bash
ansible-playbook server_playbook.yml --check
```

## Best Practices

### 1. **Always Encrypt Secrets**
```bash
ansible-vault encrypt group_vars/*/secrets.yml
ansible-vault encrypt group_vars/*/*/secrets.yml
```

### 2. **Use Strong Passwords**
```bash
# Generate 32-character password
openssl rand -base64 32
```

### 3. **Document Changes**
Add comments in vars.yml files explaining non-obvious settings

### 4. **Version Control**
- Commit `vars.yml` files
- Commit encrypted `secrets.yml` files (safe with ansible-vault)
- Never commit unencrypted secrets

### 5. **Reference Documentation**
See `docs/ENVIRONMENT_VARIABLES.md` for complete variable reference organized by role

## Troubleshooting

### Ansible Can't Find Variables

**Problem:** `variable 'POSTGRESQL_VERSION' is not defined`

**Solution:** Ansible may not be loading role-specific variable files. Ensure:
1. File structure matches role names exactly
2. Files are named `vars.yml` or `secrets.yml`
3. Run with proper inventory: `ansible-playbook -i inventory.ini server_playbook.yml`

### Can't Decrypt Secrets

**Problem:** `ERROR! Vault password was incorrect`

**Solution:** Provide vault password:
```bash
# Via prompt
ansible-playbook server_playbook.yml --ask-vault-pass

# Via file
ansible-playbook server_playbook.yml --vault-password-file ~/.vault_pass
```

### Variables Not Taking Effect

**Problem:** Changes to vars.yml not reflected in deployment

**Solution:** Ansible caches facts. Clear cache:
```bash
# Delete fact cache
rm -rf ~/.ansible/fact_cache/*

# Re-run with -vvv for debugging
ansible-playbook server_playbook.yml -vvv
```

## See Also

- **[docs/ENVIRONMENT_VARIABLES.md](../../docs/ENVIRONMENT_VARIABLES.md)** - Complete variable reference
- **[docs/POSTGRES_DATABASE_MANAGEMENT.md](../../docs/POSTGRES_DATABASE_MANAGEMENT.md)** - PostgreSQL usage guide
- **[docs/README.md](../../docs/README.md)** - Documentation index

---

**Last Updated:** October 25, 2025
