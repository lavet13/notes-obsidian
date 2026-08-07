---
Docs: https://github.com/Alexays/Waybar/wiki  ·  config: .../wiki/Configuration  ·  styling: .../wiki/Styling
---

## config.jsonc — which modules show

`modules-left/center/right` arrays decide what RENDERS. A module's config block (e.g. `"battery": {…}`)
is DORMANT unless the module name is also in one of the arrays — so to drop a module you just remove it
from the array; you don't need to delete its config block.

```jsonc
"modules-right": ["idle_inhibitor","pulseaudio","network","cpu","memory",
                  "temperature","sway/language","clock","tray"],
"sway/language": { "format": "{short}", "tooltip-format": "{long}" },   // us/ru indicator
"bluetooth":     { "format": " {status}", "on-click": "kcmshell6 kcm_bluetooth" },  // reads bluez
                 // over D-Bus (no applet needed); on-click opens KDE's Bluetooth page. A module
                 // reacts to clicks ONLY if given on-click; otherwise it's status text, not a button.
```

## style.css — GTK3 CSS

- `@define-color name #hex;` then `@name` — single source of truth for the palette.
- Selectors: `#clock` = module id; `.class` = a live STATE class waybar adds
  (`#network.disconnected`, `#battery.charging`, `#workspaces button.focused`).
- Cascade: more-specific / later rule wins → state rules sit after base rules.
- `box-shadow: inset 0 -3px @mint` = the underline trick (3px line pinned to the bottom edge,
  drawn WITHOUT shifting the text — why waybar uses it over a border).
- Translucency: `background-color: alpha(@bg, 0.85);`

## Reload without restarting sway

```bash
killall -SIGUSR2 waybar   # re-reads config + style.css in place
```
