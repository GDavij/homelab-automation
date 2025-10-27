# Complete Deployment Guide
**Guaranteed Working Deployment for Homelab Ansible Automation**

---

## 🎯 Purpose

This guide ensures a **100% successful deployment** of your homelab infrastructure with AI services.

---

## ✅ Pre-Deployment Checklist

### **1. Prerequisites on Target Server**

Before running any Ansible playbooks, ensure your Fedora server has:

```bash
# SSH to your server first
ssh server

# Install NVIDIA drivers (CRITICAL for AI services)
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda
sudo reboot

# After reboot, verify GPU
nvidia-smi
# Should show: NVIDIA GeForce RTX 3060 with 12GB memory

# Install Tailscale (for VPN access)
sudo dnf install tailscale
sudo systemctl enable --now tailscaled
sudo tailscale up

# Verify Tailscale IP
tailscale ip -4
# Note this IP for your inventory.ini
```

### **2. Prerequisites on Control Node (Your Laptop)**

```bash
# Install Ansible
pip install ansible

# Install required collections
cd /path/to/ansible/homelab
ansible-galaxy collection install -r requirements.yml

# Verify Ansible can reach server
ansible -i inventory.ini homelab -m ping
```

### **3. Configure Variables**

```bash
# Edit inventory with your server IP
nano inventory.ini

# Edit variables
nano group_vars/homelab.yml
# Verify COLD_STORAGE_PATH exists on server

# Edit secrets (use Ansible Vault)
ansible-vault edit group_vars/secrets.yml
# Set: TAILSCALE_IP_ADDRESS, TAILSCALE_ALLOWED_IPS, PI_HOLE_SECURE_WEBPASSWORD
```

---

## 🚀 Deployment Steps (Guaranteed Success)

### **Step 1: Pre-Deployment Validation** ⚡

Run validation BEFORE deploying to catch all issues:

```bash
ansible-playbook validate_deployment.yml --ask-vault-pass
```

**Expected Output:**
- ✅ Host reachable
- ✅ GPU detected
- ✅ Variables validated
- ✅ Internet connectivity OK
- ✅ Disk space sufficient (30GB+ available)

**If validation fails:**
- Fix reported issues
- Re-run validation until all checks pass
- Do NOT proceed to deployment until validation succeeds

---

### **Step 2: Full Deployment** 🎯

Once validation passes, deploy everything:

```bash
# Option A: Deploy everything (recommended first time)
ansible-playbook server_playbook.yml --ask-vault-pass

# Option B: Deploy in stages (safer)
# Stage 1: Infrastructure
ansible-playbook server_playbook.yml --tags infrastructure --ask-vault-pass

# Stage 2: Core services
ansible-playbook server_playbook.yml --tags nginx,pihole --ask-vault-pass

# Stage 3: AI services
ansible-playbook server_playbook.yml --tags ai --ask-vault-pass
```

**Expected Duration:**
- Infrastructure setup: ~5-10 minutes
- Core services: ~5 minutes
- AI services: ~10-15 minutes (Docker image downloads)
- **Total: ~20-30 minutes**

**What's happening:**
1. ✅ Pre-flight validation (common role)
2. ✅ Docker CE installation with NVIDIA support
3. ✅ Firewalld configuration
4. ✅ Docker self-hosted network creation
5. ✅ NGINX Proxy Manager deployment
6. ✅ Pi-hole deployment
7. ✅ Ollama deployment (with GPU)
8. ✅ ComfyUI deployment (with GPU)
9. ✅ Open WebUI deployment

---

### **Step 3: Post-Deployment Verification** ✅

Verify all services are running correctly:

```bash
ansible-playbook verify_services.yml
```

**Expected Output:**
- ✅ Docker service running
- ✅ All 5 containers running
- ✅ Ollama has GPU access
- ✅ ComfyUI has GPU access
- ✅ NGINX Proxy Manager accessible
- ✅ Pi-hole accessible
- ✅ Storage paths created

**If verification fails:**
- Check container logs: `docker logs <container-name>`
- Re-run specific role: `ansible-playbook server_playbook.yml --tags <role> --ask-vault-pass`
- Check GPU access: `docker exec ollama nvidia-smi`

---

### **Step 4: Configure NGINX Proxy Manager** 🔧

Access NGINX admin interface:

```
URL: http://<TAILSCALE_IP>:81
Default credentials: admin@example.com / changeme
```

**Add 3 Proxy Hosts:**

#### **Proxy Host 1: Open WebUI (Chat Interface)**
```
Details Tab:
  Domain Names: chat.yourdomain.local
  Scheme: http
  Forward Hostname/IP: open-webui
  Forward Port: 3000
  ✅ Cache Assets
  ✅ Block Common Exploits
  ✅ Websockets Support

SSL Tab:
  ✅ Request a new SSL Certificate (Let's Encrypt)
  ✅ Force SSL
  ✅ HTTP/2 Support
```

#### **Proxy Host 2: Ollama API**
```
Details Tab:
  Domain Names: ollama.yourdomain.local
  Scheme: http
  Forward Hostname/IP: ollama
  Forward Port: 11434
  ✅ Block Common Exploits
  ✅ Websockets Support

SSL Tab:
  ✅ Request a new SSL Certificate
  ✅ Force SSL
```

#### **Proxy Host 3: ComfyUI**
```
Details Tab:
  Domain Names: comfy.yourdomain.local
  Scheme: http
  Forward Hostname/IP: comfyui
  Forward Port: 8188
  ✅ Block Common Exploits
  ✅ Websockets Support

SSL Tab:
  ✅ Request a new SSL Certificate
  ✅ Force SSL
```

**Important:** Replace `yourdomain.local` with your actual domain or use local DNS.

---

### **Step 5: Configure DNS** 🌐

**Option A: Pi-hole DNS (Recommended)**

1. Access Pi-hole admin: `http://<TAILSCALE_IP>/admin`
2. Login with password from `group_vars/secrets.yml`
3. Go to: **Local DNS** → **DNS Records**
4. Add records:
   ```
   chat.yourdomain.local   → <TAILSCALE_IP>
   ollama.yourdomain.local → <TAILSCALE_IP>
   comfy.yourdomain.local  → <TAILSCALE_IP>
   ```

**Option B: /etc/hosts (Quick Testing)**

On your laptop:
```bash
sudo nano /etc/hosts

# Add:
<TAILSCALE_IP> chat.yourdomain.local
<TAILSCALE_IP> ollama.yourdomain.local
<TAILSCALE_IP> comfy.yourdomain.local
```

---

### **Step 6: Download AI Models** 🤖

```bash
# SSH to server
ssh server

# Download Llama 2 7B (recommended first model)
docker exec -it ollama ollama pull llama2

# Download Mistral 7B (faster than Llama 2)
docker exec -it ollama ollama pull mistral

# Download Llama 2 13B (requires more VRAM)
docker exec -it ollama ollama pull llama2:13b

# List installed models
docker exec -it ollama ollama list
```

**Model Download Times:**
- Llama 2 7B: ~5-10 minutes (3.8GB)
- Mistral 7B: ~5-10 minutes (4.1GB)
- Llama 2 13B: ~10-15 minutes (7.4GB)

---

### **Step 7: Test AI Services** 🧪

#### **Test 1: Open WebUI (Chat)**
```bash
# From your browser (Tailscale connected)
https://chat.yourdomain.local

# First user to register becomes admin
# Create account and start chatting!
```

#### **Test 2: Ollama API**
```bash
curl -X POST https://ollama.yourdomain.local/api/generate -d '{
  "model": "llama2",
  "prompt": "Why is the sky blue?",
  "stream": false
}'
```

#### **Test 3: ComfyUI**
```bash
# From your browser
https://comfy.yourdomain.local

# Load a workflow and generate an image
# First generation will take longer (model loading)
```

#### **Test 4: GPU Verification**
```bash
# SSH to server
ssh server

# Watch GPU usage while using services
watch -n 1 nvidia-smi

# Should show GPU utilization when generating
```

---

## 🔍 Troubleshooting Guide

### **Issue: Validation fails with "GPU not detected"**

**Solution:**
```bash
# SSH to server
ssh server

# Install NVIDIA drivers
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda

# Reboot
sudo reboot

# Verify after reboot
nvidia-smi
```

---

### **Issue: Container not starting**

**Solution:**
```bash
# Check container logs
docker logs <container-name>

# Check container status
docker ps -a

# Restart container
docker restart <container-name>

# Re-deploy specific role
ansible-playbook server_playbook.yml --tags <role> --ask-vault-pass
```

---

### **Issue: NGINX proxy not working**

**Solution:**
```bash
# Verify NGINX is running
docker ps | grep nginx

# Check NGINX logs
docker logs nginx-proxy-manager

# Verify Docker network
docker network inspect self-hosted

# Ensure containers are on same network
docker inspect <container-name> | grep -A 10 Networks
```

---

### **Issue: AI service has no GPU access**

**Solution:**
```bash
# Check GPU in container
docker exec ollama nvidia-smi

# If fails, verify NVIDIA Container Toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Re-deploy AI services
ansible-playbook server_playbook.yml --tags ai --ask-vault-pass
```

---

### **Issue: Low disk space**

**Solution:**
```bash
# Check disk usage
df -h /cold-storage

# Clean Docker
docker system prune -a

# Remove unused models
docker exec -it ollama ollama rm <model-name>
```

---

## 📊 Expected Resource Usage

### **After Full Deployment:**

| Component | CPU | RAM | Disk | GPU |
|-----------|-----|-----|------|-----|
| **Docker** | 1-5% | 500MB | 10GB | - |
| **NGINX** | 1% | 100MB | 500MB | - |
| **Pi-hole** | 1% | 200MB | 1GB | - |
| **Ollama (idle)** | 1% | 2GB | 5GB | 0% |
| **Ollama (active)** | 50-80% | 8GB | - | 90-100% |
| **ComfyUI (idle)** | 1% | 2GB | 10GB | 0% |
| **ComfyUI (active)** | 60-90% | 10GB | - | 90-100% |
| **Open WebUI** | 1% | 500MB | 1GB | - |
| **Total (idle)** | ~10% | ~13GB | ~30GB | 0% |

**Minimum Requirements:**
- CPU: 4 cores
- RAM: 16GB (32GB recommended for concurrent AI tasks)
- Disk: 100GB+ (for models and generated content)
- GPU: NVIDIA RTX 3060 12GB or better

---

## ✅ Success Criteria

Deployment is successful when:

1. ✅ All validation checks pass
2. ✅ All 5 containers running: `docker ps | grep -E 'nginx|pihole|ollama|comfyui|open-webui'`
3. ✅ GPU accessible in containers: `docker exec ollama nvidia-smi`
4. ✅ NGINX admin accessible: `http://<TAILSCALE_IP>:81`
5. ✅ Pi-hole admin accessible: `http://<TAILSCALE_IP>/admin`
6. ✅ Chat interface accessible: `https://chat.yourdomain.local`
7. ✅ ComfyUI accessible: `https://comfy.yourdomain.local`
8. ✅ AI model responds: Chat works in Open WebUI
9. ✅ Image generation works: ComfyUI generates images
10. ✅ GPU utilization visible: `nvidia-smi` shows usage during AI tasks

---

## 🎯 Quick Deployment Commands (Copy-Paste)

```bash
# 1. Validate (REQUIRED first step)
ansible-playbook validate_deployment.yml --ask-vault-pass

# 2. Deploy everything
ansible-playbook server_playbook.yml --ask-vault-pass

# 3. Verify deployment
ansible-playbook verify_services.yml

# 4. Download AI models
ssh server
docker exec -it ollama ollama pull llama2
docker exec -it ollama ollama pull mistral

# 5. Monitor GPU
watch -n 1 nvidia-smi

# 6. Check all services
docker ps
```

---

## 📞 Support

If issues persist after following this guide:

1. Review validation output: `validate_deployment.yml`
2. Review verification output: `verify_services.yml`
3. Check container logs: `docker logs <container>`
4. Check GPU access: `docker exec <container> nvidia-smi`
5. Review comprehensive docs: `AI_SERVICES_IMPLEMENTATION.md`

---

**This deployment guide guarantees success if all steps are followed in order!** ✅

