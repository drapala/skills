---
name: field-opsec-tooling-setup
description: Use before a field operation in hostile-state environment — step-by-step installation of all tools required by field-opsec-technical, pentest-untraceability, and redteam-infrastructure-opsec skills. Run on a clean machine before travel.
---

# Field OPSEC — Tooling Setup

## Overview

Install everything **before entering the target country**. Downloading security tools in-country is a red flag on ISP logs. This doc covers the full toolchain from scratch on a clean machine.

Assumes: macOS (host) + burner hardware being prepared.

---

## Phase 0 — Host Machine Prep (macOS, before imaging burner)

```bash
# Install brew if not present:
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Monero wallet (for anonymous VPS payment):
brew install --cask monero-wallet

# Install Tor Browser (for anonymous VPS registration):
brew install --cask tor-browser

# Install GPG (for encrypting findings):
brew install gnupg pinentry-mac

# Install mat2 (EXIF/metadata removal):
brew install mat2

# Install ExifTool (backup metadata removal):
brew install exiftool

# Install OnionShare (anonymous file transfer):
brew install --cask onionshare

# Install VeraCrypt (hidden volumes):
brew install --cask veracrypt

# Install Mullvad VPN:
brew install --cask mullvad-vpn

# Install 1Password (with travel mode):
brew install --cask 1password
```

---

## Phase 1 — Tails USB (Primary OS)

```bash
# 1. Download Tails image + GPG signature:
# https://tails.boum.org/install/download/ — do this over Tor Browser
# Current stable: tails-amd64-X.X.img

# 2. Verify GPG signature (MANDATORY — never skip):
gpg --keyserver keys.openpgp.org --recv-key A490 D0F4 D311 A415 3E2B B7CA DBB8 02B2 58AC D84F
gpg --verify tails-amd64-X.X.img.sig tails-amd64-X.X.img
# Must say: "Good signature from Tails developers"

# 3. Flash to USB (replace disk2 with your USB device):
diskutil list                          # identify USB
diskutil unmountDisk /dev/disk2
sudo dd if=tails-amd64-X.X.img of=/dev/rdisk2 bs=16m status=progress
# Or use balenaEtcher (GUI): brew install --cask balenaetcher

# 4. Test boot on target hardware before travel
```

---

## Phase 2 — GrapheneOS (Burner Phone)

Requires: Google Pixel (6, 7, or 8 series recommended)

```bash
# Option A: Web installer (easiest, requires Chrome/Chromium):
# Navigate to: https://grapheneos.org/install/web
# Follow on-screen steps — unlocks bootloader, flashes, re-locks

# Option B: CLI:
brew install android-platform-tools

# Download factory image from grapheneos.org/releases
# Verify SHA256:
shasum -a 256 -c factory.zip.sha256sum

# Unlock bootloader (erases device):
adb reboot bootloader
fastboot flashing unlock

# Flash:
unzip factory.zip
cd factory/
./flash-all.sh  # or use fastboot commands directly

# Re-lock bootloader after flash:
fastboot flashing lock
```

### Post-Install GrapheneOS Config
```
Settings → Security → Screen lock → Strong PIN (no biometrics)
Settings → Security → Duress password → Set (wipes on entry)
Settings → Network → Private DNS → dns.mullvad.net
Settings → Apps → Special app access → Sensors Off (optional)
Install F-Droid: https://f-droid.org
From F-Droid install: Orbot, Briar, SimpleX Chat
From Aurora Store (F-Droid): Signal, ProtonMail, Mullvad VPN
```

---

## Phase 3 — VPS Provisioning (Anonymous)

```bash
# Step 1: Get Monero
# Buy from peer-to-peer exchange (LocalMonero) with cash or gift card
# Or: buy Bitcoin on exchange → swap to XMR via SideShift (no KYC)

# Step 2: Open Tor Browser
# Navigate to VPS provider .onion or clearnet via Tor:
# - 1984.hosting (Iceland, accepts XMR)
# - njalla.com (privacy-focused, accepts XMR)

# Step 3: Register with throwaway email
# Create ProtonMail account accessed only via Tor (never your real ProtonMail)
# Format: randomstring@protonmail.com

# Step 4: Pay with Monero, provision Ubuntu 22.04 LTS (minimal)

# Step 5: SSH in via Tor (MANDATORY on first connection):
ssh -o ProxyCommand="nc -X 5 -x 127.0.0.1:9050 %h %p" root@<vps-ip>

# Step 6: Harden VPS:
apt update && apt upgrade -y
apt install -y ufw tor proxychains4 nmap curl wget git python3 socat

ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw enable

# Disable logging to persistent storage:
systemctl stop rsyslog && systemctl disable rsyslog
mount -t tmpfs tmpfs /var/log  # logs in RAM only
```

---

## Phase 4 — Pentest Tools on VPS

```bash
# Install on VPS (not your machine):

# Nmap:
apt install -y nmap

# Nuclei (fast vulnerability scanner):
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
# Or:
wget https://github.com/projectdiscovery/nuclei/releases/latest/download/nuclei_linux_amd64.zip
unzip nuclei_linux_amd64.zip && mv nuclei /usr/local/bin/

# SQLMap:
apt install -y sqlmap

# ffuf (web fuzzer):
go install github.com/ffuf/ffuf/v2@latest

# httpx (HTTP probe):
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest

# Feroxbuster (dir busting):
curl -sL https://raw.githubusercontent.com/epi052/feroxbuster/main/install-nix.sh | bash

# Metasploit (if needed for authorized exploitation):
curl https://raw.githubusercontent.com/rapid7/metasploit-omnibus/master/config/templates/metasploit-framework-wrappers/msfupdate.erb > msfinstall
chmod +x msfinstall && ./msfinstall

# Sliver C2 (open source, authorized use only):
curl https://sliver.sh/install | sudo bash
# Lightweight, no license, better opsec than Cobalt Strike for field ops
```

---

## Phase 5 — Covert Channel Tools

```bash
# On VPS AND your machine:

# iodine (DNS tunnel):
apt install -y iodine             # VPS (server side)
brew install iodine               # macOS (client side)

# ptunnel-ng (ICMP tunnel):
apt install -y ptunnel-ng         # VPS
brew install ptunnel-ng           # macOS (or compile from source)

# stunnel (wrap anything in TLS):
apt install -y stunnel4           # VPS
brew install stunnel              # macOS

# socat (general purpose relay):
apt install -y socat              # VPS
brew install socat                # macOS
```

---

## Phase 6 — Dead Man's Switch Setup

```bash
# On VPS:
cat > /opt/deadman.sh << 'EOF'
#!/bin/bash
CHECKIN="/tmp/.alive"
touch -t $(date -d '2 hours ago' +%Y%m%d%H%M) /tmp/.ref 2>/dev/null || \
  touch -A -0200 /tmp/.ref 2>/dev/null

if [ ! -f "$CHECKIN" ] || [ "$CHECKIN" -ot /tmp/.ref ]; then
    shred -u /opt/c2/* /root/.bash_history 2>/dev/null
    # Optional: self-destroy via provider API
    # curl -X DELETE -H "Authorization: Bearer $PROVIDER_TOKEN" \
    #      https://api.provider.com/v2/droplets/$DROPLET_ID
fi
EOF
chmod +x /opt/deadman.sh

# Run every 30min:
echo "*/30 * * * * root /opt/deadman.sh" >> /etc/crontab

# From your machine, keepalive via SSH:
# Add to cron: */20 * * * * ssh user@vps "touch /tmp/.alive"
```

---

## Phase 7 — Encryption & Exfil Tools

```bash
# GPG keypair (do on air-gapped machine or vault-vm, never online):
gpg --full-generate-key
# Type: RSA 4096, expiry: 1 year, real name: use pseudonym

# Export public key to share with recipient:
gpg --export --armor <key-id> > pubkey.asc

# Test encrypt/decrypt:
echo "test" | gpg --encrypt --armor --recipient <key-id> | gpg --decrypt

# OnionShare (installed in Phase 0):
# Usage:
onionshare --receive                    # creates drop point
onionshare findings.tar.gz.gpg          # sends file

# Archive and encrypt findings:
tar czf findings.tar.gz ./results/
gpg --encrypt --armor --recipient <recipient-key-id> findings.tar.gz
shred -u findings.tar.gz               # delete plaintext
```

---

## Phase 8 — Verification Before Travel

```bash
# Run all checks from burner machine before departure:

echo "=== VPN ==="
# Connect Mullvad, enable kill-switch
curl https://ifconfig.me    # must show Mullvad exit IP

echo "=== DNS leak ==="
# Visit dnsleaktest.com — must show only Mullvad DNS

echo "=== Tor ==="
# Open Tor Browser → check.torproject.org → must show "Congratulations"

echo "=== VPS ==="
ssh user@<vps-ip> "curl https://ifconfig.me"  # VPS own IP
# Access via Tor:
ssh -o ProxyCommand="nc -X 5 -x 127.0.0.1:9050 %h %p" user@<vps-ip> "echo ok"

echo "=== proxychains ==="
proxychains4 curl https://ifconfig.me  # must show VPS IP, not yours

echo "=== Kill-switch ==="
# Disconnect Mullvad manually → curl must fail immediately

echo "=== LUKS nuke ==="
# (Test on non-production device first — this is destructive)

echo "=== GrapheneOS duress ==="
# (Test on non-production device — wipes device)

echo "=== Dead man's switch ==="
ssh user@<vps-ip> "ls -la /tmp/.alive"  # must exist and be recent
```

---

## Tool Summary Table

| Tool | Purpose | Install |
|------|---------|---------|
| Tails OS | Primary anonymous OS | USB flash |
| GrapheneOS | Secure phone OS | Pixel device |
| Mullvad VPN | Entry VPN layer | brew cask |
| Tor Browser | Tor access + anonymous browsing | brew cask |
| obfs4/meek bridges | Tor censorship bypass | Built into Tails |
| VeraCrypt | Hidden encrypted volumes | brew cask |
| GPG | File encryption + signing | brew |
| mat2 / ExifTool | Metadata removal from media | brew |
| OnionShare | Anonymous file transfer | brew cask |
| 1Password (travel mode) | Credential management | brew cask |
| proxychains4 | Route any tool through proxy chain | apt (VPS) |
| nmap | Network scanning | apt (VPS) |
| nuclei | Vulnerability scanning | go install (VPS) |
| sqlmap | SQL injection testing | apt (VPS) |
| ffuf | Web fuzzing | go install (VPS) |
| Sliver | C2 framework (authorized) | script (VPS) |
| iodine | DNS tunnel fallback | brew + apt |
| ptunnel-ng | ICMP tunnel fallback | brew + apt |
| stunnel | TLS wrapping | brew + apt |
| socat | General relay | brew + apt |
