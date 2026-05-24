---
name: remote-setup
description: Configure full remote access for a new machine: SSH server, Tailscale VPN, Wake-on-LAN, SSH keys, and helper scripts. Use whenever the user wants to set up remote access, add a new machine to their network, configure SSH, install Tailscale, or enable WoL on another device. Also triggers for "add machine to remote setup", "configure new PC for remote work", "setup SSH on new computer".
tools: Read, Write, Edit, Bash, Glob, Grep
---

# Remote Setup Skill

Configure secure remote access for a new machine using the proven stack: **Tailscale + OpenSSH + Wake-on-LAN**.

The user's reference setup (working as of 2026-05-24):
- Chromebook (client) → SSH → Windows PC (target) via Tailscale
- Raspberry Pi always-on → Wake-on-LAN → wakes Windows PC
- Tailscale account: gluca.3d@gmail.com
- Reference guide: `D:\LAVORO_REMOTO\chromebook_config.md`

---

## Phase 1: Gather Context

Ask the user:
1. **Target machine OS** — Windows, Linux (which distro), macOS?
2. **Client machine** — where they'll connect from (Chromebook, Linux, Windows, macOS)?
3. **WoL needed?** — Is there an always-on device (Pi, NAS, old PC) that can send magic packets?
4. **Tailscale account** — same account as existing devices, or new one?

If the target is Windows, jump to the Windows section. If Linux/macOS, jump to the appropriate section.

---

## Windows Target

### Step 1: OpenSSH Server

```powershell
# Install
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# Enable + autostart
Set-Service -Name sshd -StartupType Automatic
Start-Service sshd

# Firewall rule (usually auto-created, verify)
Get-NetFirewallRule -Name *ssh*
```

Verify listening: `netstat -an | findstr :22`

### Step 2: Default shell → PowerShell

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell `
  -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" `
  -PropertyType String -Force
```

### Step 3: SSH Key Auth

On the **client**, generate key if needed:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_<machinename>
```

On Windows, install the public key:
```powershell
# For admin users: use administrators_authorized_keys (NOT ~/.ssh/authorized_keys)
$pubkey = Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub"  # or paste from client
$authKeysPath = "C:\ProgramData\ssh\administrators_authorized_keys"
Add-Content $authKeysPath $pubkey

# Fix permissions (required — SSH ignores keys with wrong ACLs)
icacls $authKeysPath /inheritance:r /grant "SYSTEM:(R)" /grant "Administrators:(R)"
```

Test: `ssh -i ~/.ssh/id_<machinename> Username@<ip>`

### Step 4: Disable password auth (optional but recommended)

Edit `C:\ProgramData\ssh\sshd_config`:
```
PasswordAuthentication no
PubkeyAuthentication yes
```
Then `Restart-Service sshd`.

### Step 5: WoL BIOS/Windows config

**BIOS (UEFI):**
- ErP Ready → **Disabled**
- Power On By PCI-E → **Enabled**
- Save & exit

**Windows (Device Manager):**
- Find the NIC (e.g., Intel I219-V)
- Properties → Power Management: enable "Allow this device to wake the computer"
- Properties → Advanced: enable "Wake on Magic Packet"

**Windows Fast Startup** — must be disabled for WoL to work:
```powershell
# Disable via registry
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power" `
  -Name HiberbootEnabled -Value 0
```

Get MAC address for WoL:
```powershell
Get-NetAdapter | Select-Object Name, MacAddress, Status
```

---

## Linux Target

### OpenSSH Server

```bash
# Debian/Ubuntu
sudo apt install openssh-server
sudo systemctl enable --now ssh

# Arch
sudo pacman -S openssh
sudo systemctl enable --now sshd
```

### SSH Key Auth

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "ssh-ed25519 AAAA... user@client" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### WoL on Linux (as wake-sender, e.g. Raspberry Pi)

```bash
sudo apt install wakeonlan ethtool

# Test WoL capability of NIC
sudo ethtool eth0 | grep Wake

# Enable WoL at boot (systemd service)
sudo tee /etc/systemd/system/wol.service <<EOF
[Unit]
Description=Enable Wake-on-LAN
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ethtool -s eth0 wol g
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable --now wol.service
```

Wake script:
```bash
# ~/wake-<machinename>.sh
#!/bin/bash
wakeonlan AA:BB:CC:DD:EE:FF
```

---

## Tailscale Setup (all platforms)

1. Install from tailscale.com (one command on Linux: `curl -fsSL https://tailscale.com/install.sh | sh`)
2. `sudo tailscale up` → opens browser for auth → log in with gluca.3d@gmail.com
3. Verify: `tailscale ip -4` → note the 100.x.x.x address

Check all devices are visible: `tailscale status`

---

## SSH Config on Client

Add to `~/.ssh/config` on the client machine:

```
Host <alias>
    HostName 100.x.x.x        # Tailscale IP of target
    User <username>
    IdentityFile ~/.ssh/id_<machinename>
    ServerAliveInterval 60
```

Test: `ssh <alias>`

---

## Checklist Before Declaring Done

- [ ] SSH service running and autostarts on target
- [ ] Firewall allows port 22
- [ ] Key auth works (no password prompt)
- [ ] Tailscale connected on both client and target, same account
- [ ] `ssh <alias>` works from client
- [ ] WoL: BIOS configured, magic packet enabled on NIC, Fast Startup disabled
- [ ] MAC address noted for wake scripts
- [ ] `~/wake-<machinename>.sh` script tested end-to-end
- [ ] `~/.ssh/config` entry added on client
- [ ] Guide saved to `D:\LAVORO_REMOTO\<machinename>_remote_setup.md`

---

## Save a Guide

At the end, save a machine-specific reference file to `D:\LAVORO_REMOTO\<machinename>_remote_setup.md` with:
- Tailscale IP
- MAC address
- SSH alias and key path
- Wake script location
- Any quirks encountered during setup
