# URGENT: Connection info for Ben

## Status: Tunnel restarted and working from server side!

Just restarted cloudflared - 4 new connections established to Cloudflare SJC edge.

## Try again first:
```bash
curl -X POST https://si.tools/agent/chat \
  -H "Content-Type: application/json" \
  -d '{"model":"nemotron-super","messages":[{"role":"user","content":"hello"}],"max_tokens":30}'
```

If still 404, the event wifi might be blocking Cloudflare tunnels.

---

## BACKUP: SSH Tunnel

**Public IP: 99.124.159.107**

```bash
# On Mac, run this to create tunnel:
ssh -N -L 9100:localhost:9100 admin@99.124.159.107

# Password: 1864

# Then use localhost:
curl -X POST http://localhost:9100/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"nemotron-super","messages":[{"role":"user","content":"hello"}],"max_tokens":30}'
```

For the workshop code, use:
- **Base URL:** `http://localhost:9100/v1`
- **Chat endpoint:** `http://localhost:9100/v1/chat/completions`
- **Model:** `nemotron-super`

---

## Server test (just ran, works):
```
> POST /agent/chat HTTP/2
< HTTP/2 200
< access-control-allow-origin: *
< server: cloudflare
< via: 1.1 Caddy
```

Tunnel is definitely up. If event wifi blocks it, use the SSH tunnel above.
