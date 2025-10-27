# Development Role - Executive Summary

**Status**: 📋 Planning Phase - Ready for Implementation  
**Complexity**: Medium  
**Estimated Time**: 4-6 hours  
**Risk Level**: Low

---

## 🎯 What This Role Does

Installs and configures a complete development environment on your homelab server with:

1. **Python 3.x + pip** (for Python/AI development)
2. **Node.js LTS + PNPM** (for JavaScript/TypeScript development)
3. **.NET 9 SDK** (for C# development)

All tools are installed **directly on the host** (not in containers) to support:
- VS Code Remote Development (Remote-SSH)
- JetBrains Gateway (Rider, PyCharm, IntelliJ IDEA)

---

## 🏗️ Architecture Overview

```
development/
├── tasks/
│   ├── main.yml                 # Orchestrator
│   ├── python/
│   │   ├── main.yml             # Python orchestrator
│   │   ├── validate.yml         # Pre-check: Is Python installed?
│   │   ├── install.yml          # Install Python + pip
│   │   └── verify.yml           # Post-check: Does Python work?
│   ├── nodejs/
│   │   ├── main.yml             # Node.js orchestrator
│   │   ├── validate.yml         # Pre-check: Is Node.js installed?
│   │   ├── install-nodejs.yml   # Install Node.js LTS
│   │   ├── install-pnpm.yml     # Install PNPM
│   │   └── verify.yml           # Post-check: Does Node.js work?
│   └── dotnet/
│       ├── main.yml             # .NET orchestrator
│       ├── validate.yml         # Pre-check: Is .NET installed?
│       ├── install.yml          # Install .NET 9 SDK
│       └── verify.yml           # Post-check: Does .NET work?
├── meta/main.yml                # Dependencies
├── vars/main.yml                # Version configuration
└── README.md                    # Documentation
```

**Design Pattern**: Same modular architecture as `database-management` role

---

## ✅ Quality Standards Implemented

### Pre-Checks (validate.yml)
- ✅ Detect existing installations (skip if already installed)
- ✅ Check disk space (>5GB required)
- ✅ Verify internet connectivity
- ✅ Display current versions
- ✅ Set facts for conditional installation

### Installation (install.yml)
- ✅ Use official package repositories (not curl scripts)
- ✅ Version pinning for reproducibility
- ✅ Proper error handling
- ✅ DNF package manager for Fedora
- ✅ Idempotent operations (safe to re-run)

### Success-Checks (verify.yml)
- ✅ Test command execution (`python3 --version`)
- ✅ Validate minimum version requirements
- ✅ Test functional operations (pip list, node -e, dotnet new)
- ✅ Verify PATH availability
- ✅ Assert non-root access

---

## 📦 What Gets Installed

### Python Stack
```bash
# Packages
- python3 (Fedora default, >= 3.9)
- python3-pip (package manager)
- python3-devel (development headers)
- python3-virtualenv (virtual environments)
- python3-setuptools (packaging tools)

# Verification
python3 --version
pip3 --version
pip3 list
```

### Node.js Stack
```bash
# Installation Method
- nvm (Node Version Manager) v0.40.3
- Node.js v22 (LTS) via nvm
- npm (included with Node.js)
- PNPM enabled via Corepack (built into Node.js)

# Installation Commands
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source "$HOME/.nvm/nvm.sh"
nvm install 22
corepack enable pnpm

# Verification
node -v        # Should print "v22.x.x"
npm -v
pnpm -v
nvm --version
```

### .NET Stack
```bash
# Packages (via DNF - Fedora repos)
- dotnet-sdk-9.0 (available in Fedora by default)

# Verification
dotnet --version
dotnet --list-sdks
dotnet --list-runtimes
```

---

## 🚀 How to Use

### Deploy All Development Tools
```bash
ansible-playbook server_playbook.yml --tags development
```

### Deploy Specific Tools
```bash
# Python only
ansible-playbook server_playbook.yml --tags development,python

# Node.js only
ansible-playbook server_playbook.yml --tags development,nodejs

# .NET only
ansible-playbook server_playbook.yml --tags development,dotnet
```

### Verify Installation
```bash
# SSH to server
ssh homelab

# Test Python
python3 --version
pip3 --version

# Test Node.js
node --version
pnpm --version

# Test .NET
dotnet --version
```

---

## 🔧 IDE Remote Development Support

### VS Code Remote-SSH
1. Install "Remote - SSH" extension in VS Code
2. Connect to server: `ssh user@100.84.146.121`
3. VS Code automatically detects Python, Node.js, .NET
4. Extensions work seamlessly

### JetBrains Gateway
1. Open JetBrains Gateway
2. Connect via SSH: `user@100.84.146.121`
3. Select project directory
4. IDE detects installed SDKs automatically

**No additional server-side setup required!** IDEs handle everything via SSH.

---

## 📋 Implementation Phases

### Phase 1: Role Structure (30 min)
Create directory structure, README, meta, vars files

### Phase 2: Python Sub-Task (60 min)
Implement validate → install → verify for Python

### Phase 3: Node.js Sub-Task (90 min)
Implement validate → install-nodejs → install-pnpm → verify

### Phase 4: .NET Sub-Task (60 min)
Implement validate → install → verify for .NET

### Phase 5: Integration Testing (60 min)
Test full role, idempotency, individual tags

### Phase 6: Documentation (30 min)
Create user guides, update main docs

### Phase 7: Quality Assurance (30 min)
Linting, security review, final testing

**Total: 4-6 hours**

---

## ✅ Success Criteria

- [ ] All tools install without errors
- [ ] All verification tests pass
- [ ] Idempotency verified (0 changes on re-run)
- [ ] VS Code Remote-SSH connects successfully
- [ ] JetBrains Gateway connects successfully
- [ ] User can develop in Python, Node.js, and .NET
- [ ] Total deployment time <10 minutes
- [ ] Documentation complete

---

## 🎯 Next Actions

1. **Review this plan** - Approve architecture and approach
2. **Start implementation** - Begin with Phase 1 (role structure)
3. **Test incrementally** - Test each sub-task independently
4. **Document as you go** - Update README and user guides
5. **Deploy to production** - Run full deployment
6. **Verify IDE connectivity** - Test VS Code and JetBrains

---

## 📚 Related Documentation

- **[Full Implementation Plan](DEVELOPMENT_ROLE_IMPLEMENTATION_PLAN.md)** - Detailed technical plan
- **[Database Management Role](DATABASE_MANAGEMENT_ROLE.md)** - Reference architecture
- **[Environment Variables](ENVIRONMENT_VARIABLES.md)** - Configuration reference
- **[Deployment Guide](DEPLOYMENT_GUIDE.md)** - General deployment instructions

---

**Ready to implement?** Review the [Full Implementation Plan](DEVELOPMENT_ROLE_IMPLEMENTATION_PLAN.md) for detailed technical specifications and checklists.
