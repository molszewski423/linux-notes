# NixOS → Debian 13 Migration (MikePC)

**Date:** May 2026

## Why NixOS — and Why We Left

NixOS was used extensively across multiple machines with genuine success: reproducible builds, atomic rollbacks, declarative config, excellent Wayland/NVIDIA support once tuned. It's a genuinely powerful system.

The breaking point was **CLI tooling friction**:

- Most AI/ML tools (`ollama`, `claude`, Python scripts, pip packages) assume a standard FHS filesystem. On NixOS, paths like `/usr/bin/python` don't exist — everything lives in the Nix store under hashed paths.
- Running arbitrary binaries (downloaded CLIs, NVIDIA run files, vendor tools) requires `patchelf`, `nix-ld`, or wrapping in `nix-shell` / `nix develop` environments. This adds overhead to every new tool.
- `pip install` into a venv works, but only inside a `nix-shell` with the right headers exposed. Without it you get missing `libstdc++`, broken `cffi` builds, etc.
- The fast-moving AI tooling ecosystem (new models, new inference engines, new SDKs weekly) clashed with NixOS's "declare everything" philosophy. Iterating quickly meant fighting the package manager constantly.

Debian 13 gives a stable, FHS-compliant base where any CLI tool just works. The tradeoff is giving up rollbacks and declarative config — acceptable for a dev workstation running containerized workloads anyway.

**TL;DR:** NixOS is excellent for stable, well-packaged software. It's friction for a machine whose primary job is running bleeding-edge AI CLI tools.

## Reason for Migration

## Backup Strategy

Everything backed up to archbox before wiping:

```bash
# From NixOS machine — rsync projects, configs, volumes to archbox
rsync -avz ~/agency/           mike@archbox:~/backups/mikenixpc/home/agency/
rsync -avz ~/pv-workbench/     mike@archbox:~/backups/mikenixpc/home/pv-workbench/
rsync -avz ~/ams-intelligence/ mike@archbox:~/backups/mikenixpc/home/ams-intelligence/
rsync -avz ~/.config/          mike@archbox:~/backups/mikenixpc/home/.config/
rsync -avz ~/.ssh/             mike@archbox:~/backups/mikenixpc/home/.ssh/
rsync -avz ~/.gnupg/           mike@archbox:~/backups/mikenixpc/home/.gnupg/
```

Podman volumes backed up separately (ChromaDB, Ollama models, n8n data).

## Restore on Debian

Restore scripts saved in `~/restore/` on MikePC:

| Script | Purpose |
|---|---|
| `01-nvidia-toolkit.sh` | NVIDIA container toolkit + CDI spec |
| `02-ollama.sh` | Ollama install + systemd overrides |
| `03-restore-files.sh` | rsync projects + configs from archbox |
| `04-restore-volumes.sh` | Restore Podman volumes from archbox backup |
| `05-pull-ollama-models.sh` | Pull qwen2.5:14b, gemma4:26b, llama3.1:8b |

## What Carried Over Cleanly

- All project code (rsync from archbox)
- SSH keys (copied from backup)
- GPG keys (copied from backup)
- Claude/Gemini configs (copied from backup)
- Podman volumes (rsync to volume mountpoints)

## What Needed Manual Work

- NVIDIA driver (run file install — apt driver too old for Blackwell)
- Ollama systemd overrides (bind to 0.0.0.0, set models path)
- Fish shell config (reinstall from dotfiles)
- Desktop setup (Hyprland on Debian required manual config)

## NixOS Config Preserved

Full NixOS configuration.nix archived at:
`archbox:~/backups/mikenixpc/home/nixos-config/`

Worth keeping — the NVIDIA Wayland env vars and CUDA setup are well-documented there.
