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
