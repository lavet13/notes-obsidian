---
Docs: https://www.lua.org/manual/5.4/  ·  yazi plugin API: https://yazi-rs.github.io/docs/plugins/overview
Context: learning Lua as a TS/JS dev — this maps Lua idioms to TS. Serves both yazi plugins AND nvim config.
---

## TS → Lua cheat map

```lua
local x = 1            -- `local` = block scope (like `let`). Without it, a var is GLOBAL — avoid.
-- comment            -- two dashes, not //.   --[[ block comment ]]
#t                     -- LENGTH operator (prefix). #arr = arr.length, #str = str.length
a ~= b                 -- "not equal" (Lua's !==).   == is equality (works on numbers/strings)
a .. b                 -- STRING concatenation (TS's + for strings). "x" .. 1 -> "x1"
nil                    -- null/undefined. No optional chaining → nil-check with `if x then`
a and b / a or b       -- like && / ||; also the ternary idiom:  cond and X or Y
```

- **No `++`/`+=`** → `i = i + 1`.
- **Tables are 1-indexed** — `t[1]` is the first element, and `#t` counts from 1.
- One data type for collections: the **table** `{}` (below).

## Tables = array AND object in one

`{}` is Lua's only structure. Integer keys 1,2,3… make it an array; string keys make it an object;
**it can be both at once** — that's the thing TS can't do:

```lua
{ "a", "b", confirm = true }
--  ^1   ^2   ^-- named (string key)
-- indices 1,2 = the "array part" (positional); confirm = the "hash part" (named)
```

That's exactly how `ya.emit("shell", { cmd, confirm = true })` reads: **positional** entries (no `key =`,
auto-indexed 1,2,3) are the command; **named** entries (`key = value`) are options. yazi splits them by
whether each entry has a key.

## pairs vs ipairs

```lua
for k, v in pairs(t)  do ... end   -- ALL key/value pairs, ANY key type, NO order guarantee
for i, v in ipairs(t) do ... end   -- ONLY integer keys 1,2,3…, IN ORDER, stops at first nil
```

Rule of thumb: `ipairs` for a **sequential array** where order matters; `pairs` for a general
table/dictionary. In the copy-uri plugin, `targets` is an array we built with `table.insert` (keys
1,2,3, order = selection order), so we iterate it with **`ipairs`**. We read `cx.active.selected`
(a set we don't control) with `pairs`.

## Functions: dot vs colon, multiple returns

```lua
function M.f(x)   end     -- plain function.  call: M.f(1)
function M:g(x)   end     -- METHOD: implicit `self` param. M:g(1) == M.g(M, 1)
                          -- the colon auto-passes the table as `self` (like `this`)
```

So `function M:preload(job)` = a method with a hidden `self` (the module table) as its first arg,
plus `job`. Dot = no self, colon = self.

Lua returns **multiple values** (no tuple/array needed):

```lua
function M:preload(job)
  return false, Err("...")   -- returns TWO values
end
local ok, err = M:preload(job)   -- capture both
```

## Globals: cx, ya (and the LSP error)

`cx` and `ya` are **ambient globals** the yazi runtime injects (like `window`/`process` in JS) — you
DON'T declare them or pass them as params. That's why `ya.sync(function() ... cx ... end)` works with
no `cx` argument: `cx` is just in scope.

Your Lua LSP flags "Undefined global cx/ya" because it doesn't know yazi's globals. Silence it per-file:

```lua
---@diagnostic disable: undefined-global
```

…or (better, long-term) point lua-language-server at yazi's type stubs via a `.luarc.json`
`workspace.library` entry — we can set that up when you build more plugins.

## ya.sync — the main-thread bridge

yazi runs your plugin's `entry` in a **background (async) worker** so a slow plugin can't freeze the
UI. But the app's live state (`cx`, the selection) lives on the **main thread** — the async worker
can't touch it directly (that would race). `ya.sync(fn)` wraps `fn` so calling it **hops over to the
main thread, runs `fn` there (where `cx` is valid), and hands the result back**. Think of it as: "one
worker owns the data; your code runs in another; to read the data you send a request and get an answer."
Anything reading `cx` must be inside a `ya.sync`.

## Debugging (the console.log of yazi)

The TUI owns the screen, so `print()` goes nowhere visible. Use:

```lua
ya.dbg(targets)     -- serializes a value (tables too) into yazi's log
ya.notify({ title = "x", content = tostring(#targets), timeout = 3 })  -- a visible popup
```

To read `ya.dbg` output, run yazi with logging on and tail the log:

```bash
YAZI_LOG=debug yazi
tail -f ~/.local/state/yazi/yazi.log
```

## Worked example — the copy-uri plugin

`~/.config/yazi/plugins/copy-uri.yazi/main.lua` — copy hovered/selected files as a text/uri-list:

```lua
local get_targets = ya.sync(function()        -- runs on MAIN thread (reads cx)
  local targets = {}
  if #cx.active.selected ~= 0 then            -- #… ~= 0  →  "selection is non-empty"
    for _, url in pairs(cx.active.selected) do -- iterate the selected set (order N/A → pairs)
      table.insert(targets, tostring(url))     -- push; tostring() : Url userdata → path string
    end
  else
    local h = cx.active.current.hovered        -- fallback: the hovered file (or nil)
    if h then table.insert(targets, tostring(h.url)) end   -- nil-check (no optional chaining)
  end
  return targets
end)

return {
  entry = function()                           -- yazi calls this on the keybind
    local targets = get_targets()
    if #targets == 0 then return end
    local args = {}
    for _, p in ipairs(targets) do             -- ordered array → ipairs
      table.insert(args, ya.quote(p))          -- shell-escape each path
    end
    local cmd = os.getenv("HOME")              -- process.env.HOME
      .. "/.config/yazi/copy-uri.sh " .. table.concat(args, " ")  -- .join(" ")
    ya.emit("shell", { cmd, confirm = true })  -- positional cmd + named confirm
    ya.notify({ title = "Clipboard", content = #targets .. " file(s) copied", timeout = 2 })
  end,
}
```

Keybind (`keymap.toml`): `{ on = "Y", run = "plugin copy-uri", desc = "Copy file:// URI(s)" }`.
The `.sh` helper percent-encodes each path and pipes the file:// lines to `wl-copy -t text/uri-list`.
