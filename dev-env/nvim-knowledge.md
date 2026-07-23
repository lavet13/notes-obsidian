---
id: nvim-knowledge
aliases: []
tags: []
---

## `loaded_*` variables are GUARDS, not status flags

Every distributed Vim/Neovim plugin opens with a guard of roughly this shape:

    if exists('g:loaded_matchit') | finish | endif
    let g:loaded_matchit = 1

So the variable means two opposite things depending on WHO set it:

- set by ME (before load) -> "don't load this plugin" (suppression)
- set by the PLUGIN (on load) -> "already loaded, don't load twice" (idempotence)

Both end up as `1`. Therefore `exists('g:loaded_matchit')` and
`echo g:loaded_matchit` CANNOT tell you whether the plugin is active — the
answer is 1 either way. It is not a diagnostic.

What actually distinguishes them: does the plugin's _effect_ exist?

    :echo exists(':MatchDebug')   " 2 = matchit's command is defined -> plugin sourced
    :map %                        " is the user-facing mapping installed?
    :echo b:match_words           " is the per-filetype data set? (ftplugin's job)

Corollary: if a plugin you removed left a `vim.g.loaded_X = 1` behind in your
config (vim-matchup does this to displace matchit), the built-in stays disabled
long after the plugin is gone. Deleting the plugin is not deleting its guard.

## matchit: what provides `%`, and the three places it can break

`%` has two layers, and they fail independently:

1. **Bracket matching** — core (`matchpairs`), always present. `%`, `d%`, `c%`
   on `(`/`[`/`{` works with zero plugins.
2. **Tag / keyword matching** (`<div>` <-> `</div>`, `if`/`endif`) — matchit,
   an OPTIONAL built-in package at `$VIMRUNTIME/pack/dist/opt/matchit/`,
   driven by a per-buffer variable `b:match_words` that the FILETYPE PLUGIN sets.

So a working tag-`%` needs all three of:

- the package loaded (guards unset, or `packadd matchit`)
- `b:match_words` set for the buffer (ftplugin sets it)
- the `%` mapping installed (also arrives via the ftplugin path)

The confusing failure mode is having 1 and 2 but not 3: `b:match_words` prints a
perfect HTML tag pattern, brackets still jump, and tags do nothing.

### The `no_plugin_maps` trap

    vim.g.no_plugin_maps = true   -- disables ALL built-in ftplugin mappings

Commonly copied from a nvim-treesitter-textobjects setup "to avoid conflicts".
It suppresses matchit's tag `%` (a mapping) while leaving `b:match_words` (a
variable) untouched — which is exactly the "1 and 2 but not 3" state above.

Fix: scope it per-filetype instead of globally (`vim.g.no_python_maps`,
`vim.g.no_ruby_maps`, ...), or drop it entirely until an actual collision shows up.

### Don't hand-map `%` to the `<Plug>`

    -- DON'T
    vim.keymap.set({"n","x","o"}, "%", "<Plug>(MatchitNormalForward)")

This _looks_ like it restores tag jumping and does — while breaking `d%`, `c%`,
`y%`, because the `<Plug>` motion doesn't compose in operator-pending the way
the built-in `%` does. General rule: when a plugin's own mapping didn't install,
find out WHY rather than substituting your own; a plugin that ships a mapping
wants to own it across all modes.

## Bisecting a config with `--clean`

The one check that can actually fail, and therefore the one worth running first:

    nvim --clean +"packadd matchit" +"e test.tsx"

`--clean` = no vimrc, no plugins, no shada. If the behavior WORKS there and not
in your config, the difference is your config — then bisect by commenting
suspects (start with anything that sets `vim.g.*` before plugins load). If it's
broken in `--clean` too, the install/runtime is at fault, not your setup.

Use this before theorizing: it splits "my config" from "nvim itself" in one run,
whereas most `echo`-style checks return the same value under both hypotheses.

## Insert-mode indent: `Ctrl-T` / `Ctrl-D`

    Ctrl-T   shift current line RIGHT one 'shiftwidth'
    Ctrl-D   shift current line LEFT  one 'shiftwidth'

Both stay in insert mode and work from any cursor column on the line.
`0` then `Ctrl-D` strips all indent from the line.

(Normal-mode equivalents are `>>` / `<<`, visual `>` / `<`.)

## nvim-treesitter-textobjects — what it's actually for

Semantic, parse-tree-based text objects instead of `iw`/`ip` heuristics:

    vaf / vif   a function / inside function
    vac / vic   a class / inside class
    daa / dia   an argument (parameter), with/without the separator
    ]f  [f      jump between functions

Value is proportional to how much you use them. If in practice you only reach
for `vaf`/`vif` and arguments, it's a thin layer over habits you already have —
worth keeping (it costs nothing) but not worth breaking other plugins for.
