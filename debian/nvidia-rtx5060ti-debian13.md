# Debian 13  -  NVIDIA RTX 5060 Ti Driver Install

**Machine:** MikePC (Ryzen 7 7800X3D, RTX 5060 Ti)
**Driver:** 570.172.08 (run file method  -  apt driver too old for Blackwell at time of install)

## Why the Run File Method

Debian 13's packaged NVIDIA driver was too old for Blackwell (RTX 50-series) at time of install.
Had to download the official `.run` file directly from NVIDIA.

## Install

```bash
# Install build dependencies
sudo apt install wget build-essential linux-headers-$(uname -r) pkg-config libglvnd-dev

# Download driver
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/570.172.08/NVIDIA-Linux-x86_64-570.172.08.run

# Install (stop display manager first if running)
sudo bash NVIDIA-Linux-x86_64-570.172.08.run
```

## NVIDIA Container Toolkit (for Podman/Docker GPU passthrough)

Required for Ollama in containers and AI workloads.

```bash
# Add repo
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit

# Generate CDI spec (required for Podman)
sudo mkdir -p /etc/cdi
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
sudo nvidia-ctk runtime configure --runtime=podman

# Test
podman run --rm --device nvidia.com/gpu=0 \
  docker.io/nvidia/cuda:12.8.0-base-ubuntu24.04 nvidia-smi
```

## Ollama Setup

```bash
# Install
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl enable --now ollama

# Bind to all interfaces (so archbox containers can reach it over Tailscale)
sudo mkdir -p /etc/systemd/system/ollama.service.d
printf '[Service]\nEnvironment="OLLAMA_HOST=0.0.0.0:11434"\n' \
  | sudo tee /etc/systemd/system/ollama.service.d/host.conf

# Set models path
printf '[Service]\nEnvironment="OLLAMA_MODELS=/home/mike/.ollama/models"\n' \
  | sudo tee /etc/systemd/system/ollama.service.d/models.conf

sudo systemctl daemon-reload && sudo systemctl restart ollama
```

## Ollama Permissions Issue

After restoring `.ollama` from backup, Ollama couldn't read models:

```bash
# Fix ownership
sudo chown -R ollama:ollama /home/mike/.ollama
# OR if that fails:
chmod -R 755 /home/mike/.ollama
sudo systemctl restart ollama
ollama list
```

## Notes

- Ollama is accessed by archbox containers via Tailscale IP `100.97.45.57:11434`
- CDI spec must be regenerated after driver updates: `sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml`
- Migrated from NixOS in May 2026 using restore scripts from archbox backup
