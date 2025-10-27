# NGINX Proxy Manager Role

## Description

Deploys NGINX Proxy Manager as a Docker container. Provides a web-based interface for managing reverse proxy configurations, SSL certificates, and access lists. Handles automatic Let's Encrypt SSL certificate generation and renewal.

## Purpose

- Deploy NGINX Proxy Manager container
- Configure persistent storage for data and certificates
- Bind services to Tailscale IP for security
- Provide web UI for proxy management
- Enable SSL/TLS certificate management
- Facilitate access to self-hosted services

## Requirements

- Ansible 2.10+
- Collections:
  - `community.docker` (for Docker modules)
- Docker installed and running
- Docker Compose plugin v2
- `self-hosted` Docker network created
- Sufficient storage space for certificates and data

## Variables

### Required

| Variable | Description | Example | Location |
|----------|-------------|---------|----------|
| `NGINX_MANAGER_DATA_STORE_PATH` | Path for NGINX data | `"/cold-storage/services/nginx-proxy/"` | `group_vars/homelab.yml` |
| `NGINX_MANAGER_CERTIFICATES_STORE_PATH` | Path for SSL certificates | `"/cold-storage/services/nginx-proxy/certificates"` | `group_vars/homelab.yml` |
| `TAILSCALE_IP_ADDRESS` | Tailscale IP for binding | `"100.x.x.x"` | `group_vars/secrets.yml` |

### Optional

| Variable | Description | Default | Location |
|----------|-------------|---------|----------|
| `NGINX_PROXY_MANAGER_VERSION` | Docker image version | `"latest"` | `group_vars/homelab.yml` |

## Architecture

```
┌────────────────────────────────────────┐
│  NGINX Proxy Manager Container        │
│                                        │
│  Ports (bound to Tailscale IP):       │
│  - 80:   HTTP traffic                 │
│  - 443:  HTTPS traffic                │
│  - 81:   Admin Web UI                 │
│                                        │
│  Volumes:                              │
│  - data: Configuration & Database     │
│  - letsencrypt: SSL Certificates      │
│                                        │
│  Network: self-hosted                 │
└────────────────────────────────────────┘
```

## Tasks Overview

### 1. Pull Docker Image
- Downloads `jc21/nginx-proxy-manager:latest`
- Ensures latest version available
- Source: Docker Hub

### 2. Ensure Docker Running
- Verifies Docker service is active
- Starts if not running
- Enables for auto-start

### 3. Create Storage Directories
- Creates data directory
- Creates certificates directory
- Sets proper ownership and permissions (0755)

### 4. Generate Environment File
- Uses Jinja2 template
- Populates variables from inventory
- Creates `.env` file for Docker Compose

### 5. Deploy with Docker Compose
- Uses Docker Compose v2
- Project name: `nginx_proxy_manager`
- Applies configuration from compose file

### 6. Health Check (Optional)
- Waits for web UI to be accessible
- Tests HTTP 200 response on port 81
- Retries up to 12 times with 5-second delay
- Tagged with `verify` and `never`

## Docker Compose Configuration

**File**: `files/docker-compose.yml`

```yaml
services:
  nginx-proxy-manager:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    
    ports:
      - "${TAILSCALE_IP}:80:80"     # HTTP
      - "${TAILSCALE_IP}:443:443"   # HTTPS
      - "${TAILSCALE_IP}:81:81"     # Admin UI
      
    volumes:
      - '${DATA_STORE_PATH}/data:/data'
      - '${CERTIFICATES_STORE_PATH}/letsencrypt:/etc/letsencrypt'
      
    networks:
      - self-hosted
```

## Environment Template

**File**: `templates/.docker-compose.env.j2`

```jinja
DATA_STORE_PATH={{ NGINX_MANAGER_DATA_STORE_PATH }}
CERTIFICATES_STORE_PATH={{ NGINX_MANAGER_CERTIFICATES_STORE_PATH }}
TAILSCALE_IP={{ TAILSCALE_IP_ADDRESS }}
```

## Tags

| Tag | Purpose |
|-----|---------|
| `nginx` | All NGINX Proxy Manager tasks |
| `services` | Service deployment |
| `deploy` | Deployment tasks |
| `images` | Image pulling |
| `setup` | Initial setup |
| `storage` | Storage configuration |
| `config` | Configuration tasks |
| `verify` | Health checks |
| `never` | Manual execution only |

## Usage

### Deploy NGINX Proxy Manager
```bash
# Deploy completely
ansible-playbook server_playbook.yml --tags nginx --ask-vault-pass

# Deploy all services
ansible-playbook server_playbook.yml --tags services --ask-vault-pass

# Deploy with health check
ansible-playbook server_playbook.yml --tags nginx,verify --ask-vault-pass
```

### Run Specific Tasks
```bash
# Setup storage only
ansible-playbook server_playbook.yml --tags nginx,storage --ask-vault-pass

# Pull image only
ansible-playbook server_playbook.yml --tags nginx,images --ask-vault-pass

# Deploy configuration
ansible-playbook server_playbook.yml --tags nginx,deploy --ask-vault-pass
```

## Example Playbook

```yaml
- name: Deploy NGINX Proxy Manager
  hosts: homelab
  become: true
  roles:
    - role: nginx-manager
      tags:
        - nginx
        - services
        - deploy
```

## Initial Setup

### 1. Access Web UI
```
URL: http://TAILSCALE_IP:81
```

### 2. Default Credentials
```
Email:    admin@example.com
Password: changeme
```

### 3. First Login
1. Login with default credentials
2. Change email and password immediately
3. Configure admin settings

### 4. Add Proxy Hosts
1. Go to "Proxy Hosts"
2. Click "Add Proxy Host"
3. Configure domain and forwarding
4. Enable SSL with Let's Encrypt

## Common Proxy Configurations

### Example: Pi-hole Web Interface
```
Domain Name: hole.homelab.local
Scheme: http
Forward Hostname: pihole
Forward Port: 80
SSL: Request Let's Encrypt Certificate (optional)
```

### Example: Another Service
```
Domain Name: service.homelab.local
Scheme: http
Forward Hostname: service-container-name
Forward Port: 8080
```

## Verification

### Check Container Status
```bash
# List running containers
docker ps | grep nginx-proxy-manager

# Check container logs
docker logs nginx-proxy-manager

# Check if ports are bound
sudo netstat -tulpn | grep :81
```

### Test Web UI Access
```bash
# From Tailscale client
curl -I http://TAILSCALE_IP:81

# Should return HTTP 200 OK
```

### Check Storage
```bash
# Verify directories created
ls -la /cold-storage/services/nginx-proxy/
ls -la /cold-storage/services/nginx-proxy/certificates/
```

## Troubleshooting

### Container Won't Start
```bash
# Check logs
docker logs nginx-proxy-manager

# Check if ports are in use
sudo ss -tulpn | grep -E ':(80|443|81)'

# Restart container
docker restart nginx-proxy-manager
```

### Can't Access Web UI
```bash
# Verify container is running
docker ps | grep nginx-proxy-manager

# Check firewall
sudo firewall-cmd --zone=self-hosted --list-all

# Test from server
curl http://localhost:81
```

### SSL Certificate Issues
```bash
# Check certificate storage
ls -la /cold-storage/services/nginx-proxy/certificates/letsencrypt/

# Check container logs for SSL errors
docker logs nginx-proxy-manager | grep -i ssl

# Ensure ports 80 and 443 are accessible for ACME challenge
```

### Storage Permission Issues
```bash
# Fix ownership
sudo chown -R $(whoami):$(whoami) /cold-storage/services/nginx-proxy/

# Fix permissions
sudo chmod -R 755 /cold-storage/services/nginx-proxy/
```

## Persistent Data

### Data Directory
```
${DATA_STORE_PATH}/data/
├── database.sqlite        # Configuration database
├── logs/                  # Access and error logs
├── nginx/                 # NGINX configurations
└── custom_ssl/           # Custom SSL certificates
```

### Certificates Directory
```
${CERTIFICATES_STORE_PATH}/letsencrypt/
├── live/                  # Active certificates
├── archive/               # Certificate history
└── renewal/              # Renewal configurations
```

## Backup Recommendations

```bash
# Backup data directory
tar -czf npm-backup-$(date +%Y%m%d).tar.gz \
  /cold-storage/services/nginx-proxy/data/

# Backup certificates
tar -czf npm-certs-backup-$(date +%Y%m%d).tar.gz \
  /cold-storage/services/nginx-proxy/certificates/
```

## Security Considerations

1. **Change Default Password**: Immediately after first login
2. **Tailscale Only**: Services bound to Tailscale IP only
3. **SSL Certificates**: Use Let's Encrypt for all domains
4. **Access Lists**: Configure access lists in NGINX Proxy Manager
5. **Regular Updates**: Keep image updated (`docker pull`)

## Performance Tips

- Use SSD storage for database if possible
- Monitor logs for errors: `docker logs -f nginx-proxy-manager`
- Keep container updated regularly
- Use caching for frequently accessed sites

## Dependencies

- **Docker Role**: Must run first
- **Networking Role**: `self-hosted` network required
- **Common Role**: Validates prerequisites

## Related Documentation

- [NGINX Proxy Manager Official Docs](https://nginxproxymanager.com/guide/)
- [Docker Hub Image](https://hub.docker.com/r/jc21/nginx-proxy-manager)

## Author

Homelab Automation Project

## License

Personal Use
