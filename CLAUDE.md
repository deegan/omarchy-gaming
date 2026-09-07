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
check-updates ← report/auto-bump packages with a newer upstream version (--fix)
clean ← interactively delete superseded package versions from repo/
nvchecker.toml ← per-package upstream version-check config, used by check-updates
docker-compose.yml ← nginx serving ./repo on localhost:8080 for local testing
CLAUDE.md ← this file (assistant/contributor instructions)
README.md ← user-facing guide: using the repo, hosting your own instance, signing, troubleshooting


## Package tiers

### Tier 1 — AUR-only, must build ourselves
- `proton-ge-custom` — binary repack of upstream GE release, no compilation
- `proton-cachyos` — binary repack of CachyOS's performance-tuned Proton
  (Steam Linux Runtime build, x86-64-v3 asset), same pattern
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

## Updating packages

Nothing here fetches upstream automatically — every `pkgver` is a hardcoded literal in its
PKGBUILD, and `bin/build` only ever builds whatever's already on disk. It never checks
whether that's the latest upstream version, so there's no "wasted build" risk from staying
still — the risk runs the other way: silently drifting behind upstream with nothing telling
you.

`bin/check-updates` closes that gap as a read-only report, not a fourth build step:

```bash
bin/check-updates
```

It uses `nvchecker` (`bin/nvchecker.toml`, one `[section]` per trackable package — GitHub
releases/tags for the Tier 1 binary repacks and gamemode/mangohud, Arch's own `mesa`
package version for the Mesa rebuild) to fetch each package's latest upstream version and
prints a table comparing it against the current `pkgver` in each PKGBUILD. It never edits a
PKGBUILD, never downloads a package artifact, and never builds anything — it exits non-zero
if anything is outdated or a check failed, so it's safe to script around. Uses the host's
`nvchecker` if present, otherwise runs it in a throwaway `archlinux` container the same way
`bin/build`/`bin/publish` fall back to Docker (nvchecker is an official `extra` package, not
AUR). Also needs `jq` on the host to parse its output.

Meta/config-only packages (`omarchy-gaming-base`, `omarchy-gaming-settings`,
`omarchy-gaming-nvidia`, `omarchy-gaming-amd`) have no upstream and aren't in
`nvchecker.toml`. `omarchy-kernel-gaming`'s entry is a version-only signal — a newer CachyOS
tag doesn't mean "just bump pkgver," since the BORE patch and base `.config` are vendored
and pinned to specific CachyOS commits and need re-vetting, not just a checksum bump.

### Auto-bumping with `--fix`

```bash
bin/check-updates --fix
```

edits the PKGBUILD directly (`pkgver`/`_srctag`, checksum, `pkgrel` reset to 1) for whichever
outdated packages that's safe to automate — currently `proton-ge-custom`, `wine-ge-custom`,
`proton-cachyos`, `protonup-qt`, `heroic-games-launcher-bin`, `mangohud`. "Safe" means
upstream publishes a checksum for the exact release asset we use — a `.sha512sum` manifest
file (the GE-Proton/CachyOS pattern) or, for a plain GitHub release asset with no such
manifest, the sha256 `digest` GitHub's own API reports for every uploaded release asset — so
bumping never means downloading the artifact just to hash it, same rule as a manual bump.

The rest print an explicit reason instead of being touched:
- `goverlay`, `gamemode`, `steamtinkerlaunch`: source is a GitHub-generated tag archive
  (`archive/refs/tags/...`), not an uploaded release asset — no manifest, no API digest,
  nothing to fetch without downloading the tarball to hash it ourselves.
- `mesa`/`lib32-mesa`: the PKGBUILD is close to a verbatim copy of CachyOS's own, which can
  change shape (driver list, patches) between versions — a version-only bump risks silently
  going stale on the rest of the recipe.
- `omarchy-kernel-gaming`: as above, needs the BORE patch re-vetted, not just a version swap.

Whether it bumped, failed, or left something for you, `check-updates --fix` finishes by
printing the right `bin/build <pkg> <pkg> ...` line for whatever it actually changed — run
that (then `bin/publish`) rather than a bare `bin/build`, which rebuilds everything
regardless of what changed. Without `--fix`, `check-updates` stays purely a read-only report
and prints the same kind of `bin/build` line for you to run after bumping by hand.

## Reclaiming disk space

`bin/build` never deletes anything — every version a package has ever had stays in `repo/`.
That's deliberate: it's a rollback option for a single bad package (a Mesa or kernel build
that boots but breaks graphics, say) that Omarchy's own whole-system snapshots don't target
that precisely. The tradeoff is `repo/` only grows (Proton/Mesa builds are hundreds of MB to
1GB+ each), so use `bin/clean` to reclaim space by hand:

```bash
bin/clean
```

It's an `fzf`-driven interactive picker (needs `fzf` on the host — no Docker fallback, this
one's meant to run at a real terminal): it cross-checks `repo/*.pkg.tar.zst` against the
published database's own `%FILENAME%` fields (`repo/*.db.tar.gz`) — not file mtime, not a
guess at version ordering — so whatever the database doesn't currently point at for a
package is offered for deletion, and whatever it does is never offered. TAB to select,
CTRL-A/CTRL-D to select/deselect all, ENTER to confirm a final size summary + y/N prompt
before anything is actually removed.

## What NOT to do

- Do not shell out to `yay` or any AUR helper. We are the AUR helper.
- Do not add packages that already exist in official Arch repos unchanged — only add
  them if we're patching, recompiling with different flags, or bundling config.
- Do not fetch full release artifacts to compute checksums. Read upstream manifests.
- Do not commit anything under `repo/` — built packages are artifacts, not source.

## Progress

Scaffolding is done: docker-compose + nginx, `bin/build`/`bin/publish`, and all of
Tier 1 (`proton-ge-custom`, `wine-ge-custom`, `protonup-qt`, `heroic-games-launcher-bin`,
`steamtinkerlaunch`, `goverlay`). `proton-cachyos` was added to Tier 1 after the fact —
CachyOS's own performance-focused Proton build (distinct upstream from GE: latency/perf
patches like NTSYNC and tuned compiler flags rather than GE's broad game-compat patch
set), packaged from their prebuilt Steam Linux Runtime release asset already built for
`x86_64_v3` (matching this repo's v3-baseline stance elsewhere — Mesa, the custom
kernel), rather than generating a `compatibilitytool.vdf` ourselves like the AUR
`proton-cachyos-native` package does, since the SLR release tarball already ships a
working one. Built and published successfully; contents verified directly from the
built `.pkg.tar.zst` (compat-tool dir, `.vdf` files, `.INSTALL`, and the
`modules-load.d` ntsync conf all land where expected). Now fully verified on real
hardware too: installed via `pacman -S proton-cachyos` on the user's main gaming
machine, selected as a game's compatibility tool in Steam, and launched successfully.
Tier 2 is done: `gamemode`+`lib32-gamemode` and
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

Package updates (see "Updating packages" above) are now handled by `bin/check-updates` +
`bin/nvchecker.toml`, one `nvchecker` source entry per trackable package (GitHub
releases/tags for Tier 1 + gamemode/mangohud, Arch's own `mesa`/`lib32-mesa` package
version for the Mesa rebuild). Verified end-to-end against this repo's real PKGBUILDs, not
just a dry run — it correctly flagged `proton-ge-custom` genuinely one release behind (11.5
vs. upstream's 11.6, since bumped and built) and `omarchy-kernel-gaming` behind CachyOS's
newest tag (though that one's an `-rc` prerelease, which is exactly why that entry's comment
warns it's a version-only signal, not a build-it-now signal). Getting nvchecker's config
right took two real bugs to shake out, worth knowing if this ever needs touching again:
nvchecker only writes the `newver` file if `oldver` is *also* set in `[__config__]` (both
are in the config, deleted before every run, since `bin/check-updates` does its own
comparison against each PKGBUILD rather than trusting nvchecker's own old/new diffing); and
the JSON it writes is nested as `{"data": {name: {"version": ...}}}`, not a flat
`{name: version}` map.

`bin/build` no longer deletes superseded package versions (see "Reclaiming disk space"
above) — `bin/clean` handles that now, interactively, via `fzf`. And `bin/check-updates
--fix` (see "Auto-bumping with `--fix`" above) auto-edits the PKGBUILD for 6 of the 9
version-tracked packages; verified correct by deliberately corrupting each of those 6
PKGBUILDs' `pkgver`/checksum to garbage values, running `--fix`, and diffing the result
against git — zero diff every time, i.e. it reconstructed the exact already-known-correct
values from upstream's own published checksums/digests, not just "didn't crash." Building
what it bumps is still a separate, explicit `bin/build <pkg>...` step (the line
`check-updates --fix` prints) — it does not chain into a build itself.

## Next steps

All items in the original tier list (Tiers 1-4) are now built, published, and verified as
far as this local dev environment and one real spare machine allow. `proton-cachyos` has
now additionally been validated on the actual gaming rig this was all built for (install +
launched a game via Steam). What's left:

1. Mesa: boot into a real graphical session with it installed and confirm nothing broke
   (games render, compositor works, no black-screen/crash-loop) — the one thing this
   session couldn't test.
2. Longer-term: the repo is still unsigned (see above) and the host's LAN IP is DHCP-
   assigned, not static — both fine for now, both worth revisiting if this setup needs to
   be more durable than "point pacman at whatever IP this dev machine currently has."
3. `omarchy-kernel-gaming` is currently behind CachyOS's newest tag, but that tag is an
   `-rc` prerelease — worth another look once CachyOS cuts a real release, not before.
4. Mesa/lib32-mesa (still pkgver 26.2.1) currently fail to *rebuild*: `bin/build mesa
   lib32-mesa` breaks in rusticl's Rust bindings with `error[E0605]: non-primitive cast:
   pipe_resource_usage as u32` (and the same for `pipe_map_flags`). Root-caused: this is a
   genuine upstream Mesa bug, not anything in our PKGBUILD (which is still an unmodified,
   verbatim copy of CachyOS's own on this point) — `struct pipe_resource`/`struct
   pipe_transfer` in `p_state.h` declare `usage` as a real C bitfield (`enum
   pipe_resource_usage usage:4`, `enum pipe_map_flags usage:24`), but rusticl's
   `meson.build` also marks both types `--bitfield-enum` for bindgen (needed because
   `pipe_map_flags` genuinely gets bitwise-OR'd in `core/device.rs`/`mesa/pipe/context.rs`),
   which makes bindgen emit them as newtype structs — and bindgen's own bitfield-accessor
   codegen for the C-bitfield fields still emits a bare `as` cast that only works on
   primitives/real enums, not the newtype it just generated. Reproduced identically against
   both current Arch `extra` bindgen (0.73.1) and Mesa's own stated minimum (0.71.1), so
   it's not a "pin an older bindgen" fix. A real fix means patching core Gallium struct
   layout (`pipe_resource`/`pipe_transfer`, used by every driver) or dropping rusticl
   (`-D gallium-rusticl=false`) — both rejected for now in favor of just waiting for
   upstream to fix it, since we don't want to drift from CachyOS's PKGBUILD.
   **This does not affect what's already published**: `repo/` still has the working,
   previously-verified `26.2.1-2` build (the one confirmed via AVX2 disassembly and a real
   install) — only a *rebuild* is currently blocked. Also worth noting: `bin/build`'s
   container floats on `archlinux:latest` + `pacman -Syu` with no pinned toolchain
   versions, so this broke between sessions purely from the build environment moving
   forward, not from anything changing in this repo — the same could happen again, in
   either direction, on any package's next rebuild.
