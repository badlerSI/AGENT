# Fix: IPv6 returns 404 + CORS headers missing

## Problem
The Cloudflare tunnel only works over IPv4. All IPv6 requests to si.tools return 404 from Cloudflare. macOS and browsers prefer IPv6, so the API is unreachable without workarounds (curl -4, /etc/hosts hacks, local proxy).

We also need CORS headers so browsers can call the API directly from any webpage (like a local HTML tool or even hosted on si.tools itself).

## Confirmed from Mac side
```
curl -4 https://si.tools/agent/chat ...   → ✅ 200, model responds
curl -6 https://si.tools/agent/chat ...   → ❌ 404 empty body from Cloudflare
curl    https://si.tools/agent/chat ...   → ❌ 404 (macOS defaults to IPv6)
browser fetch("https://si.tools/agent/chat") → ❌ CORS blocked (no Access-Control-Allow-Origin header)
```

## Fix 1: CORS headers on Caddy

Add CORS headers so browsers can call the API directly. In your Caddyfile, add to the si.tools block:

```
si.tools {
    header Access-Control-Allow-Origin *
    header Access-Control-Allow-Methods "GET, POST, OPTIONS"
    header Access-Control-Allow-Headers "Content-Type"

    @options method OPTIONS
    respond @options 204

    reverse_proxy /agent/* localhost:9100
}
```

Then reload: `sudo systemctl reload caddy`

## Fix 2: IPv6 on cloudflared

The tunnel needs to accept IPv6 connections. Check if cloudflared is binding IPv4-only:

```bash
# Check current tunnel connections
cloudflared tunnel info agent

# In the config, make sure there's no explicit IPv4-only binding
cat ~/.cloudflared/config.yml
```

If the tunnel config has `edge-ip-version: 4`, change it to `auto` or `6`:
```yaml
edge-ip-version: auto
```

Then restart: `sudo systemctl restart cloudflared`

If that doesn't fix it, it may be a Cloudflare edge routing issue. The alternative is to add an AAAA DNS record pointing to the tunnel — check:
```bash
cloudflared tunnel route ip show
```

## Fix 3: Quick alternative — serve the builder from Caddy too

If CORS + IPv6 are both fixed, we can skip localhost entirely. But even simpler — host builder.html on si.tools itself:

```
# Copy builder.html to a served directory
# Update Caddyfile:
si.tools {
    root * /var/www/si.tools
    file_server

    header Access-Control-Allow-Origin *
    header Access-Control-Allow-Methods "GET, POST, OPTIONS"
    header Access-Control-Allow-Headers "Content-Type"

    @options method OPTIONS
    respond @options 204

    reverse_proxy /agent/* localhost:9100
}
```

Then `https://si.tools` serves the chat UI and `https://si.tools/agent/*` hits the model. One URL, maximum flex.

## Verify after fixes
```bash
# IPv6 works
curl -6 -X POST https://si.tools/agent/chat \
  -H "Content-Type: application/json" \
  -d '{"model":"nemotron-super","messages":[{"role":"user","content":"hello"}],"max_tokens":50}'

# CORS preflight works
curl -X OPTIONS https://si.tools/agent/chat \
  -H "Origin: http://localhost:3000" \
  -H "Access-Control-Request-Method: POST" -v 2>&1 | grep access-control
```

## Priority
Ben needs this for an event tonight (starts 6:30pm PT). CORS fix alone would be a big win. IPv6 + hosting builder.html on si.tools is the dream.
