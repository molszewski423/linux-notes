# Arch Linux — Homelab Server Setup (archbox)

**Machine:** archbox (Intel i3-4130T, Alienware Alpha)
**Role:** 24/7 home server — 25 Podman containers, AdGuard, CrowdSec, Tailscale exit node

## Initial Setup

```bash
# AUR helper
git clone https://aur.archlinux.org/yay.git
cd yay && makepkg -si

# Core services
sudo pacman -S tailscale adguardhome crowdsec crowdsec-firewall-bouncer-nftables nftables
```

## Tailscale as Exit Node

Required kernel parameters for IP forwarding (Tailscale exit node won't work without these):

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

sudo systemctl enable --now tailscaled
sudo tailscale up --advertise-exit-node
```

### Tailscale + nftables Conflict

Tailscale needs `nf_tables` and `xt_MASQUERADE` kernel modules loaded:

```bash
sudo modprobe nf_tables
sudo modprobe xt_MASQUERADE
```

If Tailscale fails to start with nftables mode:

```bash
# Check logs
sudo journalctl -u tailscaled -n 20

# Force nftables mode
echo 'FLAGS="--tun=userspace-networking"' | sudo tee /etc/default/tailscaled
sudo mkdir -p /etc/systemd/system/tailscaled.service.d
# Add override if needed, then:
sudo systemctl daemon-reload && sudo systemctl restart tailscaled
```

### Tailscale Debug

```bash
TS_DEBUG_FIREWALL_MODE=nftables tailscale up --advertise-exit-node
```

## AdGuard Home

```bash
sudo pacman -S adguardhome
sudo systemctl disable --now systemd-resolved  # conflicts with AdGuard on port 53
sudo systemctl mask systemd-resolved
sudo systemctl enable --now adguardhome
# Admin UI: http://archbox:3000
```

## CrowdSec + nftables Bouncer

```bash
# Install collections
sudo cscli collections install crowdsecurity/linux
sudo cscli collections install crowdsecurity/sshd
sudo cscli collections install crowdsecurity/whitelist-good-actors

sudo systemctl enable --now crowdsec
sudo systemctl enable --now crowdsec-firewall-bouncer
```

The bouncer config is at `/etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml`.
It creates `crowdsec` and `crowdsec6` nftables tables that auto-ban malicious IPs.

## Rootless Podman

```bash
# Enable lingering (keeps containers running after logout)
sudo loginctl enable-linger mike

# Quadlets in ~/.config/containers/systemd/
# After editing:
systemctl --user daemon-reload
systemctl --user start agency-pod
```

## SSH Hardening

```bash
# Test config before restart
sudo sshd -t && sudo systemctl restart sshd
```

## Notes

- Previously ran AdGuard + CrowdSec on a Raspberry Pi 4 — migrated to archbox for x86 compatibility
- btrfs snapshots active (/.snapshots/) — regular snapshots taken automatically
- GRUB config: `sudo nvim /etc/default/grub && sudo grub-mkconfig -o /boot/grub/grub.cfg`
