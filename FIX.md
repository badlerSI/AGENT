# FIX: /agent/* endpoints STILL returning 404 through Cloudflare

## ⚠️ UPDATE: Not fixed yet (tested from Mac at 4:52pm PT)

Despite STATUS.md saying everything works, `POST /agent/chat` still returns **HTTP 404 with an empty body from Cloudflare**. This is a Cloudflare-level 404 (no response body, no content-length) — meaning the cloudflared tunnel is NOT delivering requests to the backend.

**This is NOT a vLLM issue or a path issue.** The request never leaves Cloudflare. The tunnel connection between Cloudflare and this machine appears to be down or misconfigured.

### What we confirmed from the Mac side:
- DNS resolves ✅ (Cloudflare IPs)
- TLS connects ✅ (valid Let's Encrypt cert)
- HTTP/2 POST reaches Cloudflare ✅
- Cloudflare returns 404 with empty body ❌ — **tunnel not delivering**

### Priority fix — check the tunnel RIGHT NOW:
```bash
# 1. Is cloudflared running?
ps aux | grep cloudflared
systemctl status cloudflared

# 2. If it shows the tunnel ID ad57b584-1c21-493b-9ab8-7b8cfa70c003, restart it:
sudo systemctl restart cloudflared

# 3. Watch the logs for errors:
journalctl -u cloudflared -f --no-pager -n 50

# 4. Test locally that Caddy→vLLM works:
curl -X POST http://localhost:80/agent/chat \
  -H "Content-Type: application/json" \
  -d '{"model":"nemotron-super","messages":[{"role":"user","content":"say hello"}],"max_tokens":50}'
```

If cloudflared is not running as a service, find and re-run it:
```bash
cloudflared tunnel run agent
```

**Ben needs this working before 6:30pm PT tonight.** Please fix and verify with:
```bash
curl -X POST https://si.tools/agent/chat \
  -H "Content-Type: application/json" \
  -d '{"model":"nemotron-super","messages":[{"role":"user","content":"say hello"}],"max_tokens":50}'
```

---

## Original Diagnostic Steps

Run these on the Ubuntu machine and report back:

### 1. Is the GPU loaded with the model?
```bash
nvidia-smi
```
Expect to see nemotron-super process using ~45-50GB VRAM.

### 2. What ports are listening?
```bash
ss -tlnp | grep -E '(8000|8080|3000|5000|8888)'
```
We need to find which port the inference server is on.

### 3. Can you hit the API locally?
```bash
# Try common ports — one of these should work
curl http://localhost:8000/v1/models
curl http://localhost:8080/v1/models
curl http://localhost:5000/v1/models
```

### 4. Is cloudflared tunnel running?
```bash
systemctl status cloudflared
# or
ps aux | grep cloudflared
```
If it's down:
```bash
sudo systemctl restart cloudflared
```

### 5. Check the tunnel config
```bash
cat ~/.cloudflared/config.yml
# or
cat /etc/cloudflared/config.yml
```
It should have an ingress rule routing `si.tools/agent/*` to the local inference server port. Example:
```yaml
ingress:
  - hostname: si.tools
    path: /agent/*
    service: http://localhost:8000
  - service: http_status:404
```

## Likely Fixes

### If the inference server is down:
Start it. Check the repo or shell history for the launch command — probably something like:
```bash
# vLLM example
python -m vllm.entrypoints.openai.api_server \
  --model /path/to/nemotron-super \
  --port 8000

# or llama.cpp / TensorRT-LLM — whatever was used
```

### If the server is up but cloudflared is down:
```bash
sudo systemctl restart cloudflared
```

### If the path mapping is wrong:
The server might serve at `/v1/chat/completions` but we need `/agent/chat`. Either:

**Option A** — Add a rewrite in the cloudflared config or a reverse proxy (nginx/Caddy)

**Option B** — If using a simple tunnel, update the agent code to use the actual paths:
```typescript
// In agent.ts and chat.html, change:
const API = "https://si.tools/agent/chat";
// To whatever the real path is, e.g.:
const API = "https://si.tools/v1/chat/completions";
```

### If cloudflared config doesn't exist:
Set up the tunnel:
```bash
cloudflared tunnel login
cloudflared tunnel create agent
cloudflared tunnel route dns agent si.tools
```
Then create `~/.cloudflared/config.yml`:
```yaml
tunnel: agent
credentials-file: /root/.cloudflared/<TUNNEL_ID>.json
ingress:
  - hostname: si.tools
    service: http://localhost:8000
  - service: http_status:404
```
Start it:
```bash
cloudflared tunnel run agent
```

## Once Fixed — Verify
```bash
curl https://si.tools/agent/health
curl https://si.tools/agent/models
curl -X POST https://si.tools/agent/chat \
  -H "Content-Type: application/json" \
  -d '{"model":"nemotron-super","messages":[{"role":"user","content":"say hello"}],"max_tokens":50}'
```

All three should return JSON responses.
