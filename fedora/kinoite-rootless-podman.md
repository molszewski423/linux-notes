# Fedora Kinoite  -  Rootless Podman on an Immutable Base

**Distro:** Fedora Kinoite (KDE Plasma, Fedora Atomic base)
**Use case:** Running rootless Podman containers on an immutable desktop OS

## What is Fedora Kinoite

Fedora Kinoite is an immutable variant of Fedora with KDE Plasma. The base OS is read-only and updated atomically via `rpm-ostree`. Think of it as NixOS-style atomic updates but with a standard RPM/FHS base underneath.

- Base OS is read-only (`/usr` is immutable)
- Updates applied as a new image, activated on next boot
- Rollback to previous image with `rpm-ostree rollback`
- GUI apps via Flatpak
- Dev environments via Toolbox / Distrobox
- Podman ships pre-installed and first-class

## Why It Works Well for Podman

Unlike NixOS, the underlying filesystem is standard FHS  -  containers see a normal Linux environment. Rootless Podman on Kinoite is the intended workflow, not a workaround.

- `/etc/subuid` and `/etc/subgid` configured automatically at install
- Podman pre-installed, no extra setup
- systemd user units work as expected
- Quadlets (`~/.config/containers/systemd/`) work identically to Debian/Arch

## Podman Setup

```bash
# Enable linger (containers survive logout)
sudo loginctl enable-linger $USER

# Verify subuid/subgid
cat /etc/subuid   # should show: mike:100000:65536
cat /etc/subgid

# Start user container via quadlet
systemctl --user daemon-reload
systemctl --user start <service>
journalctl --user -u <service> -f
```

## SELinux (main difference from Debian/Arch)

Fedora runs SELinux enforcing by default. This causes container volume mount failures that don't happen on Debian (AppArmor) or Arch (no MAC by default).

**Symptom:** Container starts but can't read/write mounted volumes. Logs show `Permission denied` even though file permissions look correct.

**Fix  -  label the volume mount:**

```bash
# Add :Z to volume mounts in quadlet (relabels for container use)
Volume=/home/mike/data:/data:Z
```

`:Z` = private label (one container), `:z` = shared label (multiple containers).

**Alternative  -  disable SELinux for container (not recommended for production):**

```bash
podman run --security-opt label=disable ...
```

## Installing Packages on an Immutable Base

The base OS is read-only  -  `dnf install` doesn't work on the host directly.

```bash
# Layer a package permanently onto the base image (survives updates)
rpm-ostree install <package>
# Reboot to activate

# Temporary dev environment (doesn't modify base)
toolbox create
toolbox enter
# Now you have a full mutable Fedora container as your shell
```

## Distrobox (better than Toolbox for dev work)

Distrobox integrates a container as a seamless shell  -  home directory is shared, GUI apps work, systemd services accessible.

```bash
# Install via rpm-ostree or Flatpak
rpm-ostree install distrobox

# Create a Debian container for dev work
distrobox create --name debian --image debian:13
distrobox enter debian
# Full Debian shell with access to ~/
```

Used this to run Debian-based tooling on a Kinoite host  -  best of both worlds.

## Updates and Rollback

```bash
# Check for updates
rpm-ostree upgrade --check

# Apply update (takes effect on next boot)
rpm-ostree upgrade

# Rollback if something breaks
rpm-ostree rollback
```

## Kinoite vs NixOS vs Debian

| | Kinoite | NixOS | Debian |
|---|---|---|---|
| Base | Immutable, FHS | Immutable, non-FHS | Mutable, FHS |
| Rollback | Yes (rpm-ostree) | Yes (generations) | No |
| Arbitrary binaries | Yes (FHS under the hood) | Friction (non-FHS) | Yes |
| Podman | First-class, pre-installed | Works, needs config | Works, manual install |
| SELinux | Enforcing (watch `:Z` on volumes) | None | AppArmor (permissive) |
| Package install | Layer via rpm-ostree or Toolbox | Declare in config | apt |

## Why I Moved On

Kinoite is excellent for a desktop that doubles as a container host. Moved to Debian as a stable base for simplicity  -  one less abstraction layer, and the immutability benefits matter less on a machine managed with dotfiles and backup scripts anyway.
