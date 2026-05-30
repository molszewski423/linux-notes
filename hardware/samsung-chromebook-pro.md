# Samsung Chromebook Pro → Linux (Caroline/Skylake)

**Board:** Caroline | **CPU:** Intel Core m3-6Y30 | **RAM:** 4GB | **Storage:** 32GB eMMC
**Final OS:** Debian 13 + Sway (Tokyo Night)
**Distros tried:** Arch Linux, Ubuntu, Fedora, Debian 13

## Distros Tried

| Distro | Result |
|---|---|
| Arch Linux | Worked but audio required manual patching every kernel update |
| Ubuntu | Worked, too heavy for 4GB RAM |
| Fedora | Worked, RPM Fusion needed for audio fix — Wayland compositor stability issues |
| **Debian 13** | Most stable — chosen as final OS |

## Hardware Write Protection Removal

The Chromebook has a physical write-protect screw that must be removed before flashing custom firmware.

1. Remove all 11 screws from the bottom panel
2. Lift the bottom cover carefully
3. Locate the WP screw near the battery connector (small silver screw on motherboard)
4. Remove it — this disables the hardware WP signal

## Disable Software Write Protection

Boot ChromeOS → `Ctrl+Alt+T` → `shell`:

```bash
sudo crossystem wpsw_boot=0
sudo crossystem wpsw_cur=0
```

Enable Developer Mode: `Esc+Refresh+Power` → Space → Enter.

## Flash UEFI Firmware (MrChromebox)

```bash
cd; curl -LO mrchromebox.tech/firmware-util.sh
sudo bash firmware-util.sh
# Select: Install/Update UEFI (Full ROM) firmware
```

Replaces Coreboot/depthcharge with full UEFI — enables standard Linux boot from USB.

## Audio Fix (CRITICAL — silent without this)

Skylake DSP requires AVS driver, not SOF. Add to `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet snd_intel_dspcfg.dsp_driver=4 i915.enable_psr=0"
```

```bash
sudo update-grub
```

Also required — set ALSA mixer level and persist it:

```bash
amixer -c 0 set Master 190
sudo alsactl store
```

**Why:** Upstream Linux ships SOF firmware for Skylake that doesn't match the ChromeOS firmware blob. The AVS driver uses Intel HDA instead, bypassing SOF entirely.

**Why `i915.enable_psr=0`:** Panel Self Refresh causes display flickering on this panel.

## Sleep / Suspend Fix

Skylake i915 DRM atomic commits fail on resume, crashing Sway with GPU hang.

`/etc/environment`:
```
WLR_DRM_NO_ATOMIC=1
WLR_DRM_NO_MODIFIERS=1
```

`/etc/systemd/logind.conf.d/lid.conf`:
```ini
[Login]
HandleLidSwitch=lock
HandleLidSwitchExternalPower=lock
```

Sleep/suspend disabled entirely. Lid close = screen off only (via Sway `bindswitch`).

**Root cause:** wlroots 0.18+ still uses atomic path for cursor planes even with `WLR_DRM_NO_ATOMIC`. No clean fix — disabling suspend is the only reliable workaround.

## RAM Management (4GB)

```bash
# 4GB swapfile
sudo fallocate -l 4G /home/swapfile
sudo chmod 600 /home/swapfile
sudo mkswap /home/swapfile
echo '/home/swapfile none swap sw,pri=10 0 0' | sudo tee -a /etc/fstab
sudo swapon /home/swapfile

# Swappiness
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swap.conf
sudo sysctl -p /etc/sysctl.d/99-swap.conf

# earlyoom — kills browser before compositor dies
sudo apt install earlyoom
# /etc/default/earlyoom:
# EARLYOOM_ARGS="-m 10 -s 15 -r 3600 --prefer '(brave|chromium)' --avoid '(sway|waybar|systemd)'"
sudo systemctl enable --now earlyoom
```

## Full config + install script

See: https://gitlab.com/molszewski423/chromebook-linux
