---
id: arch-knowledge
aliases: []
tags: []
---

# Arch / CachyOS Knowledge

Coming from apt: pacman is the official manager; paru adds the AUR on top.

## pacman — the essentials (apt map)

```bash
sudo pacman -S <pkg>      # install            (apt install)
sudo pacman -Syu          # sync db + upgrade   (apt update && apt upgrade)
sudo pacman -Rns <pkg>    # remove + orphaned deps + leftover config
sudo pacman -Rdd <pkg>    # force-remove, skip dep checks (swapping a conflict)
pacman -Ss <term>         # search AVAILABLE    (apt search)
pacman -Qs <term>         # search INSTALLED
pacman -Q  <pkg>          # is exactly this installed? (exit 0 + version)
```

- `S` = sync (remote) db, `Q` = query local db, `R` = remove.
- `-Rdd`: skips "but X needs this" AND removing deps. Use when replacing in the
  same breath: `sudo pacman -Rdd wezterm && paru -S wezterm-nightly-bin`.
- Multiple providers prompt ("N providers available for X") = which repo for a
  shared dep. Equivalent — press the default (1)/Enter.

## Pinning / not upgrading a package

```ini
# /etc/pacman.conf, under [options]:
IgnorePkg = amneziavpn-bin      # -Syu will skip it (space-separate for many)
```

## paru + the AUR

The AUR is community BUILD RECIPES (PKGBUILD), not prebuilt binaries. paru
(an AUR helper, wrapper over pacman) fetches the recipe, builds locally, hands
the result to pacman. It shows the PKGBUILD to REVIEW first — that's the
security step (recipes are unvetted). `-bin` packages just fetch the upstream
binary, so their PKGBUILD is short.

```bash
sudo pacman -S paru       # paru itself is in CachyOS's repos
paru -S <pkg>             # install from AUR (or repos — paru passes through)
paru -Ss <term>           # search repos AND AUR (pacman -Ss = repos only)
```

## Downgrading / pinning an AUR package

```bash
paru --getpkgbuild <pkg>              # clone recipe into ./<pkg>
cd <pkg>
# edit PKGBUILD: pkgver=<old>  + any hardcoded version in source=()
sudo pacman -S --needed pacman-contrib   # provides updpkgsums
updpkgsums                            # recompute checksums for the new file
makepkg -si                           # build + install
# then pin it (IgnorePkg above) so -Syu won't bump it back
```

## systemd services

```bash
sudo systemctl enable --now <name>   # start now + at every boot (no --now = boot only)
systemctl status <name>              # active? PID? recent logs?
sudo systemctl disable --now <name>  # stop now + not at boot
```

- `status` line `enabled; preset: disabled`: **enabled** = the state you set;
  **preset** = the distro default (here off). Your setting wins — not a conflict.

## Locales (no locales-all package on Arch)

```bash
sudo sed -i 's/^#\(en_US.UTF-8 UTF-8\)/\1/' /etc/locale.gen   # uncomment
sudo locale-gen                                                # generate
locale -a | grep -i en_US                                      # verify
```

KDE's Region KCM can't auto-generate on Arch (it warns) — this is the manual
route. Add `ru_RU.UTF-8` the same way if you want Russian date/currency formats.

## KDE default terminal (from the CLI)

```bash
kwriteconfig6 --file kdeglobals --group General --key TerminalApplication wezterm-here.desktop
kbuildsycoca6      # rebuild the desktop/service cache
kreadconfig6 --file kdeglobals --group General --key TerminalApplication   # verify
```

- Value is a `.desktop` FILENAME (resolved from `~/.local/share/applications`),
  not a raw command. Swapping terminals in the GUI can reset it to a bare
  command string — re-set via CLI.

### A terminal .desktop launcher — the two gotchas

```ini
[Desktop Entry]
Type=Application
Exec=wezterm start --cwd %f     # %f = folder KDE passes ("Open Terminal Here")
Terminal=false                  # the app IS a terminal; =true makes KDE wrap it
                                # in ANOTHER terminal and can swallow Exec args
NoDisplay=true
```

- Duplicate `Exec=` lines → parser uses the first, behaviour undefined. Keep one.
- `wezterm start` hands off to an already-running WezTerm GUI, which spawns the
  shell from ITS env (losing any `env VAR=` you set on the launcher). Add
  `--always-new-process` if you need the launcher's env to reach the shell.

## AMD Ryzen: random instant-off reboots at idle (hardware, but lookup-worthy)

Signature: instant black + reboot, NO blue screen, at idle/light load; stress
tests stay stable. In Windows: Event Viewer Kernel-Power **41**, `BugcheckCode
0` (= power lost, nothing caught). Cause: deep CPU C-states drop draw so low the
PSU can't hold the rail. NOT necessarily a dead PSU.

Fix in BIOS → Advanced → AMD CBS:
- `Global C-state Control` = **Disabled** (reliable; the one that matters)
- `Power Supply Idle Control` = **Typical Current Idle** (if the AGESA exposes it)

## Flatpak (a second package system, alongside pacman)

Sandboxed apps from Flathub, independent of pacman/AUR. Used for things not in
the repos or that need isolation (e.g. Sober = Roblox on Linux).

```bash
sudo pacman -S --needed flatpak                                    # one-time
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install flathub <app-id>     # e.g. org.vinegarhq.Sober
flatpak run <app-id>                 # launch from the terminal
```

Everyday:

```bash
flatpak list                 # installed apps + runtimes
flatpak update               # update ALL flatpaks  (NOT covered by pacman -Syu!)
flatpak update <app-id>      # just one
flatpak info <app-id>        # installed version / branch / commit / date
flatpak uninstall <app-id>
flatpak uninstall --unused   # drop runtimes nothing needs anymore
```

Gotchas:
- Updates are SEPARATE from pacman. `pacman -Syu` does not touch flatpaks — keep
  current with BOTH.
- After the FIRST flatpak install, log out/in. Until the session restarts,
  `XDG_DATA_DIRS` doesn't include flatpak's exports, so the app won't appear in
  the menu (it warns about exactly this). Use `flatpak run <id>` meanwhile.
- First install pulls large shared runtimes (GL drivers, GNOME/KDE platform,
  codecs — ~1 GB); later flatpaks reuse them, so it's a one-time cost.
- A broken flatpak app (e.g. Sober after a Roblox update): `flatpak update` it,
  then `flatpak info` its version and compare to the upstream releases/issues.

## pacman — flag anatomy (every letter is one switch)

Combined flags stack in any order; each letter is independent.

```bash
# -S  sync: act on REMOTE repos
#   y   refresh the LOCAL copy of the package DATABASES from the mirror
#       (the `apt update` index-fetch half)
#   u   upgrade every installed pkg now out of date vs those dbs
#   → -Syu = refresh THEN upgrade = the one true update command
#   --needed  skip pkgs already current (don't reinstall) → makes -S idempotent,
#             which is why setup scripts use it: pacman -S --needed flatpak
#
# -R  remove
#   s   recursive: also remove deps nothing else needs
#   n   nosave: delete the pkg's config files too (else kept as *.pacsave)
#   d   nodeps (version checks); -dd skips dep checks ENTIRELY, incl.
#       "but X requires this" — deliberate conflict swap only:
#       sudo pacman -Rdd wezterm && paru -S wezterm-nightly-bin
#
# -Q  query the LOCAL db (installed pkgs):
#   s search  i info  o owns-file  l list-files
#   e explicitly-installed  d installed-as-dep  t required-by-nothing

makepkg -si   # in a dir with a PKGBUILD (manual/edited recipe; paru does this for you)
#   -s  syncdeps: pacman-install missing build/runtime deps first
#   -i  install the built package afterward
```

Reverse of any install is removal: `sudo pacman -S <pkg>` ↔ `sudo pacman -Rns <pkg>`.

## The partial-upgrade footgun — never `-Sy` alone

```bash
# NEVER: sudo pacman -Sy <pkg>   (refresh dbs, upgrade only one thing)
```
Arch has no stable snapshot — every package is built against the CURRENT version
of every other. Pull one fresh package onto an otherwise-stale system and you get
a *partial upgrade*: mismatched library sonames, things segfault. Always refresh
and upgrade together: `-Syu`.

`-Syy` (double y) force-redownloads the dbs even if they look current — needed
ONLY right after switching/re-ranking mirrors (`/etc/pacman.d/mirrorlist`, via
`rate-mirrors` on CachyOS or `reflector` on Arch), where a new mirror serves dbs
pacman thinks are identical. Still pair with u: `-Syyu`.

## Overriding IgnorePkg — updating a pinned package anyway

An `IgnorePkg` line (see Pinning) makes `-Syu` skip that package — you'll see
`[ignored]` / `[игнорировано]` in the update output. To update it once WITHOUT
unpinning permanently, name it explicitly:

```bash
paru -S amneziavpn-bin
# pacman still sees IgnorePkg and PROMPTS: "...install anyway? [Y/n]" → Y.
# Overrides the ignore for THIS run only; the IgnorePkg line stays.
```

A package upgrade replaces files under `/usr` and `/opt` — it never touches your
`$HOME` config. But a MAJOR bump can migrate the config format on first launch
(one-way), so back up first:

```bash
cp -r ~/.config/AmneziaVPN ~/AmneziaVPN.config.bak    # BACK UP (find real path first)
# reverse: rm -rf ~/.config/AmneziaVPN && mv ~/AmneziaVPN.config.bak ~/.config/AmneziaVPN
#          then downgrade the pkg via the Downgrading recipe above
```

## Routine maintenance (each with its reverse)

```bash
# Orphans — deps left behind, now required by nothing:
pacman -Qdtq                                  # list (review before removing!)
pacman -Qdtq > /tmp/orphans.txt               # save the list = your undo
sudo pacman -Rns $(cat /tmp/orphans.txt)      # remove
# reverse: sudo pacman -S $(cat /tmp/orphans.txt)

# Cache never self-clears (/var/cache/pacman/pkg grows forever):
paccache -rk2          # keep latest 2 of each, delete the rest (pacman-contrib)
# do NOT use `pacman -Scc` — deletes ALL cached pkgs = kills offline downgrade,
# which is: sudo pacman -U /var/cache/pacman/pkg/<pkg>-<oldver>.pkg.tar.zst

# Stale lock ("unable to lock database") — a killed pacman left db.lck:
pgrep -a pacman                               # MUST be empty first
sudo rm /var/lib/pacman/db.lck                # then remove; recreated next run

# .pacnew — update shipped a new default for a config you edited; pacman writes
# <config>.pacnew beside yours instead of clobbering. Merge or drift silently:
find /etc -name '*.pacnew'                     # locate
sudo pacdiff                                   # interactive merge (pacman-contrib)
# reverse: cp foo foo.bak BEFORE accepting a .pacnew, so you can restore

# Keyring stale (signature errors on -Syu after a long gap):
sudo pacman -Sy archlinux-keyring cachyos-keyring && sudo pacman -Su
```

## Handy query verbs

```bash
pacman -Qo <file>   # which installed pkg OWNS this file
pacman -Ql <pkg>    # LIST every file that pkg installed
pacman -Qi <pkg>    # info: deps, size, install date
pacman -Qet         # only what I EXPLICITLY installed and nothing needs
                    #   (my real "what did I add" list)
```

## Login manager: SDDM → plasmalogin (KDE)

CachyOS KDE now uses **plasmalogin** (Plasma Login Manager), not SDDM (Simple
Desktop Display Manager). It's a stripped-down continuation of SDDM shipping with
Plasma 6.6. Session picker reads the same dirs, so adding Sway still works:

```bash
systemctl status display-manager      # confirms plasmalogin.service here
ls /usr/share/wayland-sessions/        # a sway.desktop here appears in the picker
```

## pacman flags — `-d` and `-q` demystified

`-d`'s meaning depends on the OPERATION letter it rides with — same letter, two jobs:
- under -R (remove): `-d` = --nodeps → SKIP dependency checks (-dd skips them all,
  incl. "but X requires this")
- under -Q (query):  `-d` = --deps   → FILTER to pkgs installed AS a dependency
So -Rdd (suppress dep logic) and -Qdt (dep-installed AND required-by-nothing = an
orphan) share the letter but not the meaning.

`-q` = --quiet: output bare NAMES only, no version column. Needed whenever the result
is piped into another pacman call, which wants names, not "name version":

    pacman -Qdt   → litehtml0.9 0.9-2.1      (name + version)
    pacman -Qdtq  → litehtml0.9              (name only)
    sudo pacman -Rns $(pacman -Qdtq)         # -q is what makes this substitution valid

Also works on -Ss / -Qs searches.

## Writing a distro ISO to USB (skip balena-etcher on Arch)

balena-etcher from the AUR drags in electron + rust FROM SOURCE (multi-hour build)
and its dep chain conflicts (nodejs-lts-iron vs -jod both claim `nodejs`). Not worth it.
- Native + light (KDE, in repos):  sudo pacman -S --needed isoimagewriter
- Etcher's verify-after-write, no build:  the official .AppImage (github releases)
- Distro-hopping? ventoy-bin (AUR): flash the stick ONCE, then just drop .iso files
  on it and pick from a boot menu. No reflash per distro.

## paru: clean leftover AUR build files

paru clones each PKGBUILD + downloads sources under ~/.cache/paru/clone; these pile
up, and a failed build leaves its tree behind.

    paru -c        # remove untracked/leftover AUR build files (regenerable — safe)

Only touches ~/.cache/paru; the repo cache (/var/cache/pacman/pkg) is paccache's job.

## Gotcha: a transaction that fails at "checking conflicts" installs NOTHING

pacman transactions are atomic. "failed to prepare transaction (conflicting
dependencies)" happens during PREPARE, before any file is written — so a build that
dies there (e.g. the balena-etcher nodejs conflict) leaves your system unchanged.
Verify: pacman -Qq <pkg> → "not found" = nothing installed. Then paru -c to clear
the build clones.
