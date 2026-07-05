---
id: wezterm-knowledge
aliases: []
tags: []
---

## Keybinding resolution

Two entries bound to the same key+mods -> the LAST one in the keys table wins; the
earlier one silently does nothing. Audit with:

```bash
wezterm show-keys        # prints the resolved binding table (duplicates collapsed)
```

Fix a conflict by moving one action to a free key. Keep directional sets intact
(e.g. CTRL|SHIFT h/j/k/l for panes); relocate the OTHER action -- e.g. put the
launcher on CTRL|SHIFT p (ShowLauncher) rather than stealing 'l' from pane-right.
