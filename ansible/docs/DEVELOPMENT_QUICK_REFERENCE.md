# Development Tools - Quick Reference Guide

**Role**: `development`  
**Purpose**: Python, Node.js, and .NET development environment  
**Installation Method**: Python & .NET via DNF, Node.js via nvm

---

## 🚀 Quick Deploy

```bash
# Deploy all development tools
ansible-playbook server_playbook.yml --tags development

# Deploy specific tools
ansible-playbook server_playbook.yml --tags development,python
ansible-playbook server_playbook.yml --tags development,nodejs
ansible-playbook server_playbook.yml --tags development,dotnet
```

---

## 📦 What Gets Installed

### Python Stack
```bash
# Verify
python3 --version
pip3 --version

# Usage
python3 script.py
pip3 install package-name
python3 -m venv myenv
```

### Node.js Stack (via nvm)
```bash
# ⚠️ IMPORTANT: Source nvm first!
source ~/.nvm/nvm.sh

# Verify
nvm --version
node -v
npm -v
pnpm -v

# Usage
node app.js
npm install package-name
pnpm install package-name
nvm install 20  # Install different version
nvm use 20      # Switch versions
```

### .NET SDK
```bash
# Verify
dotnet --version
dotnet --list-sdks

# Usage
dotnet new console -n MyApp
dotnet build
dotnet run
```

---

## 🔧 Common Tasks

### Python Virtual Environment
```bash
# Create
python3 -m venv myenv

# Activate
source myenv/bin/activate

# Install packages
pip install flask sqlalchemy

# Deactivate
deactivate
```

### Node.js Project Setup
```bash
# Source nvm
source ~/.nvm/nvm.sh

# Initialize project
pnpm init

# Install dependencies
pnpm install

# Run dev server
pnpm dev
```

### .NET Project Setup
```bash
# Create new project
dotnet new webapi -n MyApi

# Restore packages
dotnet restore

# Build
dotnet build

# Run
dotnet run
```

---

## 🆘 Troubleshooting

### Node.js/PNPM not found
**Problem**: `bash: node: command not found`

**Solution**:
```bash
# Source nvm
source ~/.nvm/nvm.sh

# Or restart shell (nvm auto-loads in new sessions)
exit
ssh homelab
```

### Python pip permission denied
**Problem**: `Permission denied` when installing packages globally

**Solution**: Use virtual environments (recommended)
```bash
python3 -m venv myenv
source myenv/bin/activate
pip install package-name
```

### .NET SDK not found after install
**Problem**: `dotnet: command not found`

**Solution**:
```bash
# Verify package installed
sudo dnf list installed | grep dotnet

# Check PATH
echo $PATH | grep dotnet

# Reinstall if needed
ansible-playbook server_playbook.yml --tags development,dotnet
```

---

## 📍 Installation Locations

| Tool | Location | Config |
|------|----------|--------|
| **Python** | `/usr/bin/python3` | System-wide |
| **pip** | `/usr/bin/pip3` | System-wide |
| **nvm** | `~/.nvm/` | Per-user |
| **Node.js** | `~/.nvm/versions/node/v22.x.x/` | Per-user (via nvm) |
| **PNPM** | `~/.nvm/versions/node/v22.x.x/bin/pnpm` | Per-user (via Corepack) |
| **.NET SDK** | `/usr/share/dotnet/` | System-wide |

---

## 🎯 IDE Setup

### VS Code Remote-SSH
1. Install "Remote - SSH" extension
2. Connect: `ssh user@homelab-ip`
3. Open folder
4. Install extensions (Python, Node.js, C#)
5. ✅ Done! IDE detects all SDKs automatically

### JetBrains Gateway
1. Open JetBrains Gateway
2. New Connection → SSH
3. Enter: `user@homelab-ip`
4. Select IDE (PyCharm, Rider, WebStorm, IntelliJ)
5. Select project folder
6. ✅ Done! IDE detects all SDKs automatically

---

## 📊 Version Management

### Check Installed Versions
```bash
# Python
python3 --version

# Node.js (requires nvm sourced)
source ~/.nvm/nvm.sh
node -v
nvm list

# .NET
dotnet --version
dotnet --list-sdks
```

### Update Versions
```bash
# Python (system updates)
sudo dnf update python3

# Node.js (via nvm)
source ~/.nvm/nvm.sh
nvm install 22  # Reinstall/update v22
nvm alias default 22

# .NET SDK (system updates)
sudo dnf update dotnet-sdk-9.0
```

---

## 🔄 Idempotency

✅ Running the role multiple times is safe:
- Existing installations are detected
- No changes on subsequent runs
- Safe to re-run after system updates

```bash
# Run twice - second run shows 0 changes
ansible-playbook server_playbook.yml --tags development
ansible-playbook server_playbook.yml --tags development  # 0 changes
```

---

## 📚 Related Documentation

- [Full Implementation Plan](DEVELOPMENT_ROLE_IMPLEMENTATION_PLAN.md)
- [Development Role Summary](DEVELOPMENT_ROLE_SUMMARY.md)
- [Environment Variables](ENVIRONMENT_VARIABLES.md#development-role)
- [Main README](../README.md)

---

**Last Updated**: October 25, 2025
