# Networking Role

## Description

Configures firewalld zones and Docker networks for self-hosted services. Creates a custom firewall zone, sets up Docker network isolation, and configures Tailscale access rules for secure remote access.

## Purpose

- Create custom `self-hosted` firewalld zone
- Create Docker network for service isolation
- Configure firewall rules for Docker subnet
- Allow Tailscale IPs to access self-hosted services
- Implement network segmentation and security

## Requirements

- Ansible 2.10+
- Collections:
  - `ansible.posix` (for firewalld module)
  - `community.docker` (for Docker network modules)
- Firewalld installed and running
- Docker installed and running
- Tailscale configured (optional but recommended)

## Variables

### Required

| Variable | Description | Example | Location |
|----------|-------------|---------|----------|
| `TAILSCALE_ALLOWED_IPS` | List of Tailscale IPs allowed access | `["100.x.x.x", "100.x.x.y"]` | `group_vars/secrets.yml` |

### Optional

No optional variables for this role.

## Architecture

```
┌─────────────────────────────────────────┐
│           Internet                      │
└──────────────┬──────────────────────────┘
               │
               │ Tailscale VPN
               │
┌──────────────▼──────────────────────────┐
│        Firewalld Zone: self-hosted      │
│  (Allows: Tailscale IPs + Docker subnet)│
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│    Docker Network: self-hosted          │
│   ┌─────────────┐  ┌──────────────┐    │
│   │   NGINX     │  │   Pi-hole    │    │
│   │   Proxy     │  │     DNS      │    │
│   └─────────────┘  └──────────────┘    │
└─────────────────────────────────────────┘
```

## Tasks Overview

### Main Tasks (`tasks/main.yml`)
1. Include self-hosted configuration tasks

### Self-Hosted Tasks (`tasks/self-hosted.yml`)

#### 1. Create Firewalld Zone
- Creates custom `self-hosted` zone
- Permanent configuration
- Applies immediately

#### 2. Create Docker Network
- Creates `self-hosted` Docker network
- Bridge network type
- Used by all self-hosted services

#### 3. Get Network Information
- Retrieves Docker network details
- Captures subnet information
- Registers for firewall configuration

#### 4. Configure Firewall for Docker
- Adds Docker network subnet to firewall zone
- Allows container-to-container communication
- Maintains network isolation

#### 5. Allow Tailscale Access
- Permits traffic from Tailscale IPs
- Enables remote access to services
- Configures source-based rules

## Network Flow

```
Tailscale Client (100.x.x.x)
    ↓
Firewall Zone: self-hosted (ALLOW)
    ↓
Docker Network: self-hosted (172.x.x.x/16)
    ↓
Containers: NGINX, Pi-hole
```

## Tags

| Tag | Purpose |
|-----|---------|
| `networking` | All networking tasks |
| `firewall` | Firewall configuration |
| `infrastructure` | Infrastructure setup |
| `setup` | Initial setup |
| `docker-network` | Docker network tasks |
| `tailscale` | Tailscale-related rules |

## Usage

### Deploy Network Configuration
```bash
# Setup networking
ansible-playbook server_playbook.yml --tags networking --ask-vault-pass

# Setup as part of infrastructure
ansible-playbook server_playbook.yml --tags infrastructure --ask-vault-pass
```

### Run Specific Tasks
```bash
# Firewall only
ansible-playbook server_playbook.yml --tags firewall --ask-vault-pass

# Docker network only
ansible-playbook server_playbook.yml --tags docker-network --ask-vault-pass

# Tailscale rules only
ansible-playbook server_playbook.yml --tags tailscale --ask-vault-pass
```

## Example Playbook

```yaml
- name: Configure Networking
  hosts: homelab
  become: true
  roles:
    - role: networking
      tags:
        - networking
        - firewall
        - infrastructure
```

## Verification

### Check Firewall Zone
```bash
# List all zones
sudo firewall-cmd --get-zones

# Check self-hosted zone configuration
sudo firewall-cmd --zone=self-hosted --list-all

# Should show:
# - sources: Docker subnet + Tailscale IPs
# - target: default
```

### Check Docker Network
```bash
# List networks
docker network ls

# Inspect self-hosted network
docker network inspect self-hosted

# Should show:
# - Name: self-hosted
# - Driver: bridge
# - Subnet: 172.x.x.x/16
```

### Test Connectivity
```bash
# From Tailscale client, test access
curl http://TAILSCALE_IP:81  # NGINX Proxy Manager
dig @TAILSCALE_IP google.com # Pi-hole DNS
```

## Firewall Zone Configuration

The `self-hosted` zone is configured with:
- **Target**: `default` (accept by default for allowed sources)
- **Sources**: 
  - Docker network subnet (e.g., `172.18.0.0/16`)
  - Tailscale IPs (e.g., `100.x.x.x/32`)
- **Services**: None (access controlled by source)
- **Ports**: None (containers bind to Tailscale IP)

## Security Considerations

1. **Network Isolation**: Containers are isolated in dedicated network
2. **Source Filtering**: Only Tailscale IPs can access services
3. **No Public Exposure**: Services not accessible from internet
4. **Firewall Protection**: All traffic filtered by firewalld

## Troubleshooting

### Firewall Zone Not Working
```bash
# Reload firewall
sudo firewall-cmd --reload

# Check if zone is active
sudo firewall-cmd --get-active-zones

# Verify sources
sudo firewall-cmd --zone=self-hosted --list-sources
```

### Docker Network Issues
```bash
# Remove and recreate network
docker network rm self-hosted
ansible-playbook server_playbook.yml --tags docker-network --ask-vault-pass
```

### Tailscale Access Denied
```bash
# Verify Tailscale IP is in allowed list
# Check group_vars/secrets.yml

# Test firewall rule
sudo firewall-cmd --zone=self-hosted --query-source=100.x.x.x
```

### Service Not Accessible
```bash
# Check if containers are in correct network
docker inspect CONTAINER_NAME | grep NetworkMode

# Should show: "NetworkMode": "self-hosted"
```

## Files Modified

- Firewalld zone: `/etc/firewalld/zones/self-hosted.xml`
- Docker network: `self-hosted` (bridge network)

## Dependencies

- **Docker Role**: Must run after Docker installation
- **Common Role**: Validates prerequisites

## Author

Homelab Automation Project

## License

Personal Use
