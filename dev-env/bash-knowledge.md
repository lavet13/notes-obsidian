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

---

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
