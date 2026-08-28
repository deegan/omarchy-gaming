# omarchy-gaming

A pacman repository delivering a curated gaming package stack for [Omarchy](https://omarchy.org),
following the same sovereign build philosophy as `omarchy-pkgs`: **no AUR at runtime** — everything
here is built and hosted from our own infrastructure, from source or verified upstream release
artifacts, with no `yay`/`paru` step anywhere in the chain.

## What's in here

| Tier | Packages | What it is |
|---|---|---|
| 1 | `proton-ge-custom`, `wine-ge-custom`, `protonup-qt`, `heroic-games-launcher-bin`, `steamtinkerlaunch`, `goverlay` | AUR-only tools, repackaged from upstream releases |
| 2 | `gamemode`+`lib32-gamemode`, `mangohud`+`lib32-mangohud` | Official Arch packages, rebuilt with opinionated tuned defaults (`/etc/gamemode.ini`, `/etc/MangoHud.conf`) |
| 3 | `omarchy-gaming-base`, `omarchy-gaming-settings`, `omarchy-gaming-nvidia`, `omarchy-gaming-amd` | Meta-packages: one pulls in the whole stack, one ships sysctl/udev/ananicy tuning, two are GPU-vendor-specific |
| 4 | `omarchy-kernel-gaming`(+`-headers`), `mesa`+`lib32-mesa` (and every `vulkan-*`/`opencl-mesa` driver package) | CachyOS's BORE-scheduler kernel and full Mesa driver stack, both rebuilt for an `x86-64-v3` CPU baseline instead of generic `x86-64` |

Tiers 2 and 4 intentionally rebuild packages that already exist in the official Arch repos —
that's the point of Tier 2/4: same package name, different build (tuned config, different
`CFLAGS`), installed as a drop-in replacement via normal `pacman` upgrade semantics. Tier 4 in
particular carries real risk: a bad Mesa build breaks every GPU-accelerated app on the system in
place, with no side-by-side fallback the way an alternate kernel entry gives you. Don't install it
on a machine you can't afford to be without graphics on for a while.

## Using this repo

On the machine you want to install packages on, edit `/etc/pacman.conf`:

1. **Enable multilib**, if not already (needed for every `lib32-*` package):
   ```ini
   [multilib]
   Include = /etc/pacman.d/mirrorlist
   ```

2. **Add this repo — above `[core]`/`[extra]`/`[multilib]`**, not below:
   ```ini
   [omarchy-gaming]
   Server = http://<server-ip-or-hostname>:8080
   SigLevel = Optional TrustAll
   ```
   The ordering matters: several packages here (`gamemode`, `mangohud`, `mesa`, ...) also exist in
   the official repos, and pacman resolves a same-named package using whichever repo is listed
   first in the file — including transitively, e.g. through `omarchy-gaming-base`'s dependency
   list. Below the defaults, our rebuilds silently lose to the vanilla ones and you won't get an
   error telling you so.

   `SigLevel = Optional TrustAll` is required because this repo isn't GPG-signed by default (see
   [Signing](#signing) below). Only point this at a repo you trust — untrusted-repo package
   installation runs arbitrary code as root.

3. Sync and install:
   ```bash
   sudo pacman -Sy
   sudo pacman -S omarchy-gaming-base                 # the full userspace stack
   sudo pacman -S omarchy-gaming-nvidia                # or -amd, depending on your GPU
   sudo pacman -S omarchy-kernel-gaming omarchy-kernel-gaming-headers   # optional, installs
                                                                        # alongside your current kernel
   ```

4. **Updating later:** use a full system upgrade, not `pacman -S omarchy-gaming-base` again —
   meta-packages don't carry a version bump just because their dependencies got new builds, so
   re-running `-S` on one sees "already satisfied" and does nothing. `pacman -Syu` checks every
   installed package against the sync databases independently and will pick up new `mesa`,
   `omarchy-kernel-gaming`, etc. builds correctly.

## Hosting your own instance

### Requirements

- Docker + Docker Compose — this is the only thing `bin/build` needs; every package builds inside
  a throwaway `archlinux` container, so the host itself doesn't need an Arch toolchain and doesn't
  even need to be Arch Linux. This repo has been built successfully from a NixOS host, for example.
- **`repo-add`, on the host**, for `bin/publish` (this one command runs directly on the host, not
  in Docker). On Arch, that's the `pacman-contrib` package. On a non-Arch host, the simplest fix is
  a throwaway shell with just that one binary in `PATH` — on NixOS, `pacman-contrib` isn't its own
  package (nixpkgs builds upstream pacman's whole source tree, `repo-add` included, as the single
  `pacman` derivation), so use:
  ```bash
  nix-shell -p pacman --run "bin/publish"
  ```
- A few GB of disk for the published packages (currently ~1.5GB; grows if you rebuild Tier 4 with
  debug packages included)
- If building from source rather than copying pre-built packages: expect the Tier 4 builds (kernel,
  Mesa) to take 30 minutes to a few hours combined, even on a many-core machine — see
  [Building packages](#building-packages)

### Fastest path: copy the already-built packages

If you just want to serve packages that are already built elsewhere, you don't need to rebuild
anything — `repo/` (gitignored, so it isn't in this git history) is a self-contained set of
`.pkg.tar.zst` files plus the pacman database. Copy it to the new machine and serve it:

```bash
# on the machine that already has a populated repo/
rsync -avz repo/ newserver:/path/to/omarchy-gaming/repo/

# on the new server
git clone <this-repo-url> omarchy-gaming
rsync -avz oldserver:/path/to/omarchy-gaming/repo/ omarchy-gaming/repo/
cd omarchy-gaming
docker compose up -d
```

### Building from source on the new server

```bash
git clone <this-repo-url> omarchy-gaming
cd omarchy-gaming
bin/build                 # builds every package — slow; see the Tier 4 warning above
                           # bin/build <pkgname> [<pkgname> ...] builds specific ones instead
bin/publish                # runs repo-add, refreshes the pacman database
docker compose up -d       # nginx now serves ./repo on :8080
```

`bin/build` runs each package in a clean, throwaway `archlinux` Docker container — it never builds
on the host, so the host doesn't need Arch's toolchain installed at all, just Docker. It also
mounts the local `repo/` into the build container as a `file://` pacman source (once `bin/publish`
has been run at least once), which is what lets meta-packages like `omarchy-gaming-base` resolve
dependencies on this repo's own other packages.

### Exposing it beyond a trusted LAN

`docker-compose.yml`'s `"8080:80"` port mapping binds to all interfaces by default, so the repo is
already reachable from other machines on the same network with no changes. If you want it reachable
over the internet rather than just a LAN:

- Put a reverse proxy (Caddy, nginx, Traefik) in front of it with a real TLS certificate — don't
  expose port 8080 directly to the internet over plain HTTP.
- Set up [signing](#signing) first. `SigLevel = Optional TrustAll` on the client is a reasonable
  tradeoff on a LAN you control; it's not one you want to ask anyone else to accept.
- Give the server a stable address (static IP or DNS record) — a repo whose URL changes out from
  under clients breaks silently, with pacman just reporting a 404 the next time someone syncs.

`restart: unless-stopped` in `docker-compose.yml` means the container comes back after a reboot as
long as Docker itself is enabled as a system service (`systemctl enable docker`).

## Signing

By default, `bin/publish` publishes an **unsigned** database (`repo-add` without `-s`), and
packages themselves aren't GPG-signed either. This is fine for local/LAN testing with
`SigLevel = Optional TrustAll`, but means there's no integrity check on anything this repo serves.

To sign:

1. Generate a GPG key (keep it outside this repo — it's not something to commit).
2. Set `GPG_KEY_ID` before running `bin/publish`:
   ```bash
   GPG_KEY_ID=<your-key-id> bin/publish
   ```
3. Distribute the public key to clients and change `SigLevel` in their `pacman.conf` from
   `Optional TrustAll` to something that actually verifies, e.g. `Required DatabaseOptional`, after
   importing the key with `pacman-key`.

Package-level signing (`makepkg`'s own `--sign`) isn't wired into `bin/build` yet — currently only
the database itself can be signed via the mechanism above.

## Building packages

- `bin/build` with no arguments builds every `PKGBUILD` under `pkgbuilds/`. `bin/build <name>
  [<name> ...]` builds specific packages.
- `bin/publish` runs `repo-add` against everything currently in `repo/*.pkg.tar.zst` and refreshes
  the pacman database. Run it after every `bin/build`.
- To add a new package: create `pkgbuilds/<name>/PKGBUILD` following the conventions below, then
  `bin/build <name>` and `bin/publish`.

### Conventions

- **Binary repacks** (Tier 1-style): fetch the upstream release asset, verify its checksum against
  what the upstream release manifest/API already publishes — never download an artifact purely to
  compute a hash yourself if the checksum is already published somewhere.
- **Compiled packages**: always built in a clean `archlinux` Docker container via `bin/build`,
  never on the host.
- `pkgrel` resets to `1` on every `pkgver` bump.
- Packages that already exist unchanged in the official Arch repos don't belong here — only add one
  if it's patched, recompiled with different flags, or bundles opinionated config on top.
- Nothing under `repo/` is committed — it's build output, not source.

## Troubleshooting

- **`pacman -Sy` fails with a 404 on `omarchy-gaming.db`, but `curl` can fetch the same URL fine**:
  almost always a typo in the `Server =` line (missing `://`, a leftover `$repo`/`$arch` template
  variable from a different repo layout, or a section-name mismatch — pacman requests
  `<section-name>.db`, so `[omarchy-gaming]` must produce `omarchy-gaming.db`, matching exactly).
- **Installed `omarchy-gaming-base` but still got the vanilla `mesa`/`gamemode`/etc.**: check repo
  ordering in `pacman.conf` — see the warning in [Using this repo](#using-this-repo).
- **Ran `pacman -S omarchy-gaming-base` again and nothing updated**: use `pacman -Syu` instead — see
  point 4 in [Using this repo](#using-this-repo).
- **`lib32-*` packages "target not found"**: multilib isn't enabled in `pacman.conf`.

## Known limitations

- The repo is unsigned by default (see [Signing](#signing)).
- Mesa's `x86-64-v3` rebuild has been verified to install cleanly and to actually contain
  `x86-64-v3`-targeted code (confirmed via disassembly showing AVX2 instruction usage), but hasn't
  yet been verified to run a real graphical session without issue on physical hardware — it has no
  side-by-side fallback if something's wrong, unlike the kernel.
- `omarchy-kernel-gaming` has been boot-tested successfully on real hardware, installed alongside a
  stock kernel.

## License

This repo's own original content (the PKGBUILDs, `bin/build`/`bin/publish`, `docker-compose.yml`,
and the config files we wrote — `gamemode.ini`, `MangoHud.conf`, the `omarchy-gaming-settings`/
`-nvidia`/`-amd` tuning) is [MIT](LICENSE), matching `omarchy` and `omarchy-pkgs`.

That covers our own glue code, not what it builds: every package here still carries whatever
license its upstream project uses (Mesa's is MIT/BSD-ish, `gamemode`'s is BSD-3-Clause, MangoHud's
is MIT, the Linux kernel's is GPL-2.0, and so on) — packaging something doesn't relicense it. One
specific exception worth naming: the kernel `.config` and BORE scheduler patch vendored into
`pkgbuilds/omarchy-kernel-gaming/` are pulled directly from `CachyOS/linux-cachyos`, which is
GPL-3.0-licensed.
