---
id: yazi-knowledge
aliases: []
tags: []
Docs: "https://yazi-rs.github.io/docs/configuration/overview  ·  keymap: .../configuration/keymap  ·  theme: .../configuration/theme"
Version note: written against yazi 26.5.6 (CalVer). Config MERGES over built-in defaults, so a partial file is complete.
---

## Three files

`yazi.toml` behaviour · `theme.toml` colours · `keymap.toml` keys. **26.5.6 renamed `[manager]` → `[mgr]`**;
tabs moved to their own `[tabs]` section; there's no `hovered` key under `[mgr]`.

## theme.toml — filetype rules key on `url`, NOT `name`

```toml
[filetype]
rules = [                       # matched TOP-TO-BOTTOM, first hit wins → catch-all LAST
  { url = "*/",       fg = "#66d9ef" },   # url = "*/"  → all DIRECTORIES
  { mime = "image/*", fg = "#87ffde" },
  { url = "*.md",     fg = "#3ad0b5" },
  { url = "*",        fg = "#d0b892" },    # url = "*"   → all files
]
```
`name` is the `[icon]` section's key — using it in `[filetype]` errors
"at least one of `url` or `mime` must be specified".

## yazi.toml — openers

```toml
[opener]
edit = [ { run = 'nvim "$@"', block  = true, for = "unix" } ]   # block  = TUI takes the terminal
play = [ { run = 'vlc  "$@"', orphan = true, for = "unix" } ]   # orphan = GUI detaches, keeps running
[open]
prepend_rules = [                         # prepend = higher priority than defaults
  { mime = "text/*",  use = "edit" },
  { mime = "audio/*", use = "play" },
  { mime = "video/*", use = "play" },
]
```
`o`/`Enter` opens via the matched rule; `O` = interactive "open with" chooser.

## keymap.toml — shell template

`prepend_keymap` wins over defaults. The `shell` command template placeholders:
`%h` = hovered PATH, `%H` = hovered URL (already `file://…`), `%s`/`%S` = selected paths/URLs;
a trailing `--` disables all escaping (everything after is a raw string).
```toml
[mgr]
prepend_keymap = [
  # Y → put the hovered file on the clipboard AS A FILE (uri-list), pasteable into Telegram etc.
  { on = "Y", run = '''shell -- printf '%s' "%H" | wl-copy -t text/uri-list''', desc = "Copy file URI" },
]
```

## Delete & trash

`d` = move to trash (FreeDesktop spec → `~/.local/share/Trash/`, recoverable); `D` = permanent
(confirmation). Recover: `gio trash --restore <p>` needs the gvfs backend; simpler is trash-cli:
`trash-list`, `trash-restore` (put back), `trash-empty` / `trash-rm <name>` (destroy).

## Hotkey cheat-sheet (defaults)

```
h/j/k/l  move (h=up a dir, l=enter/open)      /  find        n/N next/prev find
Space    toggle-select one    v visual-select   y yank(copy)   x cut   p paste
d  trash    D  delete-perma    a create (end / = dir)   r rename    .  toggle hidden
o/Enter open (matched rule)    O open-with        Tab spot info    q quit   ~ help
gg / G   top / bottom          -  leave dir       Enter/l enter dir
```
