---
id: wl-clipboard-knowledge
aliases:
  - wl-clipboard-knowledge
tags:
  - wl-clipboard
  - wayland
  - clipboard
  - reference
---

# wl-clipboard Knowledge

Wayland clipboard from the CLI: `wl-copy` writes, `wl-paste` reads.

## Mental model — the clipboard is SERVED by a live process

Unlike X11, Wayland has no central clipboard store: the source app hands out the
data on each paste. `wl-copy` forks a small background process to hold it. If that
process dies and nothing else took the data (a clipboard manager like `cliphist`,
or the app you pasted into), the clipboard goes empty. This is by design, and it's
why a clipboard manager exists.

## wl-copy / wl-paste — the flags that matter

```bash
wl-copy "hello"            # copy a literal string (NO trailing newline added)
some-cmd | wl-copy         # copy stdin verbatim
some-cmd | wl-copy -n      # -n / --trim-newline: strip ONE trailing newline from stdin
wl-copy -t text/uri-list … # -t / --type: advertise a MIME type (default: plain text)
wl-copy -p "x"             # -p / --primary: middle-click PRIMARY selection, not the clipboard
wl-copy -o "x"             # -o / --paste-once: clear the clipboard after the first paste
wl-copy -c                 # -c / --clear: empty the clipboard
wl-paste                   # read it back
wl-paste -l                # -l / --list-types: which MIME types are on offer right now
wl-paste -t text/uri-list  # read one specific type
```

## Copying a FILE, not text — text/uri-list + file://

To paste a file as an attachment (Telegram Desktop, Discord) rather than its path
as text: advertise `text/uri-list` and give a `file://` URI.

```bash
# one file — realpath absolutizes; file:// makes it a URI
wl-copy -t text/uri-list "file://$(realpath song.mp3)"
```

Gotcha — encoding: `file://` URIs are supposed to be percent-encoded (space -> %20).
The `q` conversion in yt-dlp (and realpath here) does NOT encode, so a literal space
lands in the URI. Qt/Electron apps (Telegram Desktop, Discord) accept the literal
path fine — verified. A strict RFC parser would want %20; only reach for encoding if
a pickier target ever rejects the paste.

## clipfiles() — copy MULTIPLE files at once

Three ideas: MIME type `text/uri-list`, one `file://` URI per line, piped to wl-copy.

```zsh
# clipfiles — copy one or more files to the clipboard AS FILES.
#   clipfiles song.mp3          (one)      clipfiles ~music/*.mp3   (glob → arg list)
clipfiles() {
  (( $# )) || { print -u2 "clipfiles: no files given"; return 1 }  # $# = arg count; -u2 = stderr
  local -a uris                   # array of file:// URIs
  local f
  for f in "$@"; do               # "$@" keeps each arg its own word (spaces survive)
    uris+=( "file://${f:A}" )     # ${f:A} = absolute path, symlinks resolved (:a = don't resolve)
  done
  printf '%s\r\n' "${uris[@]}" | wl-copy -t text/uri-list
  #      └ printf reuses the format per element → one URI per line; \r\n = CRLF (uri-list terminator)
}
```
