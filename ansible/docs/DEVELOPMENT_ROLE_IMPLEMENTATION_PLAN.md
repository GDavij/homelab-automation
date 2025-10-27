# Development Role - Implementation Plan
**Role Name**: `development`  
**Purpose**: Install and configure development tools for Python, Node.js/PNPM, and .NET development with IDE support  
**Architecture**: Modular sub-tasks following database-management pattern  
**Last Updated**: October 25, 2025

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Requirements Analysis](#requirements-analysis)
3. [Architecture Design](#architecture-design)
4. [Directory Structure](#directory-structure)
5. [Sub-Task Breakdown](#sub-task-breakdown)
6. [Implementation Checklist](#implementation-checklist)
7. [Quality Assurance](#quality-assurance)
8. [Testing Strategy](#testing-strategy)

---

## Overview

### Purpose
Deploy a complete development environment on the homelab server with:
- **Python** (with pip) - For Python development and AI workloads
- **Node.js + PNPM** - For JavaScript/TypeScript development
- **.NET 9 SDK** - For C# development
- **IDE Support** - Optimized for VS Code Remote Development and JetBrains Gateway (Rider, PyCharm, IntelliJ)

### Design Principles
1. **Idempotent Operations**: All tasks can be run multiple times safely
2. **Pre-checks**: Verify prerequisites before installation
3. **Success Checks**: Validate installation after each component
4. **Version Pinning**: Use explicit versions for reproducibility
5. **Modular Architecture**: Separate sub-tasks for each development stack
6. **Quality First**: Comprehensive validation, error handling, and rollback capability

---

## Requirements Analysis

### Functional Requirements

| Requirement | Description | Priority |
|-------------|-------------|----------|
| **FR-01** | Python 3.x installed with pip package manager | CRITICAL |
| **FR-02** | Node.js LTS version installed | CRITICAL |
| **FR-03** | PNPM package manager installed globally | CRITICAL |
| **FR-04** | .NET 9 SDK installed via official Microsoft repository | CRITICAL |
| **FR-05** | All tools accessible in user PATH | CRITICAL |
| **FR-06** | Version verification commands working | HIGH |
| **FR-07** | IDE remote development support (no additional packages needed) | MEDIUM |

### Non-Functional Requirements

| Requirement | Description | Priority |
|-------------|-------------|----------|
| **NFR-01** | Idempotent - safe to re-run without side effects | CRITICAL |
| **NFR-02** | Rollback capability on failure | MEDIUM |
| **NFR-03** | No conflicts with existing system packages | CRITICAL |
| **NFR-04** | Comprehensive logging for troubleshooting | HIGH |

### IDE Support Requirements

| IDE | Connection Method | Requirements |
|-----|------------------|--------------|
| **VS Code** | Remote-SSH extension | SSH access, no server-side setup needed |
| **JetBrains Rider** | JetBrains Gateway | .NET SDK, SSH access |
| **PyCharm** | JetBrains Gateway | Python, SSH access |
| **IntelliJ IDEA** | JetBrains Gateway | JDK (not in scope), SSH access |

**Note**: VS Code and JetBrains Gateway handle remote development setup automatically via SSH. No additional server-side packages required.

---

## Architecture Design

### High-Level Architecture

```
development Role
├── tasks/
│   ├── main.yml                    # Orchestrator (includes sub-tasks)
│   ├── python/
│   │   ├── main.yml                # Python stack orchestrator
│   │   ├── validate.yml            # Pre-checks
│   │   ├── install.yml             # Installation tasks
│   │   └── verify.yml              # Post-installation verification
    ├── nodejs/
        │   ├── main.yml                # Node.js orchestrator
        │   ├── validate.yml            # Check existing nvm/Node.js installation
        │   ├── install-nodejs.yml      # Install nvm → Node.js v22 → PNPM (Corepack)
        │   └── verify.yml              # Verify installation success
│   └── dotnet/
│       ├── main.yml                # .NET SDK orchestrator
│       ├── validate.yml            # Pre-checks
│       ├── install.yml             # .NET 9 SDK installation
│       └── verify.yml              # Post-installation verification
├── handlers/
│   └── main.yml                    # Service restart handlers (if needed)
├── meta/
│   └── main.yml                    # Role dependencies (common, docker)
├── vars/
│   └── main.yml                    # Role-specific variables (versions)
└── README.md                       # Role documentation
```

### Execution Flow

```
ansible-playbook server_playbook.yml --tags development
    │
    ├─▶ [1] Include development/tasks/main.yml
    │       │
    │       ├─▶ [2] Include python/main.yml
    │       │       ├─▶ [2.1] validate.yml (check if Python exists)
    │       │       ├─▶ [2.2] install.yml (install Python + pip)
    │       │       └─▶ [2.3] verify.yml (test Python/pip commands)
    │       │
    │       ├─▶ [3] Include nodejs/main.yml
    │       │       ├─▶ [3.1] validate.yml (check if Node.js exists)
    │       │       ├─▶ [3.2] install-nodejs.yml (install Node.js LTS)
    │       │       ├─▶ [3.3] install-pnpm.yml (install PNPM globally)
    │       │       └─▶ [3.4] verify.yml (test node/npm/pnpm commands)
    │       │
    │       └─▶ [4] Include dotnet/main.yml
    │               ├─▶ [4.1] validate.yml (check if .NET SDK exists)
    │               ├─▶ [4.2] install.yml (install .NET 9 SDK)
    │               └─▶ [4.3] verify.yml (test dotnet command)
```

### Design Decisions

### Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Use DNF for Python/.NET** | Fedora DNF for Python and .NET (stable, integrated with system updates) |
| **Use nvm for Node.js** | Node Version Manager (nvm) for flexible Node.js version management |
| **PNPM via Corepack** | Corepack (built into Node.js) enables PNPM without separate installation |
| **Version Pinning** | Use explicit versions in variables for reproducibility |
| **No Docker Containers** | Development tools installed directly on host (required for IDE remote development) |
| **Separate Sub-Tasks** | Modular approach allows independent testing and troubleshooting |
| **Pre-checks Before Install** | Skip installation if already installed (idempotency) |
| **.NET from Fedora Repos** | Microsoft provides .NET packages in Fedora repos - no additional repository needed |

---

## Directory Structure

```
roles/development/
├── README.md                           # Role documentation
├── meta/
│   └── main.yml                        # Dependencies: common role
├── vars/
│   └── main.yml                        # Version pins and configuration
├── handlers/
│   └── main.yml                        # Handlers (if needed)
└── tasks/
    ├── main.yml                        # Main orchestrator
    ├── python/
    │   ├── main.yml                    # Python orchestrator
    │   ├── validate.yml                # Check existing Python installation
    │   ├── install.yml                 # Install Python + pip
    │   └── verify.yml                  # Verify installation success
    ├── nodejs/
    │   ├── main.yml                    # Node.js + PNPM orchestrator
    │   ├── validate.yml                # Check existing Node.js installation
    │   ├── install-nodejs.yml          # Install Node.js LTS
    │   ├── install-pnpm.yml            # Install PNPM package manager
    │   └── verify.yml                  # Verify installation success
    └── dotnet/
        ├── main.yml                    # .NET SDK orchestrator
        ├── validate.yml                # Check existing .NET installation
        ├── install.yml                 # Install .NET 9 SDK
        └── verify.yml                  # Verify installation success
```

---

## Sub-Task Breakdown

### 1. Python Sub-Task

#### 1.1 Validate (`python/validate.yml`)
- [ ] Check if Python 3 is installed (`python3 --version`)
- [ ] Check if pip is installed (`pip3 --version`)
- [ ] Detect Ansible's Python interpreter (already verified by Ansible)
- [ ] Display current versions if found
- [ ] Set fact `python_needs_installation: true/false`

#### 1.2 Install (`python/install.yml`)
- [ ] Skip if `python_needs_installation == false`
- [ ] Install Python 3 via DNF (`python3`, `python3-pip`, `python3-devel`)
- [ ] Ensure pip is up-to-date (`pip3 install --upgrade pip`)
- [ ] Install common development packages (`python3-virtualenv`, `python3-setuptools`)
- [ ] Create symbolic link `python -> python3` (optional, for compatibility)

#### 1.3 Verify (`python/verify.yml`)
- [ ] Test Python command: `python3 --version`
- [ ] Test pip command: `pip3 --version`
- [ ] Test pip install: `pip3 list` (should not fail)
- [ ] Check Python can import basic modules (`import sys, os, json`)
- [ ] Display installed Python version
- [ ] Display installed pip version
- [ ] Assert version meets minimum requirements (Python >= 3.9)

---

### 2. Node.js Sub-Task

#### 2.1 Validate (`nodejs/validate.yml`)
- [ ] Check if nvm is installed (`command -v nvm`)
- [ ] Check if Node.js is installed (`node --version`)
- [ ] Check if npm is installed (`npm --version`)
- [ ] Check if PNPM is installed (`pnpm --version`)
- [ ] Display current versions if found
- [ ] Set facts:
  - `nvm_needs_installation: true/false`
  - `nodejs_needs_installation: true/false`
  - `pnpm_needs_installation: true/false`

#### 2.2 Install Node.js with PNPM (`nodejs/install-nodejs.yml`)
- [ ] Skip if `nvm_needs_installation == false`
- [ ] Download and install nvm: `curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash`
- [ ] Source nvm in current session: `source "$HOME/.nvm/nvm.sh"`
- [ ] Add nvm to shell profile (automatically done by install script in `~/.bashrc`)
- [ ] Install Node.js LTS (v22): `nvm install 22`
- [ ] Set Node.js 22 as default: `nvm alias default 22`
- [ ] Verify Node.js and npm are installed (`node -v`, `npm -v`)
- [ ] Enable PNPM via Corepack: `corepack enable pnpm`
- [ ] Verify PNPM is installed: `pnpm -v`

#### 2.3 Verify (`nodejs/verify.yml`)
- [ ] Test nvm command: `nvm --version`
- [ ] Test Node.js command: `node -v` (should print "v22.x.x")
- [ ] Test npm command: `npm -v`
- [ ] Test PNPM command: `pnpm -v`
- [ ] Test Node.js can run code: `node -e "console.log('OK')"`
- [ ] Test npm can list packages: `npm list -g --depth=0`
- [ ] Test PNPM can list packages: `pnpm list -g`
- [ ] Display installed versions (nvm, Node.js, npm, PNPM)
- [ ] Assert Node.js version is v22.x
- [ ] Assert version meets minimum requirements (Node.js >= 18.x)

---

### 3. .NET SDK Sub-Task

#### 3.1 Validate (`dotnet/validate.yml`)
- [ ] Check if .NET SDK is installed (`dotnet --version`)
- [ ] Check if dotnet command is in PATH
- [ ] Display current version if found
- [ ] Set fact `dotnet_needs_installation: true/false`

#### 3.2 Install (`dotnet/install.yml`)
- [ ] Skip if `dotnet_needs_installation == false`
- [ ] Add Microsoft package repository (official .NET repo for Fedora)
- [ ] Import Microsoft GPG key
- [ ] Install .NET 9 SDK via DNF (`dotnet-sdk-9.0`)
- [ ] Register dotnet command in PATH
- [ ] Configure dotnet telemetry opt-out (optional, privacy): `export DOTNET_CLI_TELEMETRY_OPTOUT=1`

#### 3.3 Verify (`dotnet/verify.yml`)
- [ ] Test dotnet command: `dotnet --version`
- [ ] Test dotnet SDK list: `dotnet --list-sdks`
- [ ] Test dotnet runtime list: `dotnet --list-runtimes`
- [ ] Test dotnet can create project: `dotnet new console -n test-project -o /tmp/dotnet-test && rm -rf /tmp/dotnet-test`
- [ ] Display installed .NET version
- [ ] Display installed runtimes
- [ ] Assert version meets requirements (.NET SDK >= 9.0)

---

## Implementation Checklist

### Phase 1: Role Structure Setup
- [ ] Create `roles/development/` directory
- [ ] Create `roles/development/README.md`
- [ ] Create `roles/development/meta/main.yml` (dependencies)
- [ ] Create `roles/development/vars/main.yml` (version variables)
- [ ] Create `roles/development/handlers/main.yml`
- [ ] Create `roles/development/tasks/main.yml` (orchestrator)

### Phase 2: Python Sub-Task Implementation
- [ ] Create `roles/development/tasks/python/` directory
- [ ] Implement `python/main.yml` (orchestrator)
- [ ] Implement `python/validate.yml` (pre-checks)
- [ ] Implement `python/install.yml` (installation)
- [ ] Implement `python/verify.yml` (post-checks)
- [ ] Test Python sub-task independently
- [ ] Document Python sub-task in README

### Phase 3: Node.js Sub-Task Implementation
- [ ] Create `roles/development/tasks/nodejs/` directory
- [ ] Implement `nodejs/main.yml` (orchestrator)
- [ ] Implement `nodejs/validate.yml` (pre-checks for nvm, node, npm, pnpm)
- [ ] Implement `nodejs/install-nodejs.yml` (install nvm → Node.js v22 → enable PNPM via Corepack)
- [ ] Implement `nodejs/verify.yml` (post-checks for nvm, Node.js, npm, PNPM)
- [ ] Test Node.js sub-task independently
- [ ] Document Node.js sub-task in README

### Phase 4: .NET Sub-Task Implementation
- [ ] Create `roles/development/tasks/dotnet/` directory
- [ ] Implement `dotnet/main.yml` (orchestrator)
- [ ] Implement `dotnet/validate.yml` (pre-checks)
- [ ] Implement `dotnet/install.yml` (.NET SDK installation)
- [ ] Implement `dotnet/verify.yml` (post-checks)
- [ ] Test .NET sub-task independently
- [ ] Document .NET sub-task in README

### Phase 5: Integration & Testing
- [ ] Add `development` role to `server_playbook.yml`
- [ ] Add tags: `development`, `dev-tools`, `python`, `nodejs`, `dotnet`
- [ ] Update `group_vars/all.yml` or create `group_vars/development/vars.yml`
- [ ] Test full role deployment (fresh system)
- [ ] Test idempotency (run twice, verify no changes on second run)
- [ ] Test individual sub-task tags
- [ ] Verify IDE remote development (VS Code Remote-SSH)
- [ ] Verify JetBrains Gateway connectivity

### Phase 6: Documentation
- [ ] Create `docs/DEVELOPMENT_ROLE.md` (user guide)
- [ ] Update `docs/README.md` (add Development role section)
- [ ] Update `docs/ENVIRONMENT_VARIABLES.md` (add development variables)
- [ ] Create `docs/DEVELOPMENT_QUICK_REFERENCE.md` (command cheat sheet)
- [ ] Update main `README.md` (mention development role)

### Phase 7: Quality Assurance
- [ ] Run `ansible-lint` on all task files
- [ ] Run `yamllint` on all YAML files
- [ ] Verify all tasks have proper `tags`
- [ ] Verify all tasks have proper error handling
- [ ] Verify all tasks are idempotent
- [ ] Add `changed_when` directives where appropriate
- [ ] Add `failed_when` directives for validation tasks
- [ ] Review security implications (no hardcoded secrets)

---

## Quality Assurance

### Pre-Check Requirements (validate.yml files)

| Check | Description | Exit Behavior |
|-------|-------------|---------------|
| **Existing Installation** | Check if tool is already installed | Set fact to skip installation |
| **Version Detection** | Detect currently installed version | Display version, compare with required |
| **PATH Availability** | Verify tool is in user PATH | Warn if not in PATH |
| **Internet Connectivity** | Test package repository access | Fail if no internet (skip if offline mode) |

### Success-Check Requirements (verify.yml files)

| Check | Description | Failure Behavior |
|-------|-------------|------------------|
| **Command Execution** | Test tool --version command | Fail deployment if command not found |
| **Version Validation** | Assert version >= minimum required | Fail if version too old |
| **Functional Test** | Run simple operation (e.g., pip list, node -e) | Fail if tool is broken |
| **PATH Verification** | Verify tool is in user PATH | Fail if not accessible |
| **Permissions Check** | Verify non-root user can run tool | Fail if requires sudo |

### Idempotency Checks

- [ ] Running role twice produces 0 changes on second run
- [ ] Existing installations are detected and preserved
- [ ] No unnecessary package reinstallations
- [ ] No file overwrites unless version changed
- [ ] All tasks use `changed_when: false` for read-only operations

### Error Handling

- [ ] All shell commands have `failed_when` conditions
- [ ] All file operations check for errors
- [ ] Network failures handled gracefully (retry logic)
- [ ] Package installation failures logged clearly
- [ ] Rollback mechanism on critical failures

---

## Testing Strategy

### Unit Testing (Per Sub-Task)

```bash
# Test Python sub-task only
ansible-playbook server_playbook.yml --tags development,python

# Test Node.js sub-task only
ansible-playbook server_playbook.yml --tags development,nodejs

# Test .NET sub-task only
ansible-playbook server_playbook.yml --tags development,dotnet
```

### Integration Testing (Full Role)

```bash
# Test full development role
ansible-playbook server_playbook.yml --tags development

# Test idempotency (should show 0 changes)
ansible-playbook server_playbook.yml --tags development
```

### Acceptance Testing

| Test Case | Expected Result |
|-----------|-----------------|
| **TC-01**: Fresh system deployment | All tools install successfully |
| **TC-02**: Re-run on installed system | 0 changes, all skipped |
| **TC-03**: Python development | `python3 --version`, `pip3 --version` work |
| **TC-04**: Node.js development | `node --version`, `pnpm --version` work |
| **TC-05**: .NET development | `dotnet --version` works, SDK listed |
| **TC-06**: VS Code Remote-SSH | Connect successfully, extensions load |
| **TC-07**: JetBrains Gateway (Rider) | Connect successfully, .NET detected |
| **TC-08**: JetBrains Gateway (PyCharm) | Connect successfully, Python detected |
| **TC-09**: Disk space validation | Fail gracefully if <5GB available |
| **TC-10**: Offline mode | Skip installation with clear error message |

### Post-Deployment Verification Script

```bash
# Create verify_development.yml playbook
- name: Verify Development Tools
  hosts: homelab
  tasks:
    - name: Test Python
      command: python3 --version
    - name: Test pip
      command: pip3 --version
    - name: Test Node.js
      command: node --version
    - name: Test PNPM
      command: pnpm --version
    - name: Test .NET
      command: dotnet --version
```

---

## Variables Configuration

### `roles/development/vars/main.yml`

```yaml
---
# Development Role Version Configuration
# Pin versions for reproducibility

# Python versions (Fedora manages Python version)
PYTHON_MIN_VERSION: "3.9"
PYTHON_PACKAGES:
  - python3
  - python3-pip
  - python3-devel
  - python3-virtualenv
  - python3-setuptools

# Node.js versions (via nvm - Node Version Manager)
NVM_VERSION: "0.40.3"  # nvm version
NVM_INSTALL_URL: "https://raw.githubusercontent.com/nvm-sh/nvm/v{{ NVM_VERSION }}/install.sh"
NODEJS_VERSION: "22"  # LTS version (v22.x)
NODEJS_MIN_VERSION: "18.0.0"
# Note: PNPM is enabled via Corepack (built into Node.js), no separate install needed

# .NET SDK versions
DOTNET_VERSION: "9.0"
DOTNET_MIN_VERSION: "9.0.0"

# Installation flags
DEVELOPMENT_SKIP_PYTHON: false
DEVELOPMENT_SKIP_NODEJS: false
DEVELOPMENT_SKIP_DOTNET: false
```

---

## Success Criteria

### Deployment Success
- [ ] All three sub-tasks complete without errors
- [ ] All verification tests pass
- [ ] All tools accessible in user PATH
- [ ] No conflicts with existing packages
- [ ] Idempotency verified (0 changes on re-run)
- [ ] Total deployment time <10 minutes

### Operational Success
- [ ] User can run Python scripts
- [ ] User can create Python virtual environments
- [ ] User can install packages with pip
- [ ] User can run Node.js applications
- [ ] User can use PNPM for package management
- [ ] User can create .NET projects
- [ ] User can build .NET applications
- [ ] VS Code Remote-SSH connects successfully
- [ ] JetBrains Gateway connects successfully

---

## Next Steps

1. **Review this plan** and approve architecture
2. **Implement Phase 1**: Create role structure
3. **Implement Phase 2**: Python sub-task (test independently)
4. **Implement Phase 3**: Node.js sub-task (test independently)
5. **Implement Phase 4**: .NET sub-task (test independently)
6. **Implement Phase 5**: Integration testing
7. **Implement Phase 6**: Documentation
8. **Implement Phase 7**: Quality assurance
9. **Deploy to production**: Run full deployment
10. **Verify IDE connectivity**: Test VS Code and JetBrains Gateway

---

## Implementation Tracking

Use this checklist to track implementation progress:

### Overall Progress: 0% Complete

- [ ] Phase 1: Role Structure Setup (0/6)
- [ ] Phase 2: Python Sub-Task (0/7)
- [ ] Phase 3: Node.js Sub-Task (0/8)
- [ ] Phase 4: .NET Sub-Task (0/7)
- [ ] Phase 5: Integration & Testing (0/8)
- [ ] Phase 6: Documentation (0/5)
- [ ] Phase 7: Quality Assurance (0/8)

### Estimated Time: 4-6 hours
- Phase 1: 30 minutes
- Phase 2: 60 minutes
- Phase 3: 90 minutes
- Phase 4: 60 minutes
- Phase 5: 60 minutes
- Phase 6: 30 minutes
- Phase 7: 30 minutes

---

**Document Status**: Draft - Ready for Implementation  
**Approval Required**: Yes  
**Implementation Start**: Pending Approval
