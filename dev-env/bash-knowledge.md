---
id: bash-knowledge
aliases:
  - bash-knowledge
tags:
  - bash
  - reference
---

# Bash Knowledge

A personal reference for writing safe, maintainable bash scripts.

---

## Shebang

```bash
#!/usr/bin/env bash   # preferred — asks env to find bash in PATH (portable)
#!/bin/bash           # hardcoded — fails if bash isn't at /bin/bash (NixOS, some macOS)
```

Use `#!/usr/bin/env bash` by default.

---

## Strict Mode

```bash
set -euo pipefail
```

| Flag | Long form         | Effect                                                  |
| ---- | ----------------- | ------------------------------------------------------- |
| `-e` | `set -o errexit`  | exit immediately if any command fails (non-zero exit)   |
| `-u` | `set -o nounset`  | treat use of an unset variable as an error              |
| -    | `set -o pipefail` | a pipe fails if ANY command in it fails (no short form) |

```bash
# Why pipefail matters:
cat file.txt | grep "x" | sort
# Without pipefail: if cat fails, the pipe's exit code is sort's (probably 0) → failure hidden
# With pipefail:    the pipe's exit code reflects the first failing command
```

`pipefail` has no single-letter shorthand — `set -o pipefail` is the only spelling.

---

## Variables

```bash
NAME="deploy"        # correct — NO spaces around =
NAME = "deploy"      # WRONG — bash thinks NAME is a command

echo "$NAME"         # read with $ — always double-quote
echo "${NAME}name"   # braces isolate the variable name from surrounding text
echo "$NAMEname"     # empty — bash looks for a variable called NAMEname
```

---

## Quoting — the most important quirk

```bash
NAME="John Doe"

echo $NAME           # works here but dangerous — splits on spaces into 2 words
echo "$NAME"         # John Doe — expands, spaces preserved
echo '$NAME'         # $NAME — single quotes = literal, NO expansion

FILE="my file.txt"
rm $FILE             # WRONG — tries to delete "my" AND "file.txt"
rm "$FILE"           # correct — one argument
```

**Rule of thumb: always double-quote variables.** Exceptions are rare.

---

## Variable Expansion Forms

The colon `:` means "also treat empty as unset". Without `:`, only truly unset triggers it.

```bash
VAR="hello"   EMPTY=""   # UNSET = never assigned

# ${VAR:-default}  → use VAR if set+non-empty, else "default"
echo "${VAR:-fallback}"     # hello
echo "${EMPTY:-fallback}"   # fallback
echo "${UNSET:-fallback}"   # fallback

# ${VAR-default}   → use VAR if set (even if empty), else "default"
echo "${EMPTY-fallback}"    # (empty)
echo "${UNSET-fallback}"    # fallback

# ${VAR:+replacement}  → "replacement" if VAR set+non-empty, else "" (opposite of `:-`)
echo "${VAR:+found}"        # found
echo "${EMPTY:+found}"      # (empty)

# ${VAR+replacement}  → "replacement" if VAR set(even if empty), else "" (opposite of `-`)
echo "${VAR+found}" # found
echo "${EMPTY+found}" # found

# ${VAR:?message}  → use VAR if set+non-empty, else print message and EXIT
echo "${VAR:?must be set}"  # hello
echo "${UNSET:?must be set}"# exits: "UNSET: must be set"
```

### The `${VAR+x}` pattern (safe checks under `set -u`)

```bash
# +x expands to "x" if VAR is set, "" if unset — never touches VAR directly,
# so it won't crash under set -u
if [[ -z "${VAR+x}" ]]; then echo "VAR is unset"; fi
if [[ -n "${VAR+x}" ]]; then echo "VAR is set (maybe empty)"; fi
if [[ -n "${VAR:+x}" ]]; then echo "VAR is set AND non-empty"; fi
```

`x` is just a conventional placeholder — any non-empty string works.

---

## `if` Statements

```bash
if CONDITION; then
  # true branch
elif OTHER; then
  # else-if branch
else
  # fallback
fi
```

The `;` before `then` is just a line separator — `if CONDITION; then` and
`if CONDITION` / newline / `then` are identical.

### `if` tests an EXIT CODE, not a value

`0` = success/true, anything else = failure/false (opposite of most languages).
Any command can follow `if` — no brackets required:

```bash
if command -v docker &>/dev/null; then echo "docker exists"; fi
if grep -q "x" file;              then echo "found"; fi
if id "deploy" &>/dev/null;       then echo "user exists"; fi
```

Here `&>/dev/null` silences the command's normal output — you only care about the
exit code (does the user exist? yes/no), not the text it would print.

Check the last exit code manually with `$?`:

```bash
ls /some/path
echo $?    # 0 if ls succeeded, non-zero if it failed
```

---

## `[ ]` vs `[[ ]]`

```bash
# [ ]  — POSIX, portable, quirky
# [[ ]] — bash-only, safer, more features. PREFER THIS.

if [ "$NAME" = "deploy" ];  then ...   # = (single) in [ ]
if [[ "$NAME" == "deploy" ]]; then ... # == works in [[ ]]
if [[ "$NAME" == dep* ]];    then ...  # glob matching — [[ ]] only

# [ ] needs variables quoted or it breaks on empty/spaces:
if [ $NAME = "deploy" ];   then ...    # dangerous
if [ "$NAME" = "deploy" ]; then ...    # safe
# [[ ]] is safe even unquoted (but quote anyway out of habit)
```

You can NOT put a plain command inside `[[ ]]`:

```bash
if [[ command -v docker ]]; then ...   # WRONG — [[ ]] wants a test expression
if command -v docker; then ...         # correct
```

---

## Common Test Conditions

```bash
# Strings
[[ -z "$VAR" ]]      # true if EMPTY (zero length)
[[ -n "$VAR" ]]      # true if NON-empty
[[ "$A" == "$B" ]]   # equal
[[ "$A" != "$B" ]]   # not equal

# Files
[ -f "/path" ]       # exists and is a regular file
[ -d "/path" ]       # exists and is a directory
[ -r "/path" ]       # readable
[ -s "/path" ]       # exists AND is non-empty
[ ! -f "/path" ]     # ! negates — does NOT exist

# Numbers — -eq -ne -lt -gt -le -ge
[[ $N -eq 0 ]]       # equal
[[ $N -ne 0 ]]       # not equal
[[ $N -gt 5 ]]       # greater than
[[ $N -lt 10 ]]      # less than
[[ $N -le 10 ]]      # less than or equal
[[ $N -ge 5 ]]       # greater than or equal
```

Personal example:

```bash
if [[ -s "$(dirname "$0")/test.txt" ]]; then
  echo "test.txt exists and non-empty"
else
  echo "test.txt not exists or empty"
fi
```

---

## `&&` and `||`

```bash
apt update && apt install docker   # install only if update succeeded
rm file.txt || true                # if rm fails, || true forces success (survives set -e)
mkdir /dir || echo "exists"
# A && B || C is NOT a clean if/else: C also runs if B fails.
# Safe only when B can't fail (like echo). Otherwise use a real `if`.
command -v docker &>/dev/null && echo "yes" || echo "no"
```

---

## Arithmetic

```bash
COUNT=$((COUNT + 1))          # arithmetic expansion — no $ needed on inner vars
(( COUNT++ ))                 # statement form, increments in place
(( COUNT > 5 )) && echo "big" # arithmetic as a condition
RESULT=$(( (A + B) * 2 ))     # + - * / % ** ( )
```

> [!NOTE]
> the bc command doesn't exist on mingw64
> explore the awk features, what else can I do with awk in context of bash

```bash
# $(( )) is integer-only. For decimals, shell out to bc or awk.

# bc — calculator; 'scale' = number of decimal places. -l loads math funcs (sqrt, s, c...)
echo "scale=2; 10 / 3" | bc        # 3.33
RESULT="$(echo "scale=4; 22 / 7" | bc)"   # 3.1428

# awk handles floats natively; printf for formatting
awk 'BEGIN { print 10 / 3 }'             # 3.3333333333
awk 'BEGIN { printf "%.2f\n", 10 / 3 }'  # 3.33

# Comparing floats (bash [[ ]] can't): bc returns 1 (true) or 0 (false)
if (( $(echo "3.5 > 2.1" | bc -l) )); then echo "bigger"; fi
```

---

## Redirection Operators

```bash
# Output
command > file       # stdout → file (overwrite)
command >> file      # stdout → file (append)
command 2> file      # stderr → file (overwrite)
command 2>> file     # stderr → file (append)
command &> file      # BOTH stdout+stderr → file (overwrite)
command &>> file     # BOTH → file (append)
command 2>&1         # send stderr to wherever stdout currently goes
command > file 2>&1  # ORDER MATTERS: stdout→file first, then stderr→same place

# Input
command < file       # file contents → stdin

# Here-document — multi-line stdin until the delimiter
cat << EOF
line one
$HOME expands here
EOF

cat << 'EOF'
$HOME printed literally (quoted delimiter = no expansion)
EOF

# Here-string — a single string → stdin
grep "word" <<< "this string has the word"
wc -w <<< "count these words"   # 3

# Pipes
cmd1 | cmd2          # stdout of cmd1 → stdin of cmd2
cmd1 |& cmd2         # stdout AND stderr of cmd1 → stdin of cmd2
```

Memorize first: `>` `>>` `2>` `&>` `|` `<<<`

### stdin / stdout / stderr (the three streams)

Every program has three default channels, numbered as file descriptors:

- **stdin (fd 0)** — the inbox. Default source: keyboard. Replaced by `< file`.
- **stdout (fd 1)** — the outbox for normal results. Default: terminal. Redirected by `>`.
- **stderr (fd 2)** — a SEPARATE outbox for errors/warnings. Default: terminal too,
  but kept distinct so you can redirect it independently. (This is why Docker
  Compose warnings still appear even when you redirect normal output.)

---

## `tee` — write to a file AND stdout at once

```bash
command | tee log.txt        # shows output AND saves it
command | tee -a log.txt     # append instead of overwrite
command | tee file > /dev/null  # save only, suppress the stdout copy
```

The killer use: writing a root-owned file via sudo, where a plain `>` redirect
would run in YOUR shell and be denied:

```bash
sudo cat << EOF > /etc/some.conf   # FAILS — redirect runs as non-root
sudo tee /etc/some.conf << EOF     # WORKS — tee itself runs as root
```

---

## Process Substitution `<(cmd)`

Turns a command's OUTPUT into a temporary FILENAME (a live pipe like `/dev/fd/63`),
so commands that expect file arguments can read command output:

```bash
diff <(echo -e "a\nb\nc") <(echo -e "a\nx\nc")   # compare two command outputs
cat <(echo hi)     # hi — cat reads the pipe
echo <(echo hi)    # /dev/fd/63 — echo just prints the filename, doesn't read it
```

### Reading `diff` output

```bash
# diff default format: [leftLines][cmd][rightLines]
#   d = delete from left   a = add from right   c = change
#   '<' marks a left-file line, '>' marks a right-file line, '---' separates a change

# delete:
diff <(printf 'a\nb\nc\n') <(printf 'a\nc\n')
# 2d1        delete line 2 (b) from left
# < b

# add:
diff <(printf 'a\nc\n') <(printf 'a\nb\nc\n')
# 1a2        after line 1 of left, add line 2 of right (b)
# > b

# change:
diff <(printf 'a\nb\nc\n') <(printf 'a\nx\nc\n')
# 2c2        line 2 of left changes to line 2 of right
# < b
# ---
# > x

# -u (unified, what git uses): ' '=context, '-'=removed, '+'=added
#   @@ -1,3 +1,3 @@  →  left: from line 1, 3 lines | right: from line 1, 3 lines

# -c (context, older): '***'=left block, '---'=right block, '!'=changed line
```

---

## Arrays and Loops

```bash
FRUITS=("apple" "banana" "cherry")    # or multi-line in parens

echo "${FRUITS[0]}"     # apple — zero-indexed
echo "${FRUITS[-1]}"    # cherry — last
echo "${FRUITS[@]}"     # all elements
echo "${#FRUITS[@]}"    # count: 3

# for over elements
for F in "${FRUITS[@]}"; do echo "$F"; done

# C-style counting loop
for ((i=0; i<3; i++)); do echo "$i: ${FRUITS[$i]}"; done

# range
for i in {1..5}; do echo "$i"; done
```

```typescript
// classic counting loop
for (let i = 1; i <= 5; i++) console.log(i);

// Array.from with a length + mapper (1..5)
Array.from({ length: 5 }, (_, i) => i + 1).forEach((i) => console.log(i));

// spread of keys() (0..4, then offset)
[...Array(5).keys()].forEach((i) => console.log(i + 1));
```

### `[@]` vs `[*]` — the distinction is about QUOTING

```bash
NAMES=("John Doe" "Jane Smith")

"${NAMES[@]}"   # → "John Doe" "Jane Smith"  (each element a separate word) ✅
"${NAMES[*]}"   # → "John Doe Jane Smith"     (all joined into ONE word by IFS)
 ${NAMES[@]}    # → John Doe Jane Smith        (UNquoted — splits into 4 words)
 ${NAMES[*]}    # → John Doe Jane Smith        (UNquoted — also 4 words)
```

UNquoted, `[@]` and `[*]` behave identically (both split). The difference appears
ONLY when quoted. **Always use `"${arr[@]}"` quoted** — it's the only form that
preserves elements containing spaces.

---

## `while` and `case`

```bash
while [[ $# -gt 0 ]]; do   # loop while arguments remain ($# = arg count)
  echo "$1"
  shift                     # $2→$1, $3→$2, decrements $#
done

case "$1" in
  --swap) echo "swap" ;;    # ;; ends a branch (like break)
  --help) echo "help" ;;
  *)      echo "unknown" ;; # * = catch-all
esac                        # "case" backwards
```

---

## Functions

```bash
# Document arguments in a comment — bash has no named parameters.
# Arguments:
#   $1 — username (string)
#   $2 — extra group to add (string)
create_user() {
  local username="$1"       # local = scoped to the function (ALWAYS use it)
  local group="$2"
  echo "creating $username in $group"
}

create_user "deploy" "docker"   # call without parentheses
```

Without `local`, variables leak into global scope and can clobber outer variables.

---

## Special Variables

```bash
$0    # script name
$1 $2 # positional arguments
$#    # argument count
$@    # all arguments (use "$@" — each stays a separate word)
$?    # exit code of last command
$$    # PID of current script
$!    # PID of last background command
```

Do NOT nest `$` inside `${ }`:

```bash
echo "${$0}"   # WRONG
echo "$0"      # correct
echo "${1:-no argument provided}"   # correct — with a default
```

# $@ and $\* both mean "all positional arguments" — the difference is QUOTING,

# exactly like array [@] vs [*]:

```bash
"$@"   # → "$1" "$2" "$3"     each argument stays a SEPARATE word  ← almost always what you want
"$*"   # → "$1 $2 $3"          all joined into ONE word (separated by first char of IFS, usually space)
 $@    # → unquoted: both split on whitespace — identical to $*
 $*    # → unquoted: same as $@
```

# Practical illustration:

```bash
show() {
  echo "count with \$@: $#"      # number of args
  for a in "$@"; do echo "@ → [$a]"; done   # each arg intact (spaces preserved)
  for a in "$*"; do echo "* → [$a]"; done   # ONE iteration, everything joined
}
show "first arg" "second"
# @ → [first arg]
# @ → [second]
# * → [first arg second]      ← joined into one string

# Rule: use "$@" to forward arguments to another command unchanged.
ssh_exec() { ssh -p "$PORT" "$USER@$HOST" "$@"; }   # passes args through faithfully
```

---

## Pattern Matching in `[[ ]]`

### Glob with `==` (pattern must NOT be quoted)

```bash
NAME="telegram-bot-1"
[[ "$NAME" == telegram* ]]   # starts with
[[ "$NAME" == *bot* ]]       # contains
[[ "$NAME" == *-1 ]]         # ends with
[[ "$NAME" == "telegram*" ]] # WRONG — quotes make * literal
```

### Regex with `=~` (ERE; pattern must NOT be quoted)

```bash
[[ "$NAME" =~ ^telegram ]]   # starts with
[[ "$NAME" =~ [0-9]+$ ]]     # ends with digits

# Capture groups land in BASH_REMATCH
[[ "$NAME" =~ ([a-z]+)-([a-z]+)-([0-9]+) ]]
echo "${BASH_REMATCH[0]}"    # telegram-bot-1 (whole match)
echo "${BASH_REMATCH[1]}"    # telegram (group 1)
```

> [!NOTE]
> Explore once again PCRE features.

```bash
# ERE essentials (used by [[ =~ ]] and `grep -E`, `sed -E`):
# ^       anchor: start of string
# $       anchor: end of string
# .       any single character
# *       zero or more of the preceding element
# +       one or more of the preceding element
# ?       zero or one of the preceding element
# {n,m}   interval: between n and m of the preceding element  (e.g. [0-9]{1,3})
# [abc]   one char from the set
# [^abc]  one char NOT in the set
# [0-9]   one char in the range
# [a-z]   one char in the range
# (a|b)   group + alternation (a OR b)
# |       alternation (OR)

# NO non-capturing groups in bash. POSIX ERE has only capturing ( ).
# (?:...) , \d , \w , lookahead/lookbehind, backreferences = PCRE features, NOT available here.
# Every ( ) populates BASH_REMATCH. Use [0-9] not \d.
```

### BRE vs ERE — why `sed` needs backslashes

`sed` uses **Basic** Regular Expressions by default, where `? + ( )` are LITERAL
and must be escaped to get special meaning:

```bash
# \? = zero or one of the preceding element (here: an optional '#')
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
#         ^#\?  matches both "PermitRootLogin yes" AND "#PermitRootLogin yes"
```

```bash
# sed uses BRE by default. Enable ERE with -E (or GNU -r):
sed -E 's/(foo|bar)+/X/' file     # ERE: + ( ) | are special as-is
sed    's/\(foo\|bar\)\+/X/' file # BRE: same meaning, everything escaped

# BRE essentials (the FULL list):
# ^ $ . [abc] [^abc] [0-9] [a-z]   → same as ERE (no escaping needed)
# *                                 → zero or more (same as ERE)
# \?                                → zero or one     (escaped in BRE!)
# \+                                → one or more     (escaped in BRE!)
# \{n,m\}                           → interval        (escaped in BRE!)
# \( \)                             → grouping        (escaped in BRE!)
# \|                                → alternation     (escaped in BRE, GNU)
# \1 \2                             → backreferences to groups
#
# Rule of thumb: in BRE, the "powerful" metachars need a backslash to wake up.
# In ERE they're special by default and a backslash makes them literal. Opposite worlds.
```

In ERE (`grep -E`, `[[ =~ ]]`) you'd write plain `#?`. Same meaning, different escaping.

---

## String Manipulation

```bash
NAME="telegram-bot"
echo "${#NAME}"        # 12 — length
echo "${NAME:0:8}"     # telegram — substring (start 0, length 8)
echo "${NAME:9}"       # bot — from index 9 to end
echo "${NAME^^}"       # TELEGRAM-BOT — uppercase
echo "${NAME,,}"       # telegram-bot — lowercase
echo "${NAME/bot/BOT}" # telegram-BOT — replace first
echo "${NAME//o/0}"    # telegram-b0t — replace all

# printf — aligned columns, reliable \n \t
printf "%-20s %s\n" "Container" "Status"
```

---

## `command` builtin

```bash
command -v docker      # prints path if found, exits non-zero if not (like `which`, better)
command ls             # run real ls, bypassing any function/alias named ls
type command           # "command is a shell builtin" — implemented inside bash, not on disk
```

Builtins (`echo cd set export local shift read command`) live in bash's own code;
external commands (`ls grep sed`) are files found via `$PATH`.

```bash
# export — mark a variable so CHILD processes inherit it (makes it an env var)
NAME="deploy"                 # shell variable — visible only in THIS shell/script
export NAME                   # now commands this script runs will see it
export PATH="$PATH:/new/dir"  # common: extend PATH for child commands
export DEBUG=1 node app.js    # inline: set for just this one command's environment

GREETING="hi"
bash -c 'echo "$GREETING"'    # (empty) — child didn't inherit
export GREETING
bash -c 'echo "$GREETING"'    # hi — now it does

# set — two unrelated jobs:
# 1) toggle shell options
set -euo pipefail
# 2) replace the positional parameters ($1, $2, ...); -- ends option parsing
set -- apple banana cherry    # $1=apple $2=banana $3=cherry
echo "$1 / $#"                # apple / 3
set                           # with NO args: prints all shell variables
```

---

## `.` / `source`

```bash
. /etc/os-release        # identical to: source /etc/os-release
```

Runs the file IN THE CURRENT SHELL (not a child process), so variables it defines
become available afterward. `/etc/os-release` is plain `KEY=value` lines:

```bash
DISTRO_ID="$(. /etc/os-release && echo "$ID")"   # → "debian" / "ubuntu"
```

---

## Sourcing a config file (with a guard)

```bash
ENV_FILE="$(dirname "$0")/health-check.env"   # find it next to the script
if [[ ! -f "$ENV_FILE" ]]; then
  echo "❌ Missing $ENV_FILE — copy the .example and fill it in"
  exit 1
fi
source "$ENV_FILE"   # NOTE: this EXECUTES the file — only source files you trust
```

---

## `trap` — run code on exit / error (the real home of `$LINENO`)

```bash
# Cleanup on ANY exit
trap 'rm -f /tmp/tempfile' EXIT

# Report WHERE a failure happened
# $LINENO = current line, $BASH_COMMAND = the command that ran, $? = its exit code
trap 'echo "✗ line $LINENO: [$BASH_COMMAND] exited $?" >&2' ERR
```

`>&2`
It redirects that command's stdout to stderr (file descriptor 2). Breaking it
down using what's already in your notes:

```bash
echo "oops" >&2  # send this line to stderr instead of stdout
#            │└─ fd 2 (stderr)
#            └─ & means "the file descriptor numbered...", not a literal file named "2"
```

Without the `&`, `>2` would create a file literally named `2`. With `&`, bash
reads `2` as "file descriptor 2" = stderr. So `>&2` = "send my normal output to
the error stream."

Why do it in the trap? Error/diagnostic messages belong on stderr, not stdout —
so they show up even when someone pipes the script's normal output to a file,
and they don't pollute that captured output. It's the same stdout/stderr
separation from your notes, applied deliberately.

`$FUNCNAME` is an array of the call stack — niche, useful only inside logging
helpers: `FUNCNAME[0]` is the current function, `FUNCNAME[1]` is its caller.

```bash
# $FUNCNAME — call-stack array, only meaningful inside a function.
#   [0] = current function, [1] = its caller, [2] = caller's caller ...
outer() { inner; }
inner() {
  echo "running: ${FUNCNAME[0]}"    # inner
  echo "called by: ${FUNCNAME[1]}"  # outer
}
outer

# Practical use: a log helper that auto-tags WHO logged
log() { echo "[${FUNCNAME[1]}] $*"; }   # [1] = caller of log()
deploy() { log "starting"; }            # → [deploy] starting
deploy
```

---

## `read` — interactive input

```bash
read -p "Username: " USERNAME
read -s -p "Password: " PASSWORD   # -s hides typing
```

```bash
read -p "Prompt: " VAR     # -p TEXT  : print prompt (no newline) before reading
read -s -p "Pass: " PW     # -s       : silent — don't echo typed chars (passwords)
read -r LINE               # -r       : raw — backslash is literal, not an escape (ALWAYS use)
read -n 1 KEY              # -n N     : read at most N chars, return immediately (no Enter)
read -t 5 VAR              # -t N     : time out after N seconds
read -a ARR                # -a NAME  : split input words into array NAME
read A B C                 # multiple vars: split one line on whitespace into each

# Safe everyday pattern — combine -r and -p:
read -rp "Continue? [y/N] " answer
```

---

## Argument Parsing Template

```bash
SWAP_ENABLED=false
SWAP_SIZE="1G"

while [[ $# -gt 0 ]]; do
  case "$1" in
    --swap)
      SWAP_ENABLED=true
      # optional value: take $2 only if it exists and isn't another flag
      if [[ -n "${2-}" && "${2}" != --* ]]; then
        SWAP_SIZE="$2"
        shift
      fi
      shift
      ;;
    *)
      echo "Unknown argument: $1"
      exit 1
      ;;
  esac
done
```

---

## Delivering a script from a public repo

```bash
curl -fsSL https://raw.githubusercontent.com/USER/REPO/main/scripts/setup.sh \
  | bash -s -- --swap 1G
```

- `curl -f` fail on HTTP error `-s` silent `-S` but still show real errors `-L` follow redirects
- `bash -s` read script from stdin (the pipe)
- `--` end bash's own options; everything after goes to the SCRIPT as `$1 $2 ...`

```bash
# GUARD:  [ test ] || action   →  "do it unless it's already done" (idempotency)
[ -d ~/.config ]            || mkdir ~/.config          # make dir if missing
[ -f .env ]                 || cp .env.example .env      # seed file if absent
command -v jq >/dev/null    || sudo apt-get install -y jq  # install if not present

# SWALLOW:  command || true  →  "let this one fail without killing the script"
grep -q PATTERN file || true   # grep exits 1 on no-match; don't let set -e abort
pipx install ruff    || true   # already-installed exits non-zero

# The && twin: chain only-on-success
mkdir build && cd build        # cd only if mkdir worked
```

## Glob vs Regex (find vs fd)

`find -name` takes a GLOB; `fd` takes a REGEX. The same pattern means different things:

```bash
find /tmp -name 'tmp.*'   # GLOB:  t m p . <anything>      — the dot is LITERAL
fdfind 'tmp.*' /tmp       # REGEX: t m p <any-char>*       — dot = any char, * = zero+ of it
```

|           | glob (`find -name`, shell `*`) | regex (`fd`, `grep -E`, `[[ =~ ]]`) |
| --------- | ------------------------------ | ----------------------------------- |
| `*`       | any run of chars               | zero-or-more of the preceding atom  |
| `.`       | literal dot                    | any single char                     |
| `?`       | any single char                | zero-or-one of preceding            |
| anchored? | whole-name match               | substring unless `^ $`              |

fd is also recursive-by-default, regex-by-default, and skips hidden/gitignored; find needs an
explicit start path (defaults to cwd) and matches everything.

### Anchoring — the `anchored?` row, with examples

**Anchored** = pattern must match the WHOLE string (nothing before/after).
**Unanchored** = matches if it appears ANYWHERE (a substring is enough).
Globs are anchored by default; regex is unanchored by default (`^`/`$` are literally "anchors").

```bash
touch log log.txt mylog
# GLOB is anchored: -name compares against the ENTIRE basename
find . -name 'log'    # ./log ONLY — "log.txt"/"mylog" fail (chars hang outside the match)
find . -name 'log*'   # log, log.txt      — the * lets the match EXTEND rightward
find . -name '*log*'  # log, log.txt, mylog — you manually UN-anchored both ends

# REGEX is unanchored: matches if it appears anywhere
printf 'log\nlog.txt\nmylog\n' | grep -E 'log'    # all three (substring is enough)
printf 'log\nlog.txt\nmylog\n' | grep -E '^log$'  # log only — ^ pins start, $ pins end
```

> Mental model: glob = anchored by default, WIDEN with `*`. regex = unanchored by default,
> PIN with `^ … $`. Same goal, opposite starting points.

### `sed` delimiter swap

```bash
sed 's|^\./||'   # `s` takes ANY delimiter after it. Pattern has a slash (./), so use | instead
                 # of / to avoid escaping: s/^\.\/// is the ugly equivalent. ^ = line start,
                 # \. = literal dot, / = literal (no escape — | is the delimiter). Strips "./".
```

---

```bash
## `tar` create side — `-f` is the OUTPUT path, `-C DIR .` is the INPUT
# (complements the read-side idiom above: `curl … | tar xz -C dir`)
# Flag letters below:  c=create   t=list (table-of-contents; reads, never extracts)
#                      x=extract   z=gzip/gunzip   f=archive file   C=chdir into DIR first
# On CREATE, one command does two INDEPENDENT jobs:
tar czf /backup/vol.tar.gz -C /v .
#       └──────┬─────────┘  └─┬┘ │
#              │              │  └ operand = WHAT to pack, resolved AFTER -C → "/v's contents"
#              │              └ -C: chdir into /v before reading operands
#              └ -f = WHERE the archive is written. ABSOLUTE → unaffected by -C or pwd.
#
# The `.` is the INPUT, not the output. The destination is fixed by -f. It's easy to misread
# `.` as "create it in the current dir" — it isn't; it's "pack the current dir".
#
# Why `-C /v .` instead of `tar czf out.tgz /v`? It changes the MEMBER PATHS recorded inside
# (`tar tzf X` = list X's members, no extraction — used here just to show the difference):
tar czf a.tgz /v      ; tar tzf a.tgz   # → v/  v/data/…   (leading v/ baked in; GNU also warns
                                        #   "Removing leading '/'" for absolute operands)
tar czf b.tgz -C /v . ; tar tzf b.tgz   # → ./  ./data/…   (relative to /v)
# Restore symmetry: `tar xzf b.tgz -C /v` drops ./data/… straight into /v (correct).
#   The a.tgz form would restore to /v/v/data/… — wrong.
```

## rm: -f vs -r

`rm -f` = force (no prompt, ignore missing) but does NOT recurse into directories.
`rm -r` = recurse into directories. For find-and-delete of dirs you need -r:

```bash
find /tmp -iname 'tmp.*' -exec rm -r {} +   # -f alone errors: "Is a directory"
```

---

## cp: overwrite control + merging directories

```bash
cp src dst            # overwrites dst by default
cp --update=none ...  # skip existing, no failure (modern GNU form)
cp -n ...             # DEPRECATED on GNU; Debian prints a non-portability warning. Avoid.

# Merge a dir's CONTENTS into another (no nesting), including dotfiles:
cp -r src/. dst/      # the /. = "contents of src, hidden files included"
cp -rT src dst        # -T / --no-target-directory: same effect, self-documenting
# (cp -r src dst when dst exists → nests as dst/src; src/* misses dotfiles)
```

---

## umask — default-permission mask (bits to REMOVE)

Shell builtin naming the permission bits to CLEAR from the base when a file/dir is created.
Octal, per-process, inherited by children, usually set in shell rc.

```bash
# result = base with the umask's bits removed:  result = base & ~umask
# Base: files start 666 (rw for all), dirs start 777. Files get NO execute bit by default —
# that's why source files never come out executable.
umask                # print current mask, octal (e.g. 022)
umask -S             # print it symbolically (e.g. u=rwx,g=rx,o=rx)

# Common masks and what new files/dirs land as:
#   umask 022 (typical):  file 644 rw-r--r-- ,  dir 755 rwxr-xr-x
#   umask 077 (private):  file 600 rw------- ,  dir 700 rwx------
#   umask 002 (group-wr): file 664 rw-rw-r-- ,  dir 775 rwxrwxr-x

# Worked example, umask 077 on a new FILE:
#   base   666   rw- rw- rw-
#   umask  077   --- rwx rwx    owner digit 0 = clear nothing;  grp/oth 7 = clear rwx
#   result 600   rw- --- ---
umask 077; touch f; ls -l f     # -rw-------  owner-only from birth, no chmod needed

# tldr decode: "restrict" = "mask out / take away". "restrict no permissions for the owner"
#   = owner digit 0 = keep defaults; "restrict all for everyone else" = grp/oth digit 7 =
#   strip rwx. The double-negative phrasing is what reads backwards.
# GOTCHA: umask only ever REMOVES bits — it can't ADD above the 666/777 base. chmod +x still
#   needed for an executable. (It's the default-setter behind the manual `chmod 600/700`
#   used for SSH keys below.)
```

---

## `tar` — `-f` = archive path, `-C DIR .` = what to pack

```bash
# ── CREATE: two INDEPENDENT jobs in one command ──────────────────────────────
tar czf /backup/vol.tar.gz -C /v .
#       └──────┬─────────┘  └─┬┘ │
#              │              │  └ operand = WHAT to pack, resolved AFTER -C → "/v's contents"
#              │              └ -C: chdir into /v before reading operands
#              └ -f = WHERE the archive is written. ABSOLUTE → unaffected by -C or pwd.
#
# The `.` is the INPUT, not the output. Destination is fixed by -f. Easy to misread `.`
# as "make it in the current dir" — it isn't; it's "pack the current dir".
#
# Why `-C /v .` and not `tar czf out.tgz /v`? It changes the MEMBER PATHS recorded inside:
tar czf a.tgz /v      ; tar tzf a.tgz   # → v/  v/data/…  (leading v/ baked in; GNU also warns
                                        #   "Removing leading '/'" for absolute operands)
tar czf b.tgz -C /v . ; tar tzf b.tgz   # → ./  ./data/…  (relative to /v)
# Restore symmetry: `tar xzf b.tgz -C /v` drops ./data/… straight into /v (correct).
#   The a.tgz form would restore to /v/v/data/… — wrong.
#
# ── READ (extract): -f says WHERE to read the archive FROM ────────────────────
tar xzf archive.tgz -C dir   # -f FILE → read that file;  -C dir → cd into dir first
curl … | tar xz -C dir       # NO -f → read the archive from STDIN (the pipe)
tar xz -f - -C dir           # `-f -` = the explicit stdin form (identical to no -f)
```

---

## mktemp -d — safe scratch space

```bash
tmp="$(mktemp -d)"    # atomically creates a uniquely-named 0700 dir, prints its path
# ... use "$tmp" ...   (vs `mkdir /tmp/foo`: fixed name, collisions, symlink-attack risk)
```

Pair with `trap 'rm -rf "$tmp"' EXIT` so it's removed however the script ends.

---

## trap — signals & why EXIT for cleanup

```bash
trap cleanup EXIT     # fires on ANY exit: normal, `exit`, error (set -e), post-signal. Cleanup hook.
trap 'handler' ERR    # on any command failing (non-zero). Diagnostics: $LINENO, $BASH_COMMAND.
trap 'handler' INT    # Ctrl-C (SIGINT)
trap 'handler' TERM   # kill (SIGTERM) — graceful shutdown
trap -  INT           # remove a trap (restore default)
trap '' INT           # ignore the signal (empty handler)
```

EXIT is the catch-all — it fires no matter HOW the script leaves, so one `trap cleanup EXIT`
covers every exit path. Use ERR/INT/TERM to REACT to a specific event; EXIT to ALWAYS-do-on-exit.

---

## find: grouping predicates

`find` ANDs predicates by default. To OR a group and apply -exec to the whole group:

```bash
find . -type f \( -name '*.pub' -o -name 'known_hosts*' \) -exec chmod 644 {} +
#              \(  ...  -o  ...  \)         -o = OR; \( \) escaped (shell metachars)
#  -exec ... {} +   batches files into few calls   (vs \;  = one call per file)
```

---

## SSH: trust a host non-interactively

```bash
# Avoid the "authenticity of host" prompt hanging a script:
ssh-keygen -F github.com >/dev/null 2>&1 || ssh-keyscan github.com >> ~/.ssh/known_hosts
# ssh-keygen -F = "is this host already in known_hosts?" (exit 0 if yes)
```

---

## shellcheck: SC2005 (useless echo)

```bash
echo "$(cmd)"   # SC2005 — captures output only to re-print it
cmd             # identical, cleaner
```

## ln — symbolic links

```bash
ln -s TARGET LINKNAME   # create LINKNAME as a symlink pointing at TARGET
ln -sf TARGET LINKNAME  # -f: replace an existing LINKNAME first (idempotent re-runs)
#   -s = symbolic (a path pointer; can cross filesystems, point at dirs)
#   without -s = HARD link (same inode; same filesystem only)
```

Use an ABSOLUTE target so the link resolves from anywhere:

```bash
ln -sf "$HOME/repo/dotfiles/.zshrc" "$HOME/.zshrc"   # ~/.zshrc IS the repo file now
```

Inspect:

```bash
ls -l ~/.zshrc        # shows  .zshrc -> /home/me/repo/dotfiles/.zshrc
readlink -f ~/.zshrc  # resolves the final real path
```

### Gotchas

- **Atomic-save breaks links.** Tools that write a temp file then rename over the
  target REPLACE the symlink with a regular file. Neovim edits in place (safe);
  `p10k configure` and similar may break the link — re-run `ln -sf` if so.
- **`sed -i` breaks links too** — it rewrites by replace, severing the symlink
  (or editing the real target). Don't `sed -i` a symlinked file you want to stay linked.
- **`chmod`/`cp` follow the link to the TARGET.** `chmod +x` on a symlink changes
  the repo file's mode. For a symlinked SCRIPT, commit the executable bit at the
  source instead: `git update-index --chmod=+x path/to/script` (a fresh clone of a
  100644 file is NOT executable, even if your local copy is).
- **`ln -sf TARGET DIR/`** where DIR is a symlink-to-directory can create the link
  INSIDE the dir — fine for file targets, surprising for dirs.
- **Dangling links** point at something gone (moved/deleted repo, cleaned temp dir).
  `ls -l` shows them; `find . -xtype l` lists broken ones.

### Re-linking a symlink that points at a directory

```bash
ln -sfn TARGET LINKNAME   # -n/--no-dereference: replace an existing dir-symlink
```

Plain `ln -sf TARGET LINK` when LINK already exists AND resolves to a directory
FOLLOWS the link and creates the new one INSIDE it (e.g. re-running a setup
script makes `~/.wezterm/.wezterm -> itself`). `-n` treats the existing symlink
as a plain file and replaces it. Only needed for directory targets; `-sf` is
fine when the target is a regular file.

## find: symlinks (-type l vs -xtype l)

```bash
find . -type l        # symlinks themselves (working OR broken)
find . -xtype l       # BROKEN/dangling symlinks only — target missing or unresolvable
#   -type  checks the LINK's own type
#   -xtype checks the TARGET's type; a healthy link resolves to f/d/…, so it's
#          never 'l' — only a broken link reports as 'l' (target can't be stat'd)
find . -xtype f       # symlinks that point at a regular file (resolve the target)
readlink -f LINK      # print the final resolved path a link points to (empty/err if broken)
```

Use `find ~ -xtype l` after moving or deleting a repo to spot dotfile symlinks
that now dangle.

## TTY vs PTY

- **tty** ("teletypewriter") — the kernel device for a terminal endpoint: carries I/O,
  line editing, signals (Ctrl-C → SIGINT), and the session's controlling terminal.
- **pty** ("pseudo-terminal") — a software tty handed out when there's no real hardware
  terminal. Two ends: MASTER (held by the emulator/tmux/ssh, drives the screen) and
  SLAVE (`/dev/pts/N`, what your shell sees as its tty — indistinguishable from real).
- Stacking: WezTerm opens a pty → runs zsh on the slave; tmux opens its OWN pty per pane.
  Each layer presents a tty to what runs above it.

```bash
tty            # print this shell's terminal device, e.g. /dev/pts/3
[[ -t 1 ]]     # test: is fd 1 (stdout) a tty? false for pipes, non-interactive, no-pty
```

Gotcha: a non-interactive `zsh -c` or a popup without a pty has NO tty — anything that
needs a terminal (or `[[ -t 1 ]]`) will skip/fail there.

## `--` end-of-options (with wsl.exe)

`--` tells a command "my options end here; treat the rest literally." Same marker as
`bash -s --` and `set -- …` in these notes.

```bash
wsl.exe -d Debian --cd ~ -- env NO_TMUX=1 zsh -li
# └── wsl.exe's flags ──┘  └── command run INSIDE the distro ──┘
#   without --, wsl.exe tries to parse `env …` (esp. a leading -) as its own options
```

`env VAR=val cmd` sets VAR in the environment, then execs cmd with it — here, launch an
interactive-login zsh (`-li`) with NO_TMUX set so .zshrc's auto-start guard skips.

## Signals (the ones worth knowing)

A signal is an async notification the kernel/another process sends to a process.
List them all with `kill -l`. Prefer NAMES over numbers (numbers vary by arch for
a few; the common ones below are stable on Linux x86-64).

| Name    | No. | Default action     | Typically from                          | Trappable? |
| ------- | --- | ------------------ | --------------------------------------- | ---------- |
| SIGHUP  | 1   | terminate          | terminal/session closed; also "reload"  | yes        |
| SIGINT  | 2   | terminate          | Ctrl-C                                  | yes        |
| SIGQUIT | 3   | terminate + core   | Ctrl-\                                  | yes        |
| SIGABRT | 6   | terminate + core   | abort() / assertion failure             | yes        |
| SIGKILL | 9   | terminate (forced) | `kill -9` — last resort                 | **NO**     |
| SIGSEGV | 11  | terminate + core   | invalid memory access                   | yes        |
| SIGPIPE | 13  | terminate          | write to a pipe with no reader          | yes        |
| SIGTERM | 15  | terminate (polite) | `kill` default; graceful shutdown       | yes        |
| SIGTSTP | 20  | stop (suspend)     | Ctrl-Z                                  | yes        |
| SIGSTOP | 19  | stop (suspend)     | `kill -STOP` — pause a process          | **NO**     |
| SIGCONT | 18  | continue           | `fg`/`bg`, `kill -CONT` — resume        | yes        |
| SIGUSR1 | 10  | terminate          | user-defined (whatever you trap it for) | yes        |
| SIGUSR2 | 12  | terminate          | user-defined                            | yes        |
| SIGCHLD | 17  | ignored            | a child stopped/exited                  | yes        |

```bash
kill -l                 # list all signal names
kill -TERM 1234         # polite: ask PID 1234 to shut down (== kill -15, == default kill)
kill -9 1234            # forced: kernel kills it; no cleanup, can't be caught
kill -STOP 1234         # pause;  kill -CONT 1234 resumes it
```

### Gotchas

- **SIGKILL (9) and SIGSTOP (19) cannot be caught, blocked, or ignored** — the kernel
  handles them directly. That's why `kill -9` always works and why you can't trap it
  for cleanup. Reach for it only after a polite `SIGTERM` fails.
- **SIGTERM is the default** for `kill` — always try it first; it lets the program run
  its cleanup (its EXIT/TERM trap). SIGKILL skips all of that.
- **Ctrl-C = SIGINT, Ctrl-Z = SIGTSTP, Ctrl-\ = SIGQUIT** — keyboard signals go to the
  foreground process group of the terminal.
- **SIGHUP** historically meant "the terminal hung up"; daemons (nginx, etc.) repurpose
  it to mean "reload config without restarting." `nohup` and `disown` shield a process
  from it so it survives the terminal closing.
- Ties to `trap` (see this file): `trap 'handler' INT TERM` catches the trappable ones;
  EXIT is not a real signal but a bash pseudo-signal that fires on any exit.

## SSH with GitHub: auth test + "permission denied (publickey)"

```bash
ssh -T git@github.com   # -T = no PTY (don't allocate a terminal). GitHub never gives a
                        # shell, so -T avoids "PTY allocation request failed". Success:
                        # "Hi <user>! You've successfully authenticated, but GitHub does
                        # not provide shell access." — that greeting IS the success.
```

`-T` is the opposite of `-t` (force-allocate a PTY). For a service that only does git
over ssh, you want -T — there's no interactive terminal to ask for.

### Fixing "Permission denied (publickey)"

Means no offered key was accepted. Work down this list:

```bash
ssh -vT git@github.com          # -v verbose: shows which keys are offered & why refused
ls -l ~/.ssh                    # keys present? (id_ed25519 + id_ed25519.pub)
```

Common causes & fixes:

1. **Wrong permissions** — ssh ignores keys that are too open.

```bash
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/id_ed25519        # private key: owner-only
   chmod 644 ~/.ssh/id_ed25519.pub    # public key
```

2. **Key not registered on GitHub** — copy the PUBLIC key and add it at
   github.com → Settings → SSH and GPG keys:

```bash
   cat ~/.ssh/id_ed25519.pub          # paste this (the .pub, never the private one)
```

3. **Agent doesn't have the key loaded** (or a non-default filename):

```bash
   eval "$(ssh-agent -s)" && ssh-add ~/.ssh/id_ed25519
```

4. **Remote is HTTPS, not SSH** — `git push` then asks for a password, which GitHub
   removed in 2021. Switch the remote to SSH:

```bash
   git remote -v                                         # check current URL
   git remote set-url origin git@github.com:USER/REPO.git
```

### Gotcha (WSL specifically)

Keys copied from Windows (`/mnt/c/Users/<you>/.ssh`) arrive without Unix perms, so ssh
refuses them until you re-`chmod` as above — which wsl-setup.sh does in step 5.
