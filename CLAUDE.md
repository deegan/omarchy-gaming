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

# Test from an Arch system or container.
# IMPORTANT: add this entry ABOVE [core]/[extra]/[multilib] in /etc/pacman.conf,
# not below. Several of our packages (gamemode, mangohud, ...) intentionally
# repackage a package that also exists in the official repos, and pacman
# resolves an unqualified name (including transitively, e.g. via
# omarchy-gaming-base's depends=()) using whichever repo is listed first. Below
# the defaults, our tuned rebuilds silently lose to the vanilla ones.
# [omarchy-gaming]
# Server = http://localhost:8080
# SigLevel = Optional TrustAll
pacman -Sy
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
`steamtinkerlaunch`, `goverlay`). Tier 2 is done: `gamemode`+`lib32-gamemode` and
`mangohud`+`lib32-mangohud` (both split packages, 64-bit built with the full daemon/app,
32-bit as a client-lib-only companion). Tier 3 is done: `omarchy-gaming-settings`,
`omarchy-gaming-base` (meta-package depending on the whole stack), and the GPU-vendor
metas `omarchy-gaming-nvidia` / `omarchy-gaming-amd`. Everything built is verified against
the local docker-compose repo server, including that `omarchy-gaming-base` resolves our
own rebuilt `gamemode`/`mangohud` rather than the vanilla `extra`/`multilib` ones (this
only works if `[omarchy-gaming]` is listed *above* the default repos in `pacman.conf` —
see the warning in "Local test workflow" above). `bin/build` now also mounts the local
`repo/` as a `file://` pacman source inside the build container (only if `bin/publish`
has already run), which is what lets meta-packages resolve same-repo dependencies via
`makepkg -s`.

`omarchy-kernel-gaming` (Tier 4) is done: `omarchy-kernel-gaming` +
`omarchy-kernel-gaming-headers`, built from CachyOS's own `cachyos-7.2.1-1` release
tarball with just the BORE scheduler patch applied, `x86-64-v3` baseline (portable,
unlike CachyOS's own machine-tuned default), and their other build-matrix knobs (LTO,
scheduler choice, bundled ZFS/nvidia-open/r8125, debug package) hardcoded off rather
than left runtime-configurable — same pattern as `gamemode.ini`/`MangoHud.conf` being
fixed opinions. Patch and base `.config` are vendored into `pkgbuilds/omarchy-kernel-gaming/`
pinned to specific CachyOS commits (not `master`), not fetched at build time. The build
takes ~25-30 min on 12 cores; `bin/build` handled it fine as a plain backgrounded run (no
changes needed there). Verified: installs cleanly, the `initramfs` package's mkinitcpio
hook correctly picks it up via the `pkgbase` file and builds a working
`/boot/vmlinuz-omarchy-kernel-gaming` + initramfs, `depmod` runs via the `kmod` hook, and
it **boots successfully** — confirmed on a real second machine (a spare PC running
Omarchy, added as a pacman repo over the LAN), installed alongside the stock kernel and
selected at the bootloader menu.

The repo has also now been validated end-to-end as a real remote pacman repo, not just
the local docker-compose loopback test: `docker-compose.yml`'s port mapping binds nginx
to `0.0.0.0:8080` by default, so it's reachable from other LAN machines without any
change; `restart: unless-stopped` plus Docker being enabled as a system service means the
repo survives reboots of the host machine unattended. Still unsigned (no GPG key exists
yet — every `bin/publish` run prints the "GPG_KEY_ID not set" warning) — currently
acceptable for LAN testing via `SigLevel = Optional TrustAll`, but CLAUDE.md's "All
packages are signed" line is aspirational, not yet true, if this repo is ever exposed
beyond a trusted LAN.

The Mesa rebuild (Tier 4) is done: `pkgbuilds/mesa/` (17 packages: `mesa`, `opencl-mesa`,
one `vulkan-*` per driver, the two `vulkan-mesa-*-layers` packages, `mesa-docs`) and
`pkgbuilds/lib32-mesa/` (the same 16, minus docs). Both are CachyOS's own PKGBUILDs
(themselves upstream Arch's PKGBUILD plus the SteamOS gamescope-fps-limiter patch) taken
near-verbatim — full driver matrix (`gallium-drivers=all`, every `vulkan-drivers=` entry,
rusticl, video-codecs=all), no LTO (upstream disabled it themselves over a real GCC14
miscompile bug) — with exactly one change: `CFLAGS`/`CXXFLAGS`/`RUSTFLAGS` overridden to
`-march=x86-64-v3` in `build()`, for both the 64-bit and the `--cross-file lib32` 32-bit
build. That the v3 flags actually took effect (not just "the build didn't error") was
checked directly: disassembling the built `vulkan-radeon` `.so` from both architectures
shows AVX2 (`ymm`) register usage, which the generic x86-64/SSE2 baseline would never
produce. Same package-name-replace pattern as `gamemode`/`mangohud` (no custom
provides/conflicts needed) — `mesa`/`lib32-mesa`/`vulkan-radeon`/`lib32-vulkan-radeon`
installed cleanly over the official ones in the docker-compose test. Build was ~35 min for
both combined, faster than the multi-hour estimate given beforehand. `mesa-debug`/
`lib32-mesa-debug` (1.1GB combined) are deliberately excluded from the published repo —
same call already made for `mangohud-debug` earlier.

**Not verified: actually running a graphical session on it.** Unlike the kernel, Mesa has
no side-by-side fallback — a bad build breaks every GPU-accelerated app in place. This
hasn't been tested on real hardware yet; the user's plan is to lean on Omarchy's
snapshot/rollback system as the safety net for that test, rather than this repo adding
its own (e.g. keeping the old package cached for a `pacman -U` downgrade).

## Next steps

All items in the original tier list (Tiers 1-4) are now built, published, and verified as
far as this local dev environment and one real spare machine allow. What's left is real
hardware validation on the actual gaming rig this was all built for:

1. Mesa: boot into a real graphical session with it installed and confirm nothing broke
   (games render, compositor works, no black-screen/crash-loop) — the one thing this
   session couldn't test.
2. Longer-term: the repo is still unsigned (see above) and the host's LAN IP is DHCP-
   assigned, not static — both fine for now, both worth revisiting if this setup needs to
   be more durable than "point pacman at whatever IP this dev machine currently has."
