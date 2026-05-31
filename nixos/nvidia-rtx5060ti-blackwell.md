# NixOS  -  NVIDIA RTX 5060 Ti (Blackwell GB206)

**Machine:** MikePC (AMD Ryzen 7 7800X3D)
**Driver version:** 570.86.15+ required

## The Problem

RTX 50-series (Blackwell) GPUs **do not work** with the NVIDIA proprietary closed kernel module.
The closed module simply does not support the Blackwell architecture. Using it results in a black screen or failed modeset.

## The Fix

```nix
hardware.nvidia = {
  open = true;   # REQUIRED  -  do NOT set false on RTX 5060 Ti
  modesetting.enable = true;
  package = config.boot.kernelPackages.nvidiaPackages.stable;
};
```

`open = true` uses the open-source kernel module (not nouveau  -  this is NVIDIA's own open module released in 2022). Driver 570+ required.

## Wayland Environment Variables

All required for NVIDIA + Wayland (KDE Plasma / Hyprland):

```nix
environment.variables = {
  LIBVA_DRIVER_NAME         = "nvidia";
  GBM_BACKEND               = "nvidia-drm";
  __GLX_VENDOR_LIBRARY_NAME = "nvidia";
  NVD_BACKEND               = "direct";       # nvidia-vaapi-driver direct mode
  WLR_NO_HARDWARE_CURSORS   = "1";            # fixes invisible/glitchy cursor
  ELECTRON_OZONE_PLATFORM_HINT = "auto";      # Discord, VS Code on Wayland
  MOZ_ENABLE_WAYLAND        = "1";            # Firefox native Wayland
};
```

## CUDA Support

```nix
nixpkgs.config.cudaSupport = true;

hardware.nvidia-container-toolkit.enable = true;

# After install, generate CDI spec:
# sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

CUDA path for Python venvs (avoids LD_LIBRARY_PATH hacks in every venv):

```nix
environment.variables = {
  CUDA_PATH = "${pkgs.cudaPackages.cudatoolkit}";
  LD_LIBRARY_PATH = lib.mkForce (lib.concatStringsSep ":" [
    "${pkgs.cudaPackages.cudatoolkit}/lib"
    "${pkgs.cudaPackages.cudnn}/lib"
    "${pkgs.cudaPackages.libcublas}/lib"
  ]);
};
```

## Ollama on NixOS

Ollama with GPU support on NixOS requires `cudaSupport = true` and the open module. Once set, Ollama picks up the GPU automatically via the container toolkit CDI spec.

## Notes

- Migrated from NixOS to Debian 13 in May 2026  -  NixOS config preserved in archbox backup
- KDE Plasma 6 and Hyprland both worked with this setup (SDDM at login)
- 32-bit libraries needed for Steam/Proton: `hardware.graphics.enable32Bit = true`
