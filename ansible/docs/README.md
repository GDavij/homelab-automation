# Homelab Ansible Documentation

Complete documentation for the homelab automation project using Ansible, Docker, and Tailscale.

---

## 📚 Documentation Index

### Getting Started

| Document | Description |
|----------|-------------|
| [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md) | Complete deployment walkthrough |
| [DEPLOYMENT_GUARANTEE.md](DEPLOYMENT_GUARANTEE.md) | Deployment requirements and guarantees |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture and design principles |

### Configuration Reference

| Document | Description |
|----------|-------------|
| **[ENVIRONMENT_VARIABLES.md](ENVIRONMENT_VARIABLES.md)** | **Complete environment variables reference** organized by role and sub-role |
| [AI_QUICK_REFERENCE.md](AI_QUICK_REFERENCE.md) | Quick reference for AI services |
| [DEVELOPMENT_QUICK_REFERENCE.md](DEVELOPMENT_QUICK_REFERENCE.md) | Quick reference for development tools (Python, Node.js, .NET) |

### Service-Specific Guides

| Document | Description |
|----------|-------------|
| [POSTGRES_DATABASE_MANAGEMENT.md](POSTGRES_DATABASE_MANAGEMENT.md) | PostgreSQL database management guide |
| [AI_SERVICES_IMPLEMENTATION.md](AI_SERVICES_IMPLEMENTATION.md) | AI services implementation details |
| [AI_IMPLEMENTATION_SUMMARY.md](AI_IMPLEMENTATION_SUMMARY.md) | AI services summary |
| [DATABASE_MANAGEMENT_PLAN.md](DATABASE_MANAGEMENT_PLAN.md) | Database architecture plan |

---

## 🚀 Quick Start

### 1. Review System Requirements

**Recommended Hardware:**
- CPU: 4+ cores
- RAM: 16GB+ (for AI services)
- Storage: 500GB+ HDD/SSD
- GPU: NVIDIA RTX 3060 12GB (for AI workloads)

**See:** [DEPLOYMENT_GUARANTEE.md](DEPLOYMENT_GUARANTEE.md)

### 2. Configure Environment Variables

All configuration is managed through Ansible variables:

```bash
# Edit user-configurable variables
nano group_vars/homelab.yml

# Edit encrypted secrets
ansible-vault edit group_vars/secrets.yml
```

**See:** [ENVIRONMENT_VARIABLES.md](ENVIRONMENT_VARIABLES.md) for complete reference organized by role.

### 3. Deploy Services

```bash
# Deploy all services
ansible-playbook server_playbook.yml

# Deploy specific services
ansible-playbook server_playbook.yml --tags docker,networking,database
```

**See:** [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md) for detailed instructions.

---

## 📖 Documentation by Role

### Core Infrastructure

#### Docker Role
- **Purpose:** Install and configure Docker CE
- **Configuration:** No user variables required
- **Documentation:** `../roles/docker/README.md`

#### Networking Role
- **Purpose:** Configure Tailscale VPN and Docker networks
- **Configuration:** Auto-detects Tailscale IP
- **Documentation:** `../roles/networking/README.md`
- **Variables:** See [ENVIRONMENT_VARIABLES.md#networking-role](ENVIRONMENT_VARIABLES.md#networking-role)

### Services

#### NGINX Proxy Manager
- **Purpose:** Reverse proxy with SSL/TLS management
- **Ports:** 80 (HTTP), 443 (HTTPS), 81 (Admin)
- **Documentation:** `../roles/nginx-manager/README.md`
- **Variables:** See [ENVIRONMENT_VARIABLES.md#nginx-proxy-manager-role](ENVIRONMENT_VARIABLES.md#nginx-proxy-manager-role)

#### Pi-hole
- **Purpose:** Network-wide ad blocking and DNS
- **Ports:** 53 (DNS), 80 (Admin)
- **Documentation:** `../roles/pihole/README.md`
- **Variables:** See [ENVIRONMENT_VARIABLES.md#pi-hole-role](ENVIRONMENT_VARIABLES.md#pi-hole-role)

### AI Services

#### Ollama
- **Purpose:** Local LLM server (Llama 3, Mistral, etc.)
- **Ports:** 11434 (API)
- **GPU:** NVIDIA RTX 3060 12GB
- **Documentation:** [AI_SERVICES_IMPLEMENTATION.md](AI_SERVICES_IMPLEMENTATION.md)
- **Variables:** See [ENVIRONMENT_VARIABLES.md#ollama-llm-server](ENVIRONMENT_VARIABLES.md#ollama-llm-server)

#### ComfyUI
- **Purpose:** Stable Diffusion image generation
- **Ports:** 8188 (Web UI)
- **GPU:** NVIDIA RTX 3060 12GB
- **Documentation:** [AI_SERVICES_IMPLEMENTATION.md](AI_SERVICES_IMPLEMENTATION.md)
- **Variables:** See [ENVIRONMENT_VARIABLES.md#comfyui-stable-diffusion](ENVIRONMENT_VARIABLES.md#comfyui-stable-diffusion)

#### Open WebUI
- **Purpose:** ChatGPT-like interface for Ollama
- **Ports:** 3000 (Web UI)
- **Documentation:** [AI_SERVICES_IMPLEMENTATION.md](AI_SERVICES_IMPLEMENTATION.md)
- **Variables:** See [ENVIRONMENT_VARIABLES.md#open-webui-chat-interface](ENVIRONMENT_VARIABLES.md#open-webui-chat-interface)

### Database Services

#### PostgreSQL
- **Purpose:** Relational database with pgvector (AI embeddings)
- **Ports:** 5432 (PostgreSQL), 5050 (pgAdmin - optional)
- **Documentation:** [POSTGRES_DATABASE_MANAGEMENT.md](POSTGRES_DATABASE_MANAGEMENT.md)
- **Variables:** See [ENVIRONMENT_VARIABLES.md#postgresql-sub-role](ENVIRONMENT_VARIABLES.md#postgresql-sub-role)

**Database Management Features:**
- ✅ PostgreSQL 16 with pgvector extension
- ✅ Automated backups (daily/weekly/monthly)
- ✅ Multi-database support
- ✅ User permission management (owner/allowed/readonly)
- ✅ Optimized for low RAM usage (512MB)
- ✅ Tailnet-only access (100.64.0.0/10)
- 🔜 SQL Server support (architecture ready)
- 🔜 Oracle support (architecture ready)

### Development Tools

#### Development Role
- **Purpose:** Host-level development tools for remote IDE support (VS Code Remote-SSH, JetBrains Gateway)
- **Technologies:** Python 3.x, Node.js v22 (via nvm), PNPM (via Corepack), .NET 9 SDK
- **Documentation:** `../roles/development/README.md`, [DEVELOPMENT_QUICK_REFERENCE.md](DEVELOPMENT_QUICK_REFERENCE.md)
- **Variables:** See [ENVIRONMENT_VARIABLES.md#development-role](ENVIRONMENT_VARIABLES.md#development-role)

**Development Stack:**
- ✅ **Python:** System-level Python 3.x, pip, virtualenv, development headers
- ✅ **Node.js:** nvm (Node Version Manager) + Node.js v22 + npm + PNPM (via Corepack)
- ✅ **.NET:** .NET 9 SDK with telemetry opt-out
- ✅ **IDE Support:** Optimized for VS Code Remote-SSH and JetBrains Gateway (Rider, PyCharm, IntelliJ)
- ✅ **Idempotent:** Safe to re-run, checks existing installations
- ✅ **Version Management:** nvm for multiple Node.js versions, virtualenv for Python isolation

**Quick Start:**
```bash
# Deploy all development tools
ansible-playbook server_playbook.yml --tags development

# Deploy specific tools
ansible-playbook server_playbook.yml --tags python
ansible-playbook server_playbook.yml --tags nodejs
ansible-playbook server_playbook.yml --tags dotnet

# Verify installation
ssh homelab-server
python3 --version
source ~/.nvm/nvm.sh && node --version && pnpm --version
dotnet --version
```

---

## 🔧 Common Tasks

### Adding a New Database

**See:** [POSTGRES_DATABASE_MANAGEMENT.md](POSTGRES_DATABASE_MANAGEMENT.md#how-to-add-a-new-database)

```yaml
# 1. Add password to secrets.yml
ansible-vault edit group_vars/secrets.yml

MYAPP_DB_PASSWORD: "secure_password_here"

# 2. Define user and database in homelab.yml
POSTGRES_DATABASE_USERS:
  - username: myapp_user
    password: "{{ MYAPP_DB_PASSWORD }}"
    connection_limit: 10

POSTGRES_DATABASES:
  - name: myapp_db
    owner: myapp_user
    enable_pgvector: true

# 3. Deploy
ansible-playbook server_playbook.yml --tags postgres
```

### Managing AI Models

**See:** [AI_SERVICES_IMPLEMENTATION.md](AI_SERVICES_IMPLEMENTATION.md)

```bash
# Download Ollama model
docker exec -it ollama ollama pull llama3

# Access ComfyUI
http://100.64.x.x:8188

# Access Open WebUI
http://100.64.x.x:3000
```

### Setting Up Development Environment

**See:** [DEVELOPMENT_QUICK_REFERENCE.md](DEVELOPMENT_QUICK_REFERENCE.md)

```bash
# Deploy development tools
ansible-playbook server_playbook.yml --tags development

# Connect with VS Code Remote-SSH
# File > Connect to Host > homelab-server

# Create Python virtual environment
ssh homelab-server
python3 -m venv myproject-env
source myproject-env/bin/activate

# Use Node.js with nvm (source required in each session)
source ~/.nvm/nvm.sh
node --version
pnpm install

# Create .NET project
dotnet new console -n MyApp
cd MyApp && dotnet run
```

### Running Backups

**See:** [POSTGRES_DATABASE_MANAGEMENT.md](POSTGRES_DATABASE_MANAGEMENT.md#backup-and-recovery)

```bash
# Manual backup
ansible-playbook server_playbook.yml --tags database-bkp

# Automated backups run daily at 2 AM (configurable)
```

### Viewing Logs

```bash
# View service logs
ssh homelab-server
docker logs <container_name>

# PostgreSQL logs
tail -f /cold-storage/database/postgresql/logs/postgresql.log
```

---

## 📊 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Tailscale VPN (100.64.0.0/10)            │
│                   Encrypted WireGuard Tunnel                │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────────┐
│                   Fedora Server (16GB RAM)                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │ NGINX Proxy   │  │  Pi-hole     │  │  PostgreSQL 16  │ │
│  │ :80, :443     │  │  DNS :53     │  │  :5432          │ │
│  └───────────────┘  └──────────────┘  └─────────────────┘ │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │         AI Services (NVIDIA RTX 3060 12GB)            │ │
│  ├───────────────┬──────────────┬──────────────────────┐ │ │
│  │ Ollama        │  ComfyUI     │  Open WebUI          │ │ │
│  │ LLMs :11434   │  SD :8188    │  Chat :3000          │ │ │
│  └───────────────┴──────────────┴──────────────────────┘ │ │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │     Cold Storage (/cold-storage - HDD/SSD)          │   │
│  │  • AI Models (100GB+)                               │   │
│  │  • PostgreSQL Data (2-3GB+)                         │   │
│  │  • Service Configs & Logs                           │   │
│  │  • Automated Backups                                │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Network Security:**
- All services accessible only via Tailscale VPN
- PostgreSQL restricts to 100.64.0.0/10 CIDR (Tailnet only)
- Public internet access blocked at firewall level
- Encrypted transport via WireGuard

**See:** [ARCHITECTURE.md](ARCHITECTURE.md) for detailed design.

---

## 🔒 Security

### Secret Management

All passwords and sensitive data are stored in encrypted `secrets.yml`:

```bash
# View secrets
ansible-vault view group_vars/secrets.yml

# Edit secrets
ansible-vault edit group_vars/secrets.yml

# Change vault password
ansible-vault rekey group_vars/secrets.yml
```

### Network Isolation

- **Tailscale VPN:** Zero-trust network access
- **Docker Network:** Internal `self-hosted` network for service communication
- **Firewall:** Block public internet, allow Tailnet only
- **PostgreSQL:** `pg_hba.conf` restricts to Tailnet CIDR

### Best Practices

1. ✅ Use strong passwords (32+ characters)
2. ✅ Rotate passwords regularly
3. ✅ Keep Ansible Vault password secure
4. ✅ Review `pg_hba.conf` and firewall rules
5. ✅ Enable backup encryption for sensitive data
6. ✅ Use read-only database users for analytics

**See:** [POSTGRES_DATABASE_MANAGEMENT.md](POSTGRES_DATABASE_MANAGEMENT.md#security-best-practices)

---

## 🐛 Troubleshooting

### Service Not Accessible

```bash
# Check service status
ssh homelab-server
docker ps

# Check Tailscale connection
tailscale status

# View service logs
docker logs <container_name>
```

### Database Connection Issues

```bash
# Test PostgreSQL connection
psql "postgresql://postgres:PASSWORD@100.64.x.x:5432/postgres"

# Check pg_hba.conf
cat /cold-storage/database/postgresql/conf/pg_hba.conf

# View PostgreSQL logs
tail -f /cold-storage/database/postgresql/logs/postgresql.log
```

**See:** [POSTGRES_DATABASE_MANAGEMENT.md](POSTGRES_DATABASE_MANAGEMENT.md#troubleshooting)

### AI Services Not Using GPU

```bash
# Check NVIDIA driver
nvidia-smi

# Check Docker GPU access
docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu22.04 nvidia-smi

# Check container GPU access
docker exec ollama nvidia-smi
```

**See:** [AI_SERVICES_IMPLEMENTATION.md](AI_SERVICES_IMPLEMENTATION.md)

---

## 📝 Configuration Files

### Variable Files (Priority Order)

1. **`../group_vars/homelab.yml`** - User-configurable variables
2. **`../group_vars/database-management-internal.yml`** - Computed internal variables
3. **`../group_vars/secrets.yml`** - Encrypted passwords (ansible-vault)
4. **Command-line:** `-e` extra vars (highest priority)

**See:** [ENVIRONMENT_VARIABLES.md](ENVIRONMENT_VARIABLES.md) for complete reference.

### Playbooks

- **`../server_playbook.yml`** - Main deployment playbook
- **`../validate_deployment.yml`** - Post-deployment validation
- **`../verify_services.yml`** - Service health checks

### Inventory

- **`../inventory.ini`** - Server inventory and connection settings

---

## 🤝 Contributing

When adding new services or features:

1. Follow existing role patterns (see `../roles/comfyui/`, `../roles/nginx-manager/`)
2. Use `group_vars/homelab.yml` for user variables (no `defaults/main.yml`)
3. Store computed variables in `group_vars/<role>-internal.yml`
4. Use `TAILSCALE_IP_ADDRESS` variable (not custom variants)
5. Add comprehensive documentation to `docs/`
6. Update [ENVIRONMENT_VARIABLES.md](ENVIRONMENT_VARIABLES.md) with new variables organized by role

---

## 📞 Support

For issues or questions:

1. Check relevant documentation in this folder
2. Review [ENVIRONMENT_VARIABLES.md](ENVIRONMENT_VARIABLES.md) for configuration
3. Check service logs with `docker logs <container_name>`
4. Validate deployment with `ansible-playbook validate_deployment.yml`

---

## 📄 License

This project is for personal homelab use. Adapt as needed for your environment.

---

**Last Updated:** October 25, 2025
