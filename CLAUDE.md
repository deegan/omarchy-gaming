# omarchy-gaming

An Arch Linux pacman repository delivering a curated gaming package stack for Omarchy,
following the same sovereign build philosophy as omarchy-pkgs: no AUR at runtime,
everything built and hosted from our own infrastructure.

## Repository structure

omarchy-gaming/
pkgbuilds/ ← one directory per package, each containing a PKGBUILD
repo/ ← built .pkg.tar.zst files + pacman database (gitignored)
bin/
build ← build one or all packages
publish ← run repo-add and refresh the database
docker-compose.yml ← nginx serving ./repo on localhost:8080 for local testing
CLAUDE.md


## Package tiers

### Tier 1 — AUR-only, must build ourselves
- `proton-ge-custom` — binary repack of upstream GE release, no compilation
- `wine-ge-custom` — same pattern
- `protonup-qt` — binary repack
- `heroic-games-launcher-bin` — binary repack
- `steamtinkerlaunch` — bash scripts, no compilation
- `goverlay` — compiles from source

### Tier 2 — Arch official, repackaged with tuning
- `gamemode` + `lib32-gamemode` — compile from source, ship opinionated /etc/gamemode.ini
- `mangohud` + `lib32-mangohud` — compile from source, ship default MangoHud.conf

### Tier 3 — Meta and config packages (omarchy-style opinions)
- `omarchy-gaming-base` — meta-package pulling in the full stack
- `omarchy-gaming-settings` — sysctl + udev rules (vm.max_map_count, controller udev, ananicy rules)
- `omarchy-gaming-nvidia` — NVIDIA-specific meta
- `omarchy-gaming-amd` — AMD/RADV-specific meta

### Tier 4 — Future / heavy
- `omarchy-kernel-gaming` — linux-cachyos-style kernel with BORE scheduler, tuned for desktop/gaming
- Mesa rebuild with x86-64-v3 targeting (requires full rebuild pipeline)

## PKGBUILD conventions

- Binary repacks: fetch upstream release asset, verify sha256, repackage. Never download
  more than needed — read the upstream manifest/index to get the URL and checksum,
  don't fetch the artifact just to hash it.
- Compiled packages: use a clean `archlinux` Docker container for builds to ensure
  reproducibility. Do not build on the host directly.
- `pkgrel` resets to 1 on every `pkgver` bump.
- All packages are signed (GPG). Key lives outside the repo.

## Local test workflow

```bash
# Start the local repo server
docker compose up -d

# Build a single package
bin/build proton-ge-custom

# Refresh the pacman database
bin/publish

# Test from an Arch system or container
# /etc/pacman.conf entry:
# [omarchy-gaming]
# Server = http://localhost:8080
# SigLevel = Optional
pacman -Sy omarchy-gaming
pacman -S proton-ge-custom
```

## What NOT to do

- Do not shell out to `yay` or any AUR helper. We are the AUR helper.
- Do not add packages that already exist in official Arch repos unchanged — only add
  them if we're patching, recompiling with different flags, or bundling config.
- Do not fetch full release artifacts to compute checksums. Read upstream manifests.
- Do not commit anything under `repo/` — built packages are artifacts, not source.

## Progress

Scaffolding is done: docker-compose + nginx, `bin/build`/`bin/publish`, and all of
Tier 1 (`proton-ge-custom`, `wine-ge-custom`, `protonup-qt`, `heroic-games-launcher-bin`,
`steamtinkerlaunch`, `goverlay`). From Tier 2/3, `gamemode` and `omarchy-gaming-settings`
are also done. Everything built is verified against the local docker-compose repo server.

## Next steps

1. `lib32-mangohud` and `mangohud` — compile from source, ship a default `MangoHud.conf`
   (Tier 2)
2. `lib32-gamemode` — 32-bit companion to the already-built `gamemode` (Tier 2)
3. `omarchy-gaming-base` — meta-package pulling in the full stack (Tier 3)
4. `omarchy-gaming-nvidia` / `omarchy-gaming-amd` — GPU-vendor meta packages (Tier 3)
5. `omarchy-kernel-gaming` — custom kernel, BORE scheduler, tuned for desktop/gaming (Tier 4)
6. Mesa rebuild with x86-64-v3 targeting (Tier 4, requires a full rebuild pipeline)
