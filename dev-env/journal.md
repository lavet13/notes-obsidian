---
id: journal
aliases:
  - journal
tags: []
---

# Journal

## 2026-07-31 — Docker disk usage, prune semantics, volume backup

**What we covered.** Docker disk accounting end to end: the four buckets +
layer sharing, read-only inspectors, prune semantics and gotchas, dangling vs
unused images, tag/untag, volume backup/restore, DB dumps, and the `tar` -f/-C
mechanics under the backup one-liner.

**Key insight.** "Is docker lying about size?" — no: Docker reports LOGICAL
sizes, `du` reports PHYSICAL blocks; the gap was btrfs zstd compression
(confirmed, see findings). My first guess — shared-layer inflation — was
DISPROVEN by SHARED SIZE=0 in `df -v`; the inflation effect is real in general
but didn't apply to this image set. Other load-bearing facts:
- `image prune -a` is gated by ANY container reference (running OR stopped), not
  by tag. All four images showed `In Use` → prune -a reclaims 0B.
- "Unused volume" = zero container refs. The `--rm` backup helper never orphans
  the volume — the real owner container holds it throughout (LINKS 1->2->1).
- `docker system prune --volumes` sweeps only ANONYMOUS volumes; named unused
  ones need `docker volume prune -a`. Real danger = `compose down` (keeps named
  volumes) then a broad `volume prune -a`.
- `tar` create: `-f` = OUTPUT path (absolute → independent of -C/pwd); `-C DIR .`
  = INPUT, records members relative to DIR so restore lands correctly.

**What we built/changed.**
- NEW dev-env/docker-knowledge.md (ref drafted, NOT yet committed): inspect cmds,
  df-vs-du, dangling/unused, prune defaults, untag/retag, volume backup/restore,
  ## Database backups (pg_dumpall/mysqldump), builder deprecation.
- bash-knowledge.md: new `## tar` section (‑f create-output/extract-input incl.
  `-f -` stdin, `-C DIR .`, member paths); de-staled `### sed delimiter swap`
  (dropped "+ tar stdin / two idioms" — only the sed idiom remains there).
- Anki: docker-knowledge-cards.tsv, 14 cards (dev-env). Card 14 (tar `-C .`)
  re-tagged `dev-env bash tar` since its mechanics moved to bash-knowledge.

**Findings about this machine.**
- /var/lib/docker is on btrfs subvol /@ with `compress=zstd:1` → `du` (~1.2G,
  physical) < Docker logical (~1.7G), ~1.4x ratio.
- Docker uses the containerd image store: `docker images` = DISK USAGE (unpacked)
  vs CONTENT SIZE (compressed blobs) + In Use; `docker image ls --tree` lists
  per-platform variants (0B = platform not pulled).
- The two DB volumes belong to TWO separate compose projects sharing one daemon
  (php-apache-mysql-containerized_data → mysql; donbass-post_donbass-post-data →
  postgres). Daemon view is global, not per-directory.
- Legacy (non-BuildKit) builder active → `pacman -S docker-buildx` to switch.

**Next step to resume.** Paste the docker-knowledge.md + bash-knowledge.md blocks,
import docker-knowledge-cards.tsv, commit notes-obsidian. Optional: `sudo compsize
/var/lib/docker` for the exact ratio; `sudo pacman -S docker-buildx` clears the
deprecation.

**Commit (notes-obsidian).**
docs(dev-env): docker disk/prune/volume notes + tar -f/-C in bash-knowledge

## 2026-07-28 — Arch/pacman deep-dive + Linux-learning track kickoff

**What we covered.**
- Decomposed pacman's combined flags letter-by-letter (-Syu, -Rns, -Rdd,
  --needed, makepkg -si) and the -Qsub verbs. Key insight: every letter is one
  independent switch; the partial-upgrade footgun (`-Sy` alone) follows directly
  from "no stable snapshot — pkgs built against each other's current versions."
- Six maintenance gotchas not previously in notes: orphans, cache/paccache,
  db.lck, .pacnew, keyring refresh, query verbs. Each with a reverse.
- Login manager: I wrongly guessed SDDM; `systemctl status display-manager`
  showed **plasmalogin** (Plasma Login Manager) — the SDDM successor shipping
  with Plasma 6.6. Session mechanism unchanged (/usr/share/wayland-sessions/),
  so the Sway-as-second-session plan is intact.

**Findings about this machine.**
- DM is plasmalogin.service, not sddm. Sway would register as a Wayland session
  and appear in its picker; KDE stays default.
- amneziavpn-bin is pinned via IgnorePkg (4.8.21.0), so -Syu skips it with
  [ignored]. Reason for the pin unconfirmed — ASK next session before updating.

**What we built/changed (notes repo).**
- arch-knowledge.md: appended flag-anatomy, partial-upgrade footgun, IgnorePkg
  override, routine-maintenance, query-verbs, and login-manager sections.
- anki/arch-pacman-2026-07-28.tsv: 15 cards (dev-env; pacman/flags/aur/
  maintenance/kde sub-tags).
- Decided: general-OS learning notes → NEW dev-env/linux-knowledge.md (not yet
  created); arch-knowledge stays pacman-specific, bash-knowledge stays scripting.

**Resume pointer / next step.**
1. START the Linux-learning track, GRADUAL mode: Shotts (TLCL) foundational
   chapter first (filesystem + navigation) to give the puzzles ground, THEN
   OverTheWire Bandit level 0. Ivan sends the level task; we solve + card.
2. Create dev-env/linux-knowledge.md on the first durable general-OS fact.
3. Open Q: why is amneziavpn-bin pinned? Confirm before updating 4.8 → 5.0.

**Commit (notes-obsidian):** `arch-knowledge: pacman flag anatomy + maintenance;
add arch-pacman cards`

## 2026-07-24 — Desktop migration: Windows 10/WSL -> native CachyOS

**What we covered.** Debugged a flaky Windows desktop, then migrated it to
native CachyOS (KDE Plasma / Wayland / Btrfs / Limine) and re-provisioned the
whole dev environment.

**Key insight (the real root cause).** Two DIFFERENT faults were tangled:
- **Instant-off reboots** (Kernel-Power 41, `BugcheckCode 0` = power lost, no
  crash caught) — traced to AMD **C-state / idle power** collapsing the PSU rail
  at low load, NOT a dead PSU. Proven by: stress (MemTest, 68 min) stable but
  idle dies; Windows High-Performance plan masked it; USB-boot (no OS tuning)
  still died. Fixed in BIOS: **Global C-state Control = Disabled** (this AGESA
  didn't expose Power Supply Idle Control). BIOS update 3002->3636 also helped.
- **Freezes -> BSOD** — minidump showed `0x9F DRIVER_POWER_STATE_FAILURE`, a
  Windows driver hanging on a power-state transition. Windows-specific; a fresh
  Linux install sidesteps it. This was the actual reason Linux was worth doing.
- Hardware verified sound before wiping: MemTest86 clean (1 full pass, DDR4-3200
  = stock), PSU is a good Montech Century Gold 650W ~1yr old at ~20% load.

**What we built/changed.**
- `arch-setup.sh` in debian-p10k-zsh — a pacman-based mirror of `wsl-setup.sh`
  (apt->pacman, NodeSource dropped, /mnt/c SSH copy -> restore-or-generate, native
  docker, + wezterm-nightly/fonts/wl-clipboard/tree-sitter-cli/locale-gen).
- `.zshrc`: dropped tmux auto-start — tmux is now manual (a launcher entry / run
  `tmux`), which also killed the whole KDE `NO_TMUX` context-menu saga.
- `wezterm.lua`: single file branches on `target_triple` (see wezterm-knowledge).
- Dotfiles ported UNCHANGED (no /mnt/c, fdfind, clip.exe). Disks: wiped the 1 TB
  NVMe, kept the 223 GiB SATA (Windows 11 + backups) untouched.
- Office: WPS Office (+ ttf-wps-fonts) — ONLYOFFICE was laggy on the iGPU.

**Findings about this machine.**
- Ryzen 5 5600G APU (no dGPU) — "gaming" is light; AMD iGPU is great on Linux.
- WPS short-date format is CELL-format driven, not locale; forced Russian via
  `LANG`/`LC_TIME` on WPS's `.desktop` launchers.
- Limine wants the ESP at `/boot` (not `/boot/efi`) and >= 4 GiB (it holds kernels).

**Resume pointer / open threads.**
1. THE test: use it a few days. If instant-offs return, the C-state fix didn't
   hold -> reopen PSU (MemTest was clean, so RAM's ruled out).
2. Run `arch-setup.sh` end-to-end on the next fresh box to shake out bugs (this
   run hit: corepack is its own pkg; `ln -sfn` for the .wezterm dir link).
3. Nice-to-haves: try Sway later (KDE is the fallback); WinPodX only if WPS falls
   short for real Word/Excel.

**Commits (already drafted):** nvim-lsp — `wezterm: branch shell config on
target_triple`; debian-p10k-zsh — `add arch-setup.sh` + the tmux-manual `.zshrc`.

## 2026-07-03 - Networking fundamentals: subnetting, ARP, routing, NAT (on the VPS)

Covered (concept -> key insight):
- CIDR/masks: prefix = count of 1-bits, not an octet value. /26 = 255.255.255.192, not .4.
- Subnet math: block size = 256 - last-octet; /24 -> /26 = 4 subnets stepping 64, 62 usable each.
- ARP/NDP: LAN delivers by MAC; broadcast request, unicast reply. Key: ARP never leaves the
  subnet, so for outside IPs you resolve the GATEWAY's MAC.
- MAC vs IP across hops: IP dst constant end-to-end, MAC rewritten every hop.
- Routing: longest-prefix match; `onlink` explains the /32 interface + out-of-subnet default gw.
- ip route get: routing stage only, no NAT - proved routing and NAT are separate netfilter stages.
- NAT: MASQUERADE = SNAT in POSTROUTING, DNAT = inbound in PREROUTING. Confirmed the VPS runs
  DOUBLE NAT: 10.8.1.x -> 172.29.172.2 -> 153.76.117.52.
- FORWARD: policy DROP + Docker chains; ufw forward hooks show 0 pkts because Docker accepts
  container transit first (ufw = host INPUT only).

Analyzed real VPS output: ip neigh (live ARP/NDP cache; IPv6 gw FAILED), ip route (onlink default),
ip -6 route, ip route get (transit sim), iptables nat (PREROUTING DOCKER jump, two MASQUERADE layers,
DNAT udp 31657 -> amnezia container 172.29.172.2), FORWARD (DOCKER-USER/DOCKER-FORWARD, ufw bypass).

No code changed - learning + infra analysis session. No commit.

Open / resume:
- NEXT: expand DOCKER-FORWARD (`iptables -L DOCKER-FORWARD -vn`) to see the mirror ACCEPT rules
  for amn0 and close the loop on the VPN forwarding path.
- IPv6 gateway 2a12:bec4:1ac0::1 is FAILED (no NDP reply) - outbound IPv6 likely broken.
- conntrack pkg not confirmed installed; `apt install conntrack` if we want translation tuples.

## 2026-06-22 — WSL dev-env: tmux auto-start, clipboard mechanism, dotfile symlinks

**tmux auto-start ordering.** The auto-start `exec tmux` block must sit ABOVE p10k's
instant-prompt block in .zshrc — otherwise p10k grabs the TTY first and tmux dies with
"open terminal failed: not a terminal." Key insight: guard it with `[[ -t 1 ]]` (is fd 1
a tty?) so non-interactive/no-tty shells skip it cleanly. Deleted the stale duplicate
tmux block lower in .zshrc (it was below instant-prompt → would re-trigger the bug).

**Per-launch tmux opt-out.** Added a `NO_TMUX` escape hatch: `[[ -z "${NO_TMUX:-}" ]]`
in the guard, plus a second WezTerm launch_menu entry that runs
`wsl.exe -d Debian --cd ~ -- env NO_TMUX=1 zsh -li`. The `--` ends wsl.exe's own options;
the rest runs inside the distro. `env NO_TMUX=1` sets the var, interactive-login zsh
sources .zshrc, sees NO_TMUX, skips the auto-start → plain shell on demand.

**Clipboard — corrected understanding.** Copy reaches the Windows clipboard via OSC 52,
NOT via clip.exe. `set-clipboard external` (the default, confirmed via
`tmux show -gv set-clipboard`) makes tmux emit an OSC 52 escape sequence on every copy;
WezTerm decodes it onto the Windows clipboard. This fires for `copy-selection-and-cancel`
too — independent of `copy-command`. So `y` already works; removed the redundant
`if-shell … copy-command 'clip.exe'` line. Kept the vi `v`/`y` binds (they override
tmux's non-Vim defaults: Space/Enter, and v=rectangle-toggle).

**Dotfiles → symlinks.** Step 9 of wsl-setup.sh now symlinks (ln -sf) instead of copying,
guarded: `[[ -n "$CLONED_DOTFILES" ]]` → cp (temp-clone standalone run), else → ln -sf
(persistent repo). Live working copy: edits in ~ ARE edits in the repo, no drift.
Committed the executable bit on tmux-sessionizer at the source
(`git update-index --chmod=+x`, 100644 → 100755) and dropped the script's `chmod +x` —
a fresh clone now gets an executable file the symlink inherits.

**Open threads / resume:** none — verified working end to end (re-ran wsl-setup.sh, tmux
auto-starts, no-tmux launch entry works, clipboard yanks to Windows, sessionizer runs).

## 2026-06-17 — WSL dev-env: wsl-setup.sh hardening, tmux/zsh wiring, git workflow

### Covered (concept → key insight)
- **Sessionizer "opens & closes"**: `display-popup -E tmux-sessionizer` runs via a
  NON-interactive `zsh -c`, which sources `.zshenv` only — not `.zshrc`, where
  `~/.local/bin` was added. KEY: a pane's PATH ≠ the PATH a popup/`zsh -c` child gets.
  Fix: PATH export moved to `.zshenv`. Also: tmux auto-start belongs in interactive
  `.zshrc` so the server is born with the right PATH.
- **glob vs regex**: `find -name` = glob (`.` literal, `*` = any run); `fd` = regex
  (`.` = any char, `*` = zero+ of preceding). Same pattern, different meaning.
- **cp overwrite/merge**: `-n` deprecated on GNU (Debian warns) → use `--update=none`.
  `src/.` or `-rT` merge a dir's CONTENTS into dest without nesting.
- **trap**: `EXIT` is the catch-all — fires however the script leaves, so one
  `trap cleanup EXIT` covers every exit path; `ERR`/`INT`/`TERM` react to specific events.
- **mktemp -d**: race-safe, uniquely-named 0700 scratch dir vs mkdir's fixed name.
- **git auth**: HTTPS password auth was removed in 2021 → use SSH (`remote set-url
  git@github.com:…`). `id_ed25519` authenticates by default.
- **detached HEAD**: checking out a TAG detaches; for a live working copy, `git switch main`.
- **--ff-only**: pull only if it's a clean fast-forward; refuse (don't merge) on divergence.
- **lazy.nvim**: `:Lazy sync` UPDATES plugins + rewrites lazy-lock.json (the "churn");
  `:Lazy! restore` installs at the locked versions without rewriting → reproducible.

### Changed / built
- wsl-setup.sh: `trap cleanup EXIT` + `ERR` diagnostics; SSH repo URLs upfront +
  `ssh-keyscan github.com`; step 7 → live working copy (no tag checkout); step 8 →
  `git pull --ff-only`; graft via `cp -rT --update=none`; `.zshenv` copied + CRLF-stripped;
  pre-warm `sync` → `restore`; added shellcheck/info/bash-doc/cht.sh.
- `.zshenv` created (PATH); `.zshrc` tmux auto-start; `.tmux.conf` `prefix f` sessionizer;
  removed nvim `<C-f>` remap (single trigger now); default_prog = Git Bash, WSL via launch_menu.
- nvim-lsp: switched to `main`, fast-forwarded; ready to delete stale `nvim-0.11` branch.
- README regenerated WSL-only; decided to drop the Docker dev-container + Gemini entirely.
- Notes layout settled: per-topic workspace with `journal.md` (this) + `*-knowledge.md` (reference).
- Mentor prompt: generalized "Progress, handoffs & reference" with `wrap` + `ref` triggers.

### Open threads / next step
1. Commit + push repo cleanup: `git rm Dockerfile docker-compose.yml gemini-workflow.md`,
   `git rm -r .gemini`, drop README's old sections (done), commit.
2. Delete stale branch: `git push origin --delete nvim-0.11 && git branch -d nvim-0.11`.
3. `shellcheck wsl-setup.sh` and clear any findings.
4. Create `notes/dev-env/` in notes-obsidian + register it in obsidian.nvim; add
   `journal.md`, `bash-knowledge.md`, `git-knowledge.md`.
5. Apply the generalized section to mentor-main-prompt.md.

### Commit message
  chore: go WSL-only — drop Docker dev-container + Gemini

  - remove Dockerfile, docker-compose.yml, .gemini/, gemini-workflow.md
  - README rewritten for the wsl-setup.sh flow
  - wsl-setup.sh: SSH repo URLs + ssh-keyscan, live nvim working copy,
    notes pull --ff-only, Lazy restore (reproducible), trap cleanup/ERR
