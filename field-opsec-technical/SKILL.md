---
name: field-opsec-technical
description: Use when operating in hostile-state environments as journalist, security researcher, or field operative — covers device hardening, OS selection, network anonymization, RF/wireless threat mitigation, and storage encryption for adversaries with ISP-level surveillance and physical inspection capability.
---

# Field OPSEC — Technical Layer

## Overview

Layered technical defense for operating against state-level adversaries with ISP monitoring, IMSI catchers, and physical device inspection capability. Assumes: no trust in local infrastructure, possible physical compromise, need for plausible deniability.

---

## Threat Model Reference

| Threat | Capability Level |
|--------|-----------------|
| ISP traffic interception | High — assume full packet capture |
| IMSI catchers (fake cell towers) | Medium — common in authoritarian states |
| WiFi monitoring | High — passive + active |
| Physical device inspection | High — border crossings, raids |
| Bluetooth/RF sniffing | Low-Medium |
| Power analysis / side-channel | Low (advanced labs only) |

---

## Devices

### Hardware Selection
- **Burner laptop**: ThinkPad X series (open firmware, well-supported by Tails/Qubes). New or factory-wiped. Paid cash.
- **Burner phone**: Android with unlockable bootloader (Pixel recommended for GrapheneOS). No Google account. SIM inserted only when needed.
- **USB drives x2**: One for Tails OS, one for encrypted data exfil.
- Never bring: daily-use laptop, phone with biometrics tied to real identity.

### Physical Hardening
- Webcam: physical tape or mechanical cover
- Microphone: internal mic disabled in BIOS if possible; external only when needed
- BIOS password set — prevents cold-boot attack via USB at border
- Disable Wake-on-LAN, Bluetooth in BIOS if not needed
- Faraday bag for phone when not in use (blocks IMSI catcher passive tracking)

### Border Crossing Protocol
- **Power off completely** before checkpoint (not sleep/hibernate — RAM retention). "Before First Unlock" (BFU) state is the only safe state: Secure Enclave doesn't release keys, keychain inaccessible even with correct PIN.
- **iPhone: power off completely** — do NOT use SOS mode. SOS mode only disables biometrics; device stays on, remains accessible via PIN under coercion. Power off = BFU.
- **Android (GrapheneOS)**: reboot → do not unlock after reboot. BFU state.
- Enable full-disk encryption BEFORE travel — can't be forced to encrypt at border
- Use travel mode in 1Password — hides sensitive vaults
- If compelled to unlock: only show the decoy OS/profile (see OS section)

---

## Operating System

### Primary: Tails OS
```
Use case: daily operations, browsing, comms
Boot from USB, leaves zero trace on hardware
All traffic routed through Tor automatically
Encrypted persistent volume (optional, passphrase-protected)
```
- Download at tails.boum.org, verify GPG signature before use
- Never boot Tails on a machine that will later be inspected — USB is the artifact, not the laptop

### Secondary: Qubes OS (if persistent install needed)
```
Use case: pentesting work requiring persistence
Compartmentalized VMs — network VM, work VM, vault VM isolated
Compromise of one VM doesn't reach others
```
- Security domain separation: `pentest-vm` never touches `journalist-vm`
- `vault-vm` (no network): GPG keys, sensitive docs only

### Phone: GrapheneOS
- Install before travel
- No Google Play Services (use F-Droid + Aurora Store)
- Separate profiles: "public" profile (empty, for border) + "work" profile (Signal, Tor Browser)
- USB restricted mode on

### Avoid
- Windows (telemetry, harder to audit, no live-boot anonymity)
- macOS (iCloud sync risks, closed firmware)
- Standard Android/iOS for anything sensitive

---

## Network

### Stack (ordered, use all layers together)

```
You → VPN (Mullvad) → Tor (obfs4 bridge) → Destination
```

**Why this order:** VPN hides from ISP that you're using Tor. Tor hides destination from VPN. Bridges hide Tor usage from deep packet inspection.

> **Myanmar-specific warning:** MPT (Myanmar Posts and Telecommunications) and Mytel (joint venture with the military) use DPI equipment (Sandvine documented in the region). WireGuard and standard obfs4 may be fingerprintable. **Prefer Shadowsocks or V2Ray/VLESS** as VPN transport in Myanmar — they present as random HTTPS traffic and have much better evasion against Asian DPI than obfs4.

### VPN
- **Mullvad**: accepts Monero/cash, no account required (account number only), audited no-log
- Configure kill-switch: if VPN drops, no traffic leaks
- **Transport preference for Myanmar**: Shadowsocks > obfs4 > WireGuard > OpenVPN
- **Install and test BEFORE entering country** — downloading VPN in-country is a red flag

### Shadowsocks Setup (Myanmar-preferred DPI bypass)
```bash
# Server (VPS):
apt install shadowsocks-libev
# /etc/shadowsocks-libev/config.json:
# { "server":"0.0.0.0", "server_port":443, "password":"<strong>", "method":"chacha20-ietf-poly1305" }
systemctl enable --now shadowsocks-libev

# Client (Tails/Linux):
apt install shadowsocks-libev
ss-local -s <vps-ip> -p 443 -l 1080 -k <password> -m chacha20-ietf-poly1305
# Then configure apps to use SOCKS5 127.0.0.1:1080
```

### Tor + Bridge Configuration
```bash
# In Tor Browser or Tails:
# Use obfs4 bridges — traffic looks like random HTTPS
# Obtain bridges at: bridges.torproject.org (do before travel)
# Alternative bridge type: meek-azure (traffic looks like Azure CDN) — good fallback
```
- For Tails: configured in Tor Connection assistant at boot
- Save bridge lines to encrypted USB before travel — can't fetch in-country if Tor is blocked

### WiFi
- Connect via VPN immediately — never browse direct on local WiFi
- Use `MAC address randomization` — enabled by default in Tails/GrapheneOS
- Prefer mobile data over hotel/café WiFi (fewer co-located adversaries)
- Disable WiFi when not in use (passive probe requests reveal device history)

### Cellular / SIM
- Local SIM = registered to passport in Myanmar — assume government has it **and has real-time access via MPT/Mytel**
- **Do not use local SIM for GPS navigation** — it creates a timestamped location history tied to your passport. Use **OsmAnd or Maps.me with offline maps** (Myanmar maps available in both, download before travel)
- Local SIM: only for non-sensitive voice calls when strictly necessary; keep off otherwise
- Sensitive comms: WiFi only, through Shadowsocks/VPN+Tor stack
- Enable airplane mode before entering sensitive locations — IMSI catcher requires active radio connection

### DNS
- Use encrypted DNS (DoH/DoT) — prevents ISP from seeing domains even without VPN
- Tails handles this automatically
- On GrapheneOS: set Private DNS to `dns.mullvad.net`

---

## RF / Wireless Threats

| Threat | Mitigation |
|--------|-----------|
| IMSI catcher (tracks location via IMEI/IMSI) | Faraday bag when not actively using; airplane mode in sensitive areas |
| WiFi probe requests (passive device tracking) | MAC randomization enabled; WiFi off when idle |
| Bluetooth tracking | BT disabled in BIOS/settings; never pair with untrusted devices |
| NFC skimming | Disable NFC when not in use |
| Shoulder surfing / optical TEMPEST | Privacy screen filter on laptop; position back to wall |

### Faraday Bag Usage
- Phone in faraday bag = invisible to all cell towers and IMSI catchers
- Verify seal: call phone while in bag — if it rings, bag is inadequate
- Alternatives: airplane mode (software, can be bypassed by compromised OS) — bag is hardware guarantee

---

## Storage & Encryption

### Encryption Stack
```
Full disk encryption (LUKS/FileVault) — first layer
VeraCrypt hidden volume — plausible deniability
GPG-encrypted archives — for exfil
```

### VeraCrypt Hidden Volume
- Outer volume: innocuous files, decoy password
- Hidden volume: sensitive data, real password
- Hidden volume is cryptographically invisible to forensic analysis of the device **without you present**
- **CRITICAL LIMITATION:** This does NOT protect you if you are present and under physical coercion. An adversary who can apply violence does not need to break the crypto — they apply pressure to you. VeraCrypt hidden volume protects against device seizure without you, not against interrogation.
- Additional forensic risk: if adversary images the disk before coercion, they can detect a hidden volume by comparing before/after sector changes after you provide the outer password. Never unlock the outer volume after seizure.
- Tutorial: veracrypt.fr/en/hidden-volume.html

### Sensitive Data Handling
- **Never store locally** if avoidable — use encrypted remote drop
- **OnionShare**: share files directly over Tor, no third party
  ```bash
  onionshare --receive  # creates .onion drop point
  ```
- **Wipe after exfil**: `shred -u file` works on **HDDs only**. On **SSDs (NVMe/SATA SSD)** — which includes most ThinkPad X series — `shred` does NOT guarantee secure deletion due to wear leveling. Reliable wipe on SSD requires either: (a) full-disk encryption from the start (then deleting the key is sufficient), or (b) `hdparm --security-erase` (ATA Secure Erase) for the whole drive. For individual files on SSD, only FDE from the start makes deletion reliable.

### Photo/Video Metadata
```bash
# Remove EXIF before sending (location, device fingerprint)
mat2 photo.jpg          # Tails has mat2 pre-installed
exiftool -all= photo.jpg  # Alternative
```

---

## Pentest Operations (Authorized)

### Network Segregation
- Pentest traffic must never touch personal/journalist identity
- Use dedicated VM (Qubes `pentest-vm`) with separate VPN exit node
- Consider VPS pivot:
  ```
  You → VPN → VPS in neutral country → target NGO system
  ```
  VPS paid with Monero. This way target logs show VPS IP, not yours.

### Authorization Documentation
- Carry signed scope letter: target systems, IP ranges, time window
- Have emergency contact at NGO who can confirm authorization immediately
- Store copy in encrypted cloud (Proton Drive) accessible from any device

### Tool Discretion
- Terminal-based tools preferred over GUI (less visually incriminating)
- Rename suspicious binaries if needed for cover
- Keep tools in encrypted container, unmounted when not in use

---

## Communications

| Tool | Use Case | Notes |
|------|----------|-------|
| Signal | Real-time comms | Disappearing messages ≤24h; no SMS fallback |
| Briar | No-internet comms | P2P via BT/WiFi mesh; works without infrastructure |
| SimpleX Chat | Metadata-resistant | No user IDs, better than Signal for anonymity |
| ProtonMail | Email | Access via Tor only |
| SecureDrop | Source → journalist | .onion only, no account needed |
| OnionShare | File transfer | No third party, ephemeral |

**Avoid:** WhatsApp, Telegram (no E2E by default), standard SMS, any platform requiring phone number tied to real identity.

---

## Quick Reference — Pre-Departure Checklist

```
HARDWARE
[ ] Burner laptop (wiped, FDE enabled, BIOS password set)
[ ] Burner phone (GrapheneOS, decoy profile configured)
[ ] Tails USB (verified GPG signature)
[ ] Faraday bag (tested)
[ ] Privacy screen filter

NETWORK
[ ] VPN installed and tested (Mullvad)
[ ] Tor bridges saved offline (obfs4/meek)
[ ] Kill-switch enabled on VPN

IDENTITY
[ ] 1Password travel mode enabled (sensitive vaults hidden)
[ ] Biometrics disabled — device powered off completely (BFU state) before any checkpoint. DO NOT use iPhone SOS mode.
[ ] Decoy OS/profile prepared

COMMS
[ ] Signal configured, disappearing messages on
[ ] Briar installed (offline fallback)
[ ] Emergency check-in protocol established with contact

PENTEST
[ ] Authorization letter signed and accessible
[ ] VPS pivot provisioned (Monero-paid)
[ ] Tools in encrypted container
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using VPN without Tor | ISP sees VPN, VPN sees destination — single point of failure |
| Tor without bridge in censored country | Tor handshake detectable — always use obfs4 |
| Biometrics enabled at border | Power off completely before checkpoint |
| WiFi on passively | Device broadcasts probe requests with hardware identifiers |
| Storing sensitive data locally | Use OnionShare + immediate wipe |
| Single identity for all activities | Strict compartmentalization: journalist ≠ researcher ≠ social |
| Installing security tools in-country | Suspicious download patterns — bring everything pre-installed |
