# NixOS — Strengths, Gotchas, and When to Use It

Used across multiple machines. Worth returning to for the right workload.

## What NixOS Does Well

- **Atomic rollbacks** — bad update? `nixos-rebuild switch --rollback` and you're back
- **Declarative everything** — one `configuration.nix` reproduces the entire system
- **No dependency hell** — multiple versions of the same library coexist cleanly
- **NVIDIA + Wayland** — once configured correctly, rock solid (see `nvidia-rtx5060ti-blackwell.md`)
- **Development shells** — `nix develop` gives a reproducible build environment per project
- **Home Manager** — declarative dotfiles management, rollbackable like the system

## The CLI Tooling Problem

NixOS breaks the assumption that `/usr/bin/`, `/usr/lib/`, and `/usr/local/` exist.
Everything lives in `/nix/store/<hash>-<pkg>/`. This causes friction with:

| Tool | Problem | Workaround |
|---|---|---|
| Downloaded binaries (claude, agy, vendor CLIs) | Wrong ELF interpreter path | `patchelf --set-interpreter` or `nix-ld` |
| `pip install` | Missing headers, libstdc++ | Run inside `nix-shell -p python3 gcc` |
| NVIDIA `.run` files | Assumes standard paths | Use Nix NVIDIA packages instead |
| Node/npm tools | Native addons fail to compile | `nix-shell -p nodejs gcc` |
| Fast-moving AI tools | Not in nixpkgs yet | Pin or overlay, adds overhead |

## nix-ld (partial fix)

`nix-ld` makes `/lib/ld-linux.so` work for most downloaded binaries:

```nix
programs.nix-ld.enable = true;
```

Fixes most pre-compiled binaries but not everything.

## When NixOS Makes Sense

- Servers with stable, well-packaged software stacks
- CI/CD build machines (reproducibility is valuable)
- Machines where rollback capability matters (production-adjacent)
- When you want the entire OS in version control

## When to Use Something Else

- Dev workstations running bleeding-edge AI/ML tooling
- Machines where you frequently install arbitrary CLIs or vendor tools
- When iteration speed matters more than reproducibility
- Anywhere `pip install X` needs to just work

## Recommended Approach

Use NixOS for servers. Use Debian on dev machines.
Keep NixOS config in git — if you return to it, the config is ready.
