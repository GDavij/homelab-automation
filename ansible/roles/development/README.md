# Development Role

Ansible role for installing and configuring development tools on the homelab server. Provides Python, Node.js (via nvm), PNPM (via Corepack), and .NET SDK for a complete development environment.

## 📚 Documentation

**Complete documentation is available in the `docs/` folder:**

- **[Development Role Implementation Plan](../../docs/DEVELOPMENT_ROLE_IMPLEMENTATION_PLAN.md)** - Complete technical specification
- **[Development Role Summary](../../docs/DEVELOPMENT_ROLE_SUMMARY.md)** - Executive summary and quick start
- **[Environment Variables Reference](../../docs/ENVIRONMENT_VARIABLES.md#development-role)** - All variables organized by sub-role

## Features

- **Multiple Development Stacks** (modular architecture)
  - **Python 3.x** with pip and development tools
  - **Node.js v22 (LTS)** via nvm (Node Version Manager)
  - **PNPM** via Corepack (built into Node.js)
  - **.NET 9 SDK** via Fedora repositories
- **IDE Support**: Optimized for VS Code Remote-SSH and JetBrains Gateway
- **Version Management**: nvm for flexible Node.js version switching
- **Idempotent Operations**: Safe to re-run without side effects
- **Pre-checks & Post-checks**: Comprehensive validation at each step
- **User-level Installation**: No sudo required for Node.js (via nvm)

## Quick Start

### 1. Deploy All Development Tools

```bash
ansible-playbook server_playbook.yml --tags development
```

### 2. Deploy Specific Tools

```bash
# Python only
ansible-playbook server_playbook.yml --tags development,python

# Node.js only (includes nvm + PNPM)
ansible-playbook server_playbook.yml --tags development,nodejs

# .NET only
ansible-playbook server_playbook.yml --tags development,dotnet
```

### 3. Verify Installation

```bash
# SSH to server
ssh homelab

# Test Python
python3 --version
pip3 --version

# Test Node.js (nvm must be sourced)
source ~/.nvm/nvm.sh
node -v
npm -v
pnpm -v
nvm --version

# Test .NET
dotnet --version
```

## Requirements

### Dependencies
- **Ansible**: 2.10 or higher
- **Roles**: `common` (must run first for validation)
- **Internet Access**: Required for downloading nvm, Node.js, and packages

### Target System
- **OS**: Fedora Server (or RHEL-based)
- **User**: Non-root user with sudo privileges
- **Disk Space**: ~5GB for all SDKs

## What Gets Installed

### Python Stack
- `python3` (Fedora default, >= 3.9)
- `python3-pip` (package manager)
- `python3-devel` (development headers)
- `python3-virtualenv` (virtual environments)
- `python3-setuptools` (packaging tools)

### Node.js Stack
- `nvm` (Node Version Manager) v0.40.3
- `Node.js` v22 (LTS) via nvm
- `npm` (included with Node.js)
- `PNPM` enabled via Corepack

### .NET Stack
- `dotnet-sdk-9.0` (from Fedora repos)

## Architecture

### Directory Structure

```
roles/development/
├── README.md                           # This file
├── meta/
│   └── main.yml                        # Role dependencies
├── vars/
│   └── main.yml                        # Version configuration
├── handlers/
│   └── main.yml                        # Handlers (if needed)
└── tasks/
    ├── main.yml                        # Main orchestrator
    ├── python/
    │   ├── main.yml                    # Python orchestrator
    │   ├── validate.yml                # Pre-checks
    │   ├── install.yml                 # Installation
    │   └── verify.yml                  # Post-checks
    ├── nodejs/
    │   ├── main.yml                    # Node.js orchestrator
    │   ├── validate.yml                # Pre-checks
    │   ├── install-nodejs.yml          # nvm + Node.js + PNPM
    │   └── verify.yml                  # Post-checks
    └── dotnet/
        ├── main.yml                    # .NET orchestrator
        ├── validate.yml                # Pre-checks
        ├── install.yml                 # .NET SDK installation
        └── verify.yml                  # Post-checks
```

### Execution Flow

1. **Python Sub-Task**: validate → install → verify
2. **Node.js Sub-Task**: validate → install (nvm → Node.js → PNPM) → verify
3. **.NET Sub-Task**: validate → install → verify

Each sub-task is independent and can be run separately using tags.

## IDE Remote Development

### VS Code Remote-SSH

1. Install "Remote - SSH" extension in VS Code
2. Connect to server: `ssh user@homelab-ip`
3. VS Code automatically detects Python, Node.js, .NET
4. Extensions work seamlessly

### JetBrains Gateway (Rider, PyCharm, IntelliJ)

1. Open JetBrains Gateway
2. Connect via SSH: `user@homelab-ip`
3. Select project directory
4. IDE detects installed SDKs automatically

**No additional server-side setup required!**

## Configuration

### Version Variables

Edit `vars/main.yml` to customize versions:

```yaml
# Python (managed by Fedora)
PYTHON_MIN_VERSION: "3.9"

# Node.js (via nvm)
NVM_VERSION: "0.40.3"
NODEJS_VERSION: "22"  # LTS
NODEJS_MIN_VERSION: "18.0.0"

# .NET SDK
DOTNET_VERSION: "9.0"
DOTNET_MIN_VERSION: "9.0.0"
```

## Idempotency

This role is fully idempotent:
- Existing installations are detected and preserved
- Running the role twice produces 0 changes on the second run
- Version checks prevent unnecessary reinstallations

## Troubleshooting

### Node.js/PNPM Not Found After Installation

**Problem**: `node: command not found` or `pnpm: command not found`

**Solution**: Source nvm in your shell:
```bash
source ~/.nvm/nvm.sh
```

Or restart your shell session. The nvm install script automatically adds sourcing to `~/.bashrc`.

### Python pip3 Permission Denied

**Problem**: `Permission denied` when running `pip3 install`

**Solution**: Use virtual environments:
```bash
python3 -m venv myenv
source myenv/bin/activate
pip install package-name
```

### .NET SDK Not Found

**Problem**: `dotnet: command not found`

**Solution**: Verify installation and PATH:
```bash
sudo dnf list installed | grep dotnet
echo $PATH
```

## Tags

- `development` - Deploy all development tools
- `python` - Python sub-task only
- `nodejs` - Node.js sub-task only (includes nvm + PNPM)
- `dotnet` - .NET sub-task only
- `validate` - Run validation tasks only
- `verify` - Run verification tasks only

## Example Playbook

```yaml
---
- name: Configure Development Environment
  hosts: homelab
  become: true
  roles:
    - role: development
      tags:
        - development
        - dev-tools
```

## License

Part of the Homelab Ansible Automation project.

## Author

Homelab Automation Project - October 2025
