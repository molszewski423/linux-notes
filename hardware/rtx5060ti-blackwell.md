# Hardware — NVIDIA RTX 5060 Ti (Blackwell GB206)

## Key Facts

- Architecture: Blackwell (GB206)
- **Requires open kernel module** — closed module does not support Blackwell
- Minimum driver: 570.86.15
- CUDA: 12.8+

## Quick Reference

| Situation | Fix |
|---|---|
| Black screen on boot | Must use `open = true` / `nvidia-open` module |
| Invisible cursor on Wayland | `WLR_NO_HARDWARE_CURSORS=1` |
| Electron apps (Discord, VS Code) broken on Wayland | `ELECTRON_OZONE_PLATFORM_HINT=auto` |
| CUDA not found in Python venv | Set `CUDA_PATH` and `LD_LIBRARY_PATH` |
| Podman GPU passthrough | `nvidia-ctk cdi generate`, then `AddDevice=nvidia.com/gpu=0` in quadlet |
| Ollama not using GPU | Check `nvidia-smi` first, then verify CDI spec exists |

## Tested on

- **NixOS** (Flakes, driver 570) — fully working, Hyprland + KDE Plasma
- **Debian 13** (run file install 570.172.08) — fully working, GNOME + Hyprland

## Ollama Performance

- gemma4:26b — works, slow on 8GB VRAM
- qwen2.5:14b — good performance
- llama3.1:8b — fast

## Notes

- RTX 5060 Ti has 8GB GDDR7 VRAM
- `nvidia-smi` should show driver version and GPU utilization
- After kernel update: DKMS rebuilds module automatically if installed via package manager
- Run file installs do NOT auto-rebuild on kernel update — use DKMS packages when available
