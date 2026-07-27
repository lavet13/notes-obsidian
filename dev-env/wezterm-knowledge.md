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

## One config for Windows + Linux (target_triple)

`wezterm.target_triple` is a runtime string like `x86_64-pc-windows-msvc` or
`x86_64-unknown-linux-gnu`. Branch on it so the SAME tracked file works on both
machines (then just symlink it — no per-host patching).

```lua
local is_windows = wezterm.target_triple:find("windows") ~= nil

-- A table LITERAL can't contain an `if` (that's a statement, not an expression),
-- so compute the OS-varying values as locals ABOVE the return, reference below.
local default_prog, launch_menu           -- declare in outer scope (empty)
if is_windows then
  default_prog = { "G:/Programs/Git/bin/bash.exe" }
  launch_menu  = { { label = "Debian", args = { "wsl.exe","-d","Debian","--cd","~" } } }
else
  default_prog = { "/usr/bin/zsh", "-l" }
  launch_menu  = { { label = "zsh", args = { "/usr/bin/zsh","-l" } } }
end

return {
  default_prog = default_prog,   -- left = table key wezterm reads; right = our local
  launch_menu  = launch_menu,
  -- ... rest unchanged ...
}
```

Gotcha: `local x = {...}` INSIDE the `if` is block-scoped and gone by the
`return`. Declare `local default_prog` before the `if`, assign inside.
