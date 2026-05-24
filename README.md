# claude-skill-remote-setup

A [Claude Code](https://claude.ai/code) skill that guides you through setting up full remote access for a new machine: SSH server, Tailscale VPN, Wake-on-LAN, and SSH key auth.

## What it does

When invoked, the skill asks a few questions about your setup (target OS, client machine, WoL device) and then walks you through the complete configuration step by step.

**Supported targets:** Windows, Linux (Debian/Ubuntu, Arch)  
**Supported clients:** any Unix-like shell (Linux, macOS, ChromeOS/Linux)  
**WoL sender:** Raspberry Pi, NAS, or any always-on Linux device

Typical topology:
```
[client laptop] ──SSH over Tailscale──► [target PC]
                                              ▲
[always-on Pi]  ──── magic packet ───────────┘
```

## Install

```bash
claude skill install https://github.com/giandemoncell-prog/claude-skill-remote-setup
```

Then in any Claude Code session:

> "set up remote access on my new Windows PC"  
> "configure SSH on the new Linux box"  
> "add this machine to my remote setup"

## What gets configured

- **OpenSSH Server** — installed, enabled, autostart
- **Firewall** — port 22 open
- **SSH key auth** — ED25519 key generated on client, installed on target (with correct ACLs on Windows)
- **Tailscale** — installed and connected on target, verified on both ends
- **Wake-on-LAN** — BIOS settings, NIC magic packet, Fast Startup disabled (Windows), systemd WoL service (Linux sender)
- **SSH config** — `~/.ssh/config` alias on client so `ssh machinename` just works
- **Reference file** — machine-specific guide saved locally with IPs, MAC, key paths

## Stack

| Component | Purpose |
|-----------|---------|
| [OpenSSH](https://www.openssh.com/) | Encrypted remote shell |
| [Tailscale](https://tailscale.com/) | Zero-config mesh VPN — works across NAT, no port forwarding needed |
| [wakeonlan](https://github.com/jpoliv/wakeonlan) | Send magic packets from the always-on device |
| [ethtool](https://man7.org/linux/man-pages/man8/ethtool.8.html) | Enable WoL on Linux NICs at boot |

## Part of

[giandemoncell-prog/claude-skills](https://github.com/giandemoncell-prog/claude-skills) — collection of all Claude Code skills.

## License

MIT
