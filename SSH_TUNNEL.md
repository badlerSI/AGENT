# URGENT: Need direct access to vLLM — Cloudflare tunnel broken from event wifi

## Situation
Ben is at the event. Cloudflare tunnel returns 404 from every network he's tried. We need a way to reach vLLM on port 9100 directly.

## Option 1: What is Mariposa03's public IP?
Please add the public IP to this file so we can SSH tunnel from Ben's Mac:
```bash
# Run this and put the result here:
curl -4 ifconfig.me
```

**Public IP: _________** (fill this in!)

Once we have the IP, Ben can run:
```bash
ssh -N -L 9100:localhost:9100 admin@<PUBLIC_IP>
```

## Option 2: Use ngrok as backup tunnel
If SSH isn't exposed, install and run ngrok:
```bash
# Install ngrok if not present
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok-v3-stable-linux-amd64.tgz | tar xz -C /usr/local/bin

# Tunnel vLLM port
ngrok http 9100
```
Then paste the ngrok URL into this file.

**ngrok URL: _________** (fill this in!)

## Option 3: Fix the actual Cloudflare tunnel
From the event wifi, ALL of these return 404:
```
curl -4 https://si.tools/agent/chat (POST) → 404
curl -6 https://si.tools/agent/chat (POST) → 404
curl --resolve si.tools:443:172.67.204.119 (POST) → 404
curl --resolve si.tools:443:104.21.77.43 (POST) → 404
```

The tunnel is completely down from external networks. It may have only worked locally before. Please check:
```bash
# Is the tunnel actually connected to Cloudflare edge?
cloudflared tunnel info agent
journalctl -u cloudflared --since "30 min ago" --no-pager
```

## THIS IS TIME SENSITIVE
The workshop is happening RIGHT NOW. Please respond ASAP with whichever option is fastest.
