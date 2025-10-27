# Pi-hole Role

## Description

Deploys Pi-hole DNS server as a Docker container. Provides network-wide ad blocking, DNS filtering, and DHCP services. Includes web interface for configuration and monitoring.

## Purpose

- Deploy Pi-hole DNS server
- Enable network-wide ad blocking
- Provide DNS filtering and monitoring
- Configure persistent storage for settings and logs
- Bind DNS service to Tailscale IP for security
- Integrate with NGINX Proxy Manager

## Requirements

- Ansible 2.10+
- Collections:
  - `community.docker` (for Docker modules)
- Docker installed and running
- Docker Compose plugin v2
- `self-hosted` Docker network created
- Port 53 (DNS) available
- Sufficient storage space for logs and configuration

## Variables

### Required

| Variable | Description | Example | Location |
|----------|-------------|---------|----------|
| `PI_HOLE_CONFIG_STORAGE_PATH` | Path for Pi-hole config | `"/cold-storage/services/pihole/config"` | `group_vars/homelab.yml` |
| `PI_HOLE_LOGS_STORAGE_PATH` | Path for Pi-hole logs | `"/cold-storage/services/pihole/logs"` | `group_vars/homelab.yml` |
| `PI_HOLE_SECURE_WEBPASSWORD` | Admin web password | `"your_secure_password"` | `group_vars/secrets.yml` (encrypted) |
| `TAILSCALE_IP_ADDRESS` | Tailscale IP for binding | `"100.x.x.x"` | `group_vars/secrets.yml` |
| `COLD_STORAGE_PATH` | Base storage path | `"/cold-storage"` | `group_vars/homelab.yml` |

### Optional

| Variable | Description | Default | Location |
|----------|-------------|---------|----------|
| `PIHOLE_VERSION` | Docker image version | `"latest"` | `group_vars/homelab.yml` |

### Hardcoded in Template

| Variable | Description | Default |
|----------|-------------|---------|
| `DNS1` | Primary upstream DNS | `1.1.1.1` (Cloudflare) |
| `DNS2` | Secondary upstream DNS | `8.8.8.8` (Google) |
| `TZ` | Timezone | `America/Sao_Paulo` |

## Architecture

```
┌────────────────────────────────────────┐
│  Pi-hole Container                     │
│                                        │
│  Ports (bound to Tailscale IP):       │
│  - 53:   DNS (TCP/UDP)                │
│  - 80:   Web Interface (internal)     │
│                                        │
│  Volumes:                              │
│  - /etc/pihole: Configuration         │
│  - /etc/dnsmasq.d: DNS Config         │
│  - /var/log: Logs                     │
│                                        │
│  Network: self-hosted                 │
│                                        │
│  Capabilities:                         │
│  - NET_ADMIN (for DHCP/networking)    │
│  - SYS_NICE (for performance)         │
└────────────────────────────────────────┘
```

## Tasks Overview

### 1. Pull Docker Image
- Downloads `pihole/pihole:latest`
- Ensures latest version available
- Source: Docker Hub

### 2. Ensure Docker Running
- Verifies Docker service is active
- Starts if not running
- Enables for auto-start

### 3. Create Storage Directories
- Creates configuration directory
- Creates logs directory
- Sets proper ownership and permissions (0755)

### 4. Generate Environment File
- Uses Jinja2 template
- Populates variables from inventory
- Creates `.env` file for Docker Compose
- Includes DNS upstream servers

### 5. Deploy with Docker Compose
- Uses Docker Compose v2
- Project name: `pi_hole`
- Applies configuration from compose file

### 6. Health Check (Optional)
- Waits for DNS port to be responsive
- Tests DNS resolution
- Displays status and web interface URL
- Tagged with `verify` and `never`

## Docker Compose Configuration

**File**: `files/docker-compose.yml`

```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "${TAILSCALE_IP}:53:53/tcp"
      - "${TAILSCALE_IP}:53:53/udp"
    environment:
      TZ: 'America/Sao_Paulo'
      FTLCONF_webserver_api_password: "${PIHOLE_WEBPASSWORD}"
      VIRTUAL_HOST: 'hole.homelab'
      WEB_VIRTUAL_HOST: 'hole.homelab'
      VIRTUAL_PORT: 80
      FTLCONF_dns_listeningMode: 'all'
      PIHOLE_DNS: "${DNS1};${DNS2}"
    volumes:
      - '${CONFIGURATION_STORE_PATH}:/etc/pihole'
      - '${CONFIGURATION_STORE_PATH}:/etc/dnsmasq.d'
      - '${LOG_STORE_PATH}:/var/log'
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    restart: unless-stopped
    networks:
      - self-hosted
```

## Environment Template

**File**: `templates/.docker-compose.env.j2`

```jinja
PIHOLE_WEBPASSWORD={{ PI_HOLE_SECURE_WEBPASSWORD }}
CONFIGURATION_STORE_PATH={{ PI_HOLE_CONFIG_STORAGE_PATH }}
LOG_STORE_PATH={{ PI_HOLE_LOGS_STORAGE_PATH }}
TAILSCALE_IP={{ TAILSCALE_IP_ADDRESS }}
DNS1=1.1.1.1
DNS2=8.8.8.8
```

## Tags

| Tag | Purpose |
|-----|---------|
| `pihole` | All Pi-hole tasks |
| `services` | Service deployment |
| `deploy` | Deployment tasks |
| `images` | Image pulling |
| `setup` | Initial setup |
| `storage` | Storage configuration |
| `config` | Configuration tasks |
| `verify` | Health checks |
| `never` | Manual execution only |

## Usage

### Deploy Pi-hole
```bash
# Deploy completely
ansible-playbook server_playbook.yml --tags pihole --ask-vault-pass

# Deploy all services
ansible-playbook server_playbook.yml --tags services --ask-vault-pass

# Deploy with health check
ansible-playbook server_playbook.yml --tags pihole,verify --ask-vault-pass
```

### Run Specific Tasks
```bash
# Setup storage only
ansible-playbook server_playbook.yml --tags pihole,storage --ask-vault-pass

# Pull image only
ansible-playbook server_playbook.yml --tags pihole,images --ask-vault-pass

# Deploy configuration
ansible-playbook server_playbook.yml --tags pihole,deploy --ask-vault-pass
```

## Example Playbook

```yaml
- name: Deploy Pi-hole
  hosts: homelab
  become: true
  roles:
    - role: pihole
      tags:
        - pihole
        - services
        - deploy
```

## Initial Setup

### 1. Access Web Interface

**Direct Access:**
```
URL: http://TAILSCALE_IP/admin
Password: (from PI_HOLE_SECURE_WEBPASSWORD)
```

**Through NGINX Proxy Manager:**
```
URL: http://hole.homelab/admin
```

### 2. First Login
1. Navigate to web interface
2. Enter admin password
3. Complete initial setup wizard

### 3. Configure DNS on Devices

**Option 1: Manual Configuration**
Set device DNS to: `TAILSCALE_IP`

**Option 2: Router Configuration**
Configure router DHCP to provide Pi-hole as DNS server

**Option 3: Tailscale DNS**
Configure in Tailscale admin console:
- DNS → Nameservers
- Add: `TAILSCALE_IP`

## Common Configurations

### Configure Upstream DNS Servers
1. Settings → DNS
2. Upstream DNS Servers
3. Select or add custom servers

### Add Custom Blocklists
1. Group Management → Adlists
2. Add list URL
3. Save
4. Tools → Update Gravity

### Whitelist/Blacklist Domains
1. Whitelist: Domains → Add to whitelist
2. Blacklist: Domains → Add to blacklist

### Enable DHCP (Optional)
1. Settings → DHCP
2. Enable DHCP server
3. Configure IP range
4. Save

## Verification

### Check Container Status
```bash
# List running containers
docker ps | grep pihole

# Check container logs
docker logs pihole

# Check if DNS port is bound
sudo netstat -tulpn | grep :53
```

### Test DNS Resolution
```bash
# From server
dig @TAILSCALE_IP google.com

# From Tailscale client
dig @TAILSCALE_IP google.com

# Test ad blocking
dig @TAILSCALE_IP doubleclick.net
# Should return 0.0.0.0 or NXDOMAIN
```

### Check Web Interface
```bash
# From Tailscale client
curl -I http://TAILSCALE_IP/admin/

# Should return HTTP 200 OK
```

### Monitor Queries
```bash
# View real-time logs
docker logs -f pihole

# Check query log in web interface
# Dashboard → Query Log
```

## Troubleshooting

### Container Won't Start
```bash
# Check logs
docker logs pihole

# Check if port 53 is in use
sudo ss -tulpn | grep :53

# Restart container
docker restart pihole
```

### DNS Not Resolving
```bash
# Test DNS from server
dig @127.0.0.1 google.com

# Check if Pi-hole is listening
sudo netstat -tulpn | grep :53

# Verify firewall allows DNS
sudo firewall-cmd --zone=self-hosted --list-ports
```

### Can't Access Web Interface
```bash
# Verify container is running
docker ps | grep pihole

# Check web server logs
docker exec pihole tail /var/log/lighttpd/error.log

# Test from server
curl http://localhost/admin/
```

### High Query Count / Performance Issues
```bash
# Check log size
du -sh /cold-storage/services/pihole/logs/

# Rotate logs
docker exec pihole pihole -f

# Disable query logging temporarily
docker exec pihole pihole logging off
```

### Wrong Password
```bash
# Reset password
docker exec -it pihole pihole -a -p

# Or set specific password
docker exec -it pihole pihole -a -p newpassword
```

## Persistent Data

### Configuration Directory
```
${PI_HOLE_CONFIG_STORAGE_PATH}/
├── pihole-FTL.db        # Query database
├── gravity.db           # Blocklist database  
├── custom.list          # Custom DNS records
├── setupVars.conf       # Configuration file
└── adlists.list         # Blocklist URLs
```

### Logs Directory
```
${PI_HOLE_LOGS_STORAGE_PATH}/
├── pihole.log           # Main log
├── pihole-FTL.log       # FTL daemon log
└── lighttpd/            # Web server logs
```

## Backup Recommendations

```bash
# Backup configuration
docker exec pihole pihole -a -t

# Manual backup
tar -czf pihole-backup-$(date +%Y%m%d).tar.gz \
  /cold-storage/services/pihole/config/

# Backup with teleporter (from web UI)
# Settings → Teleporter → Backup
```

## Restore from Backup

```bash
# Restore configuration files
tar -xzf pihole-backup-YYYYMMDD.tar.gz -C /

# Restart container
docker restart pihole

# Or use Teleporter in web UI
# Settings → Teleporter → Restore
```

## Performance Optimization

### Database Optimization
```bash
# Optimize FTL database
docker exec pihole sqlite3 /etc/pihole/pihole-FTL.db 'VACUUM;'
```

### Log Rotation
```bash
# Flush logs
docker exec pihole pihole -f

# Configure automatic rotation in web interface
# Settings → System → Logging
```

### Memory Usage
Monitor with:
```bash
docker stats pihole
```

## Security Considerations

1. **Strong Password**: Use strong admin password (stored in vault)
2. **Tailscale Only**: DNS bound to Tailscale IP only
3. **DNSSEC**: Enable in Settings → DNS
4. **Conditional Forwarding**: Configure for local network
5. **Regular Updates**: Keep container image updated
6. **Access Logs**: Monitor for suspicious queries

## Advanced Features

### Conditional Forwarding
For local network DNS resolution:
1. Settings → DNS
2. Enable Conditional Forwarding
3. Set local domain and router IP

### Custom DNS Records
Edit: `/cold-storage/services/pihole/config/custom.list`
```
192.168.x.x myserver.local
192.168.x.y nas.local
```

### Regex Blocking
1. Group Management → Domains
2. Add regex pattern
3. Type: Regex blacklist

## Statistics and Monitoring

- **Dashboard**: Total queries, blocked queries, percentage
- **Query Log**: Real-time query monitoring
- **Long-term Data**: Historical statistics
- **Top Lists**: Most queried domains, blocked domains

## Integration with NGINX Proxy Manager

To access Pi-hole through a friendly domain:
1. In NGINX Proxy Manager, add proxy host
2. Domain: `hole.homelab.local`
3. Forward to: `pihole:80`
4. Enable WebSocket Support
5. Add custom location for `/admin`

## Dependencies

- **Docker Role**: Must run first
- **Networking Role**: `self-hosted` network required
- **Common Role**: Validates prerequisites

## Related Documentation

- [Pi-hole Official Docs](https://docs.pi-hole.net/)
- [Docker Hub Image](https://hub.docker.com/r/pihole/pihole)
- [Pi-hole Discourse](https://discourse.pi-hole.net/)

## Author

Homelab Automation Project

## License

Personal Use
