# Common Role

## Description

Pre-flight validation role that ensures all prerequisites are met before deployment. This role runs automatically before all other roles to validate the environment and prevent deployment failures.

## Purpose

- Validate required variables are defined
- Ensure storage paths exist and are accessible
- Check system prerequisites (Tailscale, Firewalld)
- Display system information (disk space, etc.)
- Prevent deployment failures due to missing dependencies

## Requirements

- Ansible 2.10+
- Target system running Fedora/RHEL-based Linux
- Sudo privileges on target system

## Variables

### Required (must be defined in group_vars)

| Variable | Description | Example |
|----------|-------------|---------|
| `TAILSCALE_IP_ADDRESS` | IP address assigned by Tailscale | `"100.x.x.x"` |
| `TAILSCALE_ALLOWED_IPS` | List of allowed Tailscale IPs | `["100.x.x.x", "100.x.x.y"]` |
| `COLD_STORAGE_PATH` | Path to persistent storage | `"/cold-storage"` |
| `PI_HOLE_SECURE_WEBPASSWORD` | Pi-hole admin password | `"secure_password"` |
| `NGINX_MANAGER_DATA_STORE_PATH` | NGINX data directory | `"/cold-storage/nginx"` |
| `PI_HOLE_CONFIG_STORAGE_PATH` | Pi-hole config directory | `"/cold-storage/pihole"` |

## Tasks

1. **Validate Variables** - Ensures all required variables are defined
2. **Check Storage Path** - Verifies cold storage path exists
3. **Create Storage Path** - Creates cold storage if missing
4. **Check Disk Space** - Displays available disk space
5. **Check Tailscale** - Verifies Tailscale installation
6. **Check Firewalld** - Ensures firewalld is installed and running
7. **Install Firewalld** - Installs firewalld if missing
8. **Start Firewalld** - Enables and starts firewalld service

## Tags

- `always` - Always runs (cannot be skipped)
- `validate` - Validation tasks
- `storage` - Storage-related checks
- `tailscale` - Tailscale verification
- `firewall` - Firewall checks

## Usage

```bash
# This role runs automatically with any playbook execution
ansible-playbook server_playbook.yml --ask-vault-pass

# Run validation only
ansible-playbook server_playbook.yml --tags validate --ask-vault-pass

# Skip validation (not recommended)
# Note: Cannot skip due to 'always' tag
```

## Example Playbook

```yaml
- name: Server Setup
  hosts: homelab
  become: true
  roles:
    - role: common
      tags:
        - always
        - validate
```

## Validation Output

When successful, you'll see:
```
TASK [common : Validate required variables are defined] *****
ok: [hostname] => {
    "msg": "All required variables are defined"
}

TASK [common : Display available disk space] *****
ok: [hostname] => {
    "msg": "Available disk space on /cold-storage: 800G"
}
```

## Error Handling

If validation fails:
- **Missing Variables**: Playbook stops with error message listing missing variables
- **Missing Storage Path**: Automatically creates the path
- **No Tailscale**: Warning displayed but continues
- **No Firewalld**: Automatically installed and started

## Dependencies

None - This is the base role that runs first

## Author

Homelab Automation Project

## License

Personal Use
