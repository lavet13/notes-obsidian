---
id: sway-knowledge
aliases: []
tags: []
Docs: https://man.archlinux.org/man/sway.5  ·  https://github.com/swaywm/sway/wiki
---

## Config model

```
set $mod Mod4           # define a var; reference as $mod
bindsym $mod+Return exec $term   # bind keys → action. `exec` = run a shell command;
                                 # without exec the action is an internal sway cmd (kill, fullscreen…)
exec        foo         # run ONCE at session start (NOT re-run on reload)
exec_always foo         # run at start AND on every `reload` (dups unless guarded)
include /etc/sway/config.d/*
```
Gotcha: after editing, `reload` (Super+Shift+c) does NOT re-fire `exec` lines — only a fresh
session start does. So a new `exec waybar`/`exec AmneziaVPN` needs logout/in or a manual run.

## Layout tree & focus

Windows live in a TREE of containers. `focus parent` ($mod+a) climbs to the container holding
the current group; `focus child` ($mod+Shift+a) descends back; any `focus <dir>` also descends.
No undo for `kill` ($mod+Shift+q) — once the app exits the window is gone (keep un-restorable
state inside tmux, which survives a closed terminal window).

## Criteria selectors + `swaymsg -t`

```bash
swaymsg -t get_tree      # full window tree — app_id, pid, workspace of every con
swaymsg -t get_outputs   # monitors, current mode, ALL available modes (res@Hz)
swaymsg -t get_inputs    # every input device + its identifier (for per-device config)
# Criteria [ ... ] target a window by property, then run a command on it:
swaymsg '[app_id="org.telegram.desktop"] move workspace current, focus'   # summon it here
```
A single-instance app (Telegram) relaunched just focuses its existing window on its own
workspace — SUMMON it with the criteria selector instead of relaunching.

## Output: mode + background

```
output HDMI-A-1 mode 1920x1080@200Hz    # sway won't pick your high refresh; pin it (get_outputs)
output * bg '/path/to/img.png' fill      # image; `fill` scales to cover
output * bg #041a19 solid_color          # solid colour
```
GOTCHA: `bg` is painted by **swaybg** (optional dep). Not installed ⇒ every background is BLACK
regardless of the directive. Two `bg` lines for `*` ⇒ last wins. KDE wallpapers:
`/usr/share/wallpapers/<Name>/contents/images/<res>.png`.

## Input

```
input type:keyboard { xkb_layout "us,ru"  xkb_options "grp:alt_shift_toggle" }  # Alt+Shift cycles
input type:pointer  { accel_profile flat  pointer_accel 0.7 }  # flat = no accel; accel = -1..1 scalar
```

## Idle (swayidle)

```
exec swayidle -w \
    timeout 300 'swaymsg "output * power off"' resume 'swaymsg "output * power on"' \
    timeout 900 'systemctl suspend' \
    before-sleep 'swaylock -f -c 000000'
```
`-w` wait per cmd; `timeout N 'cmd'` after N s idle; `resume` pairs with the timeout above it;
`before-sleep` runs before suspend. `swaymsg "output * power off"` = DPMS off (binary, no % dim).
A % brightness dim needs hardware: `brightnessctl` only sees `/sys/class/backlight` (laptop panels;
empty on a desktop) — external monitors need `ddcutil` (DDC/CI over i2c).

## Screenshots (grim + slurp)

```
bindsym $mod+Shift+s exec grim -g "$(slurp -d -c 3ad0b5ff -s 3ad0b533 -w 2)" - | wl-copy
bindsym $mod+p       exec grim - | wl-copy
```
`slurp` prints a region → `grim -g` captures it; `grim -` → stdout → `wl-copy` = clipboard.
slurp flags: `-d` show dims, `-c` border, `-s` fill, `-w` width, `-b` dim-rest (drop it if too dark).

## Launcher (fuzzel)

`~/.config/fuzzel/fuzzel.ini`, `[colors]` = RRGGBBAA. Fuzzel reads config FRESH each launch (no
reload daemon). `[main] terminal=foot` is required for Terminal=true apps (btop) — else "No tty".

## System dark mode (no DE)

The system pref is a PORTAL setting apps query; set it via gsettings (needs xdg-desktop-portal-gtk,
already pulled by sway):
```bash
gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'   # Firefox/Chrome/Electron/GTK4
gsettings set org.gnome.desktop.interface gtk-theme 'Adwaita-dark'
printf '[Settings]\ngtk-application-prefer-dark-theme=1\n' >> ~/.config/gtk-3.0/settings.ini  # GTK3 stragglers
```
Qt is a SEPARATE track — GTK settings don't touch it. Reuse KDE's Breeze:
```bash
printf 'QT_QPA_PLATFORMTHEME=kde\n' > ~/.config/environment.d/qt.conf   # read at SESSION START (relog)
plasma-apply-colorscheme Naysayer
```
WPS Office ignores all of it (own skin engine) — set a dark skin inside WPS.

## Bluetooth (bluedevil under sway — GUI, no bluetoothctl)

KDE's own stack, **bluedevil**, works standalone under sway — but recent versions (6.7.x) ship NO
`bluedevil-monolithic` tray/agent binary (verify: `pacman -Ql bluedevil | grep /usr/bin/` → only
`bluedevil-wizard` + `bluedevil-sendfile`). So there's no persistent applet, and you don't need one:
- waybar's `bluetooth` module reads bluez directly over D-Bus → status shows with nothing running.
- `on-click: kcmshell6 kcm_bluetooth` opens the familiar KDE Bluetooth page — connect / disconnect /
  remove / trust, and "Add device" (which launches `bluedevil-wizard`, its own pairing agent).
- Trust a device once there → bluez auto-reconnects it on boot; no agent needs to stay running.
- So NO `exec` autostart line for Bluetooth at all.

GOTCHA (why blueman polluted KDE): blueman ships `/etc/xdg/autostart/blueman.desktop`, which fires in
EVERY XDG session — installing it for sway also spawned a 2nd applet under KDE (already has bluedevil).
`pacman -Rns blueman` removes the pkg AND that autostart entry.

## Autostart tray race

Tray apps launched by `exec` race waybar; if they dock before waybar's `tray` module exists they
vanish. Delay them like KDE's autostart did:
```
exec sh -c "sleep 3 && arch-update --tray"
```

## KDE ↔ sway coexistence

Separate sessions (one at a time, picked at login); sway is a bare compositor, so nothing KDE
auto-runs under it unless you `exec` it. They interact ONLY through shared on-disk config.

- **Shared by design (good):** `~/.config/kdeglobals` (Naysayer colours), dconf/gsettings (cursor),
  `/etc/environment`. Both sessions read them → theming stays consistent across both.
- **KDE apps under sway:** individual Qt/KDE apps run fine (Dolphin, `kcmshell6 <kcm>`, Kate). What
  you CAN'T run is the Plasma SHELL (plasmashell, kwin) — they expect to BE the compositor.
- **The one clash class — autostart:** `/etc/xdg/autostart/*.desktop` fires in EVERY XDG session
  (that's how blueman leaked into KDE). Keep sway startups as `exec` lines in the sway config, NOT
  as .desktop files there, so they stay sway-only.
- **Never clashes:** keybinds (each compositor owns its own), window rules, panels (waybar vs KDE's
  panel — different sessions, never both at once).
- **kded6 sync modules read KDE's config as source of truth.** `kded6` (startable under sway
  to regen GTK `colors.css`) also carries a cursor-sync module that reads `~/.config/kcminputrc`
  → so if that still says `breeze_cursors`, loading kded6 clobbers your gsettings cursor back to
  Breeze. Fix the SOURCE, not just the destinations:
```bash
  kwriteconfig6 --file kcminputrc --group Mouse --key cursorTheme Adwaita
```
  General rule: kded6's modules sync FROM KDE config files (`kcminputrc`, `kdeglobals`) TO
  gsettings/GTK — align the KDE source or the sync overwrites you.

Rule: share theme/env files (deliberate), keep startups in the sway config, never run plasmashell/kwin under sway.

## Default apps (xdg-mime / mimeapps.list)

`xdg-open <file>` chooses the app from the file's MIME type → that type's default `.desktop` handler.
Fix "opens in the wrong app" by reassigning the type. NOTE: this governs `xdg-open` (yazi, file
managers, terminal) — some apps (Telegram Desktop) have their OWN open logic and ignore it.

```bash
xdg-mime query filetype file.xlsx     # print the file's MIME type
xdg-mime query default  <mimetype>    # print the .desktop that currently handles that type
xdg-mime default wps-office-et.desktop <mimetype> [<mimetype>...]   # reassign the default
```

- `query filetype <file>` — reports the MIME type by content/extension (e.g. xlsx →
  `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`). You need this to know
  which type to reassign.
- `query default <mimetype>` — reports which `.desktop` wins for that type right now (the diagnosis:
  if it says `firefox.desktop`, that's why it opens in the browser).
- `default <desktop> <mimetype…>` — sets the handler. It **writes to `~/.config/mimeapps.list`**
  (your personal file), which OVERRIDES system-wide defaults — so it sticks per-user and survives
  updates. You can pass several MIME types in one call to cover xlsx/xls/ods/csv together.

Gotcha: a duplicate entry under `[Default Applications]` in `~/.config/mimeapps.list` OR
`~/.local/share/applications/mimeapps.list` can shadow your change — dedupe if a type still
mis-routes. WPS desktops: `wps-office-et` (Sheets), `-wps` (Writer), `-wpp` (Presentation).

<!-- Docs: https://wiki.archlinux.org/title/XDG_MIME_Applications -->
