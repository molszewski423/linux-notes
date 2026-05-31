# Debian 13  -  Rootless Podman Setup

Rootless Podman for running containers without root. Used on both MikePC and archbox.

## Install

```bash
sudo apt-get install -y podman uidmap slirp4netns fuse-overlayfs
```

## Enable Linger (keep containers running after logout)

```bash
sudo loginctl enable-linger $USER
```

Without this, all user containers stop when you log out.

## Quadlets (systemd-managed containers)

Quadlet files go in `~/.config/containers/systemd/`.
After adding/editing:

```bash
systemctl --user daemon-reload
systemctl --user start <service-name>
systemctl --user status <service-name>
journalctl --user -u <service-name> -f
```

## GPU Passthrough (NVIDIA)

Requires nvidia-container-toolkit with CDI spec generated:

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

In quadlet `.container` file:

```ini
[Container]
AddDevice=nvidia.com/gpu=0
```

## Subuid/Subgid

If containers fail with permission errors:

```bash
# Check
cat /etc/subuid
cat /etc/subgid

# Should have an entry like:
# mike:100000:65536

# If missing:
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER
podman system migrate
```

## Volume Restore from Backup

```bash
podman volume create <vol-name>
DEST=$(podman volume inspect <vol-name> --format '{{.Mountpoint}}')
rsync -avz --progress user@backup:/path/to/vol/ "$DEST/"
```
