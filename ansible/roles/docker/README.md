# Docker Role

## Description

Installs and configures Docker CE (Community Edition) on Fedora-based systems. Removes conflicting Podman packages, sets up official Docker repositories, configures user permissions, and ensures Docker services are enabled and running.

## Purpose

- Remove conflicting container runtimes (Podman)
- Install Docker CE from official repository
- Configure Docker user group permissions
- Enable Docker and Containerd services
- Verify Docker installation

## Requirements

- Ansible 2.10+
- Target system running Fedora/RHEL-based Linux
- Internet connectivity for package downloads
- Sudo privileges on target system

## Variables

No role-specific variables required. Uses Ansible built-in variables:
- `ansible_user_id` - Current user to add to Docker group

## Dependencies

None

## Tasks Overview

### 1. Cleanup Phase
- Removes `podman-docker` package (Docker CLI wrapper for Podman)
- Removes `podman-compose` package
- Ensures no conflicts with Podman

### 2. Repository Setup
- Downloads official Docker CE repository file
- Adds to `/etc/yum.repos.d/docker-ce.repo`

### 3. Package Installation
- Installs Docker CE engine
- Installs Docker CLI tools
- Installs Containerd runtime
- Installs Docker Buildx plugin
- Installs Docker Compose plugin (v2)

### 4. Permission Configuration
- Creates `docker` system group
- Adds current user to Docker group
- Resets SSH connection to apply group membership

### 5. Service Management
- Enables Docker service (auto-start on boot)
- Starts Docker service
- Enables Containerd service
- Starts Containerd service

### 6. Verification
- Verifies Docker version
- Tests Docker functionality (optional)

## Handlers

### Restart Docker
Restarts the Docker service when packages are updated.

**File**: `handlers/restart-docker.yml`

```yaml
- name: Restart Docker
  ansible.builtin.service:
    name: docker
    state: restarted
```

## Tags

| Tag | Purpose |
|-----|---------|
| `docker` | All Docker-related tasks |
| `infrastructure` | Infrastructure setup |
| `setup` | Initial setup tasks |
| `cleanup` | Podman removal |
| `repository` | Repository configuration |
| `packages` | Package installation |
| `permissions` | User/group setup |
| `services` | Service management |
| `verify` | Verification tasks |
| `never` | Manual execution only (for tests) |

## Usage

### Install Docker
```bash
# Install Docker completely
ansible-playbook server_playbook.yml --tags docker --ask-vault-pass

# Install Docker as part of infrastructure
ansible-playbook server_playbook.yml --tags infrastructure --ask-vault-pass
```

### Run Specific Phases
```bash
# Only install packages
ansible-playbook server_playbook.yml --tags docker,packages --ask-vault-pass

# Setup permissions only
ansible-playbook server_playbook.yml --tags docker,permissions --ask-vault-pass

# Verify installation
ansible-playbook server_playbook.yml --tags docker,verify --ask-vault-pass
```

## Example Playbook

```yaml
- name: Setup Docker
  hosts: homelab
  become: true
  roles:
    - role: docker
      tags:
        - docker
        - infrastructure
        - setup
```

## Post-Installation

After installation, you may need to:

1. **Re-login** to apply Docker group membership
   ```bash
   exit
   # SSH back in
   ```

2. **Verify installation**
   ```bash
   docker --version
   docker run hello-world
   ```

3. **Check service status**
   ```bash
   sudo systemctl status docker
   sudo systemctl status containerd
   ```

## Verification Tasks

The role includes optional verification tasks:

```bash
# Run with verification
ansible-playbook server_playbook.yml --tags docker,verify --ask-vault-pass
```

Verification includes:
- Docker version check
- Hello World container test (tagged with `never`, run explicitly)

## Troubleshooting

### Permission Denied Error
If you get "permission denied" errors after installation:
```bash
# Check group membership
groups

# If 'docker' is not listed, re-login
exit
# SSH back in
```

### Service Won't Start
```bash
# Check Docker service logs
sudo journalctl -u docker.service -n 50

# Check Containerd logs
sudo journalctl -u containerd.service -n 50
```

### Port Conflicts
If Docker fails to start due to port conflicts:
```bash
# Check what's using the Docker socket
sudo lsof /var/run/docker.sock
```

## Files Modified

- `/etc/yum.repos.d/docker-ce.repo` - Docker CE repository
- `/etc/group` - Docker group added
- User's group membership updated

## Security Considerations

- Docker group grants root-equivalent privileges
- Only add trusted users to Docker group
- Consider using rootless Docker for enhanced security
- Regular security updates recommended

## Author

Homelab Automation Project

## License

Personal Use
