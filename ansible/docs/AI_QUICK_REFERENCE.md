# AI Services Quick Reference
**NVIDIA RTX 3060 12GB GPU-Accelerated Stack**

---

## 🚀 Deployment Commands

```bash
# Deploy all AI services
ansible-playbook server_playbook.yml --tags ai --ask-vault-pass

# Deploy individual services
ansible-playbook server_playbook.yml --tags ollama --ask-vault-pass
ansible-playbook server_playbook.yml --tags comfyui --ask-vault-pass
ansible-playbook server_playbook.yml --tags open-webui --ask-vault-pass

# Deploy everything (recommended first run)
ansible-playbook server_playbook.yml --ask-vault-pass
```

---

## 🔗 Service Access URLs

**After configuring NGINX Proxy Manager:**

| Service | URL | Purpose | Port (Internal) |
|---------|-----|---------|-----------------|
| **Open WebUI** | `https://chat.yourdomain.local` | ChatGPT-like Interface | 3000 |
| **Ollama API** | `https://ollama.yourdomain.local` | LLM API Endpoint | 11434 |
| **ComfyUI** | `https://comfy.yourdomain.local` | Stable Diffusion UI | 8188 |

**Note:** Replace `yourdomain.local` with your actual domain

---

## 📋 NGINX Proxy Manager Configuration

Access: `http://${TAILSCALE_IP}:81`  
Default: `admin@example.com` / `changeme`

### Quick Setup (3 Proxy Hosts):

```
1. chat.yourdomain.local   → open-webui:3000 (Websockets: ON)
2. ollama.yourdomain.local → ollama:11434    (Websockets: ON)
3. comfy.yourdomain.local  → comfyui:8188    (Websockets: ON)
```

---

## 🤖 Download AI Models

```bash
# SSH to server
ssh server

# Ollama Models
docker exec -it ollama ollama pull llama2      # 7B model
docker exec -it ollama ollama pull llama2:13b  # 13B model
docker exec -it ollama ollama pull mistral     # Fast 7B

# List models
docker exec -it ollama ollama list
```

---

## 🔍 Monitoring Commands

```bash
# Check containers
docker ps | grep -E 'ollama|comfyui|open-webui'

# View logs
docker logs ollama
docker logs comfyui
docker logs open-webui

# Monitor GPU
nvidia-smi
watch -n 1 nvidia-smi

# Test GPU in container
docker exec -it ollama nvidia-smi
```

---

## 📊 GPU Performance (RTX 3060 12GB)

| Task | Performance |
|------|-------------|
| **Llama 2 7B** | ~5 tokens/sec |
| **Llama 2 13B** | ~2 tokens/sec |
| **SDXL (1024x1024)** | ~10-15 seconds |
| **SD 1.5 (512x512)** | ~5-8 seconds |

---

## 🛠️ Troubleshooting

```bash
# Restart service
docker restart ollama
docker restart comfyui
docker restart open-webui

# Check network
docker network inspect self-hosted

# Verify GPU access
docker exec -it ollama nvidia-smi

# Rebuild container
ansible-playbook server_playbook.yml --tags ollama --ask-vault-pass
```

---

## 📂 Storage Locations

```
/cold-storage/ai/
├── ollama/models/      # Ollama models (7-13GB each)
├── comfyui/data/       # ComfyUI workspace
├── comfyui/models/     # SD models (2-7GB each)
└── open-webui/data/    # User chats & settings
```

---

## 🔒 Security Architecture

```
Internet ❌ → Tailscale VPN ✅ → NGINX (80/443) → Internal Network
                                      ├─→ open-webui:3000
                                      ├─→ ollama:11434
                                      └─→ comfyui:8188
```

**Zero public exposure** - All access via Tailscale VPN only!

---

## ✅ First-Time Setup Checklist

- [ ] Deploy AI services: `ansible-playbook server_playbook.yml --tags ai --ask-vault-pass`
- [ ] Configure 3 NGINX proxy hosts (chat, ollama, comfy)
- [ ] Download Ollama models: `docker exec -it ollama ollama pull llama2`
- [ ] Access Open WebUI: `https://chat.yourdomain.local`
- [ ] Create admin account (first user)
- [ ] Test chat with Ollama
- [ ] Access ComfyUI: `https://comfy.yourdomain.local`
- [ ] Test image generation
- [ ] Verify GPU usage: `nvidia-smi`

---

**Full Documentation:** See `AI_SERVICES_IMPLEMENTATION.md`

