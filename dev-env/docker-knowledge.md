---
id: docker-knowledge
aliases: []
tags: []
---

# Docker Knowledge

## Inspecting disk usage (all read-only, safe)

```bash
docker system df                 # summary: images/containers/volumes/build-cache + reclaimable
docker system df -v              # verbose: per-object; SHARED SIZE vs UNIQUE SIZE per image
docker images                    # (containerd store) IMAGE ID | DISK USAGE | CONTENT SIZE | In Use
docker image ls --tree           # per-platform layer/size breakdown
docker ps -a --size              # containers: writable-layer size (+ virtual = writable+image)
docker history <image>           # per-LAYER size + the instruction that created it
```

## df vs du: logical size vs physical blocks

- Docker reports LOGICAL (uncompressed) sizes. `du -sh /var/lib/docker` reports
  PHYSICAL blocks actually allocated.
- On btrfs + zstd (CachyOS default) physical < logical, so `du` < sum of image sizes.
  This is compression, NOT shared-layer inflation — check SHARED SIZE (0 = no sharing).
- Direction test: uncompressed, `du` would be LARGER than the unpacked total (content-store
  blobs sit on disk on top of unpacked snapshots). `du` smaller => compression.

```bash
findmnt -no FSTYPE,OPTIONS -T /var/lib/docker   # look for btrfs + compress=zstd
sudo compsize /var/lib/docker                    # definitive compressed/uncompressed ratio
```

- New `docker images` columns (containerd image store):
  DISK USAGE = unpacked size on disk
  CONTENT SIZE = compressed blob size in content store (~ download size)

## Dangling vs unused images

- DANGLING = untagged `<none>:<none>` layers, usually left when a tag moves to a rebuild.
- `docker image prune` -> dangling only.
- `docker image prune -a` -> ALL images with ZERO container references (tagged included).
- A container reference (RUNNING or STOPPED) protects its image from prune -a.
  New CLI flags in-use images with `U` / `In Use`.

## Untag / retag

```bash
docker tag SRC:tag NEW:tag   # add a tag (image can hold many)
docker rmi NAME:tag          # drop THAT tag; image stays if other refs remain, else deleted
# no dedicated `docker untag` verb; rmi <tag> IS the untag when other refs remain
```

## Prune defaults (the gotchas)

```bash
docker system prune              # stopped containers + unused nets + DANGLING images + build cache
docker system prune -a           # + ALL unused images
docker system prune -a --volumes # + ANONYMOUS unused volumes only (NOT named ones!)
docker volume prune              # ANONYMOUS unused volumes only (default)
docker volume prune -a           # + NAMED unused volumes  <- the one that eats DB volumes
```

- `system prune --volumes` never removes NAMED volumes; use `volume prune -a` for those.
- Pruning is irreversible — no `docker unprune`. Inspect first; back up volumes.

## "Unused volume" = zero container references (evaluated at prune time)

- `docker compose down` (no -v) removes containers but KEEPS named volumes -> now unused
  -> `docker volume prune -a` will delete them. `compose down -v` removes them deliberately.
- The daemon is GLOBAL: `docker ps` / `images` / `volume ls` show everything, not just the
  current project's. A `<project>_<key>` volume name tells you which compose project owns it.

## Volume backup / restore (ephemeral helper pattern)

```bash
# BACKUP: throwaway alpine, volume read-only at /v, host cwd at /backup
docker run --rm -v <vol>:/v:ro -v "$PWD":/backup \
  alpine tar czf /backup/<vol>.tar.gz -C /v .
# RESTORE (reverse): volume writable, untar back in
docker run --rm -v <vol>:/v -v "$PWD":/backup \
  alpine sh -c 'tar xzf /backup/<vol>.tar.gz -C /v'
```

- Two jobs in that command: `-f /backup/<vol>.tar.gz` (ABSOLUTE) sets WHERE the archive is
  written — unaffected by `-C` or pwd. `-C /v .` sets WHAT is packed: chdir into /v, then
  `.` = its contents. `.` is the INPUT, not the output.
- `-C /v .` = pack the volume's contents with relative member paths (mechanics:
  bash-knowledge.md ## `tar` create side). `--rm` cleans up the helper only, never the volume.
- `--rm` cleans up the helper only; it does NOT orphan the volume — the real owner
  container still references it throughout (LINKS 1->2 during, back to 1 after).
- LIVE DB: don't tar a running DB's volume — see ## Database backups below.

## docker exec flags — `-i`, `-t`, `-e`

`docker exec` (and `docker run`) run a command in a container. Three flags worth knowing:

```bash
-i / --interactive   # keep the container process's STDIN open — needed to PIPE data IN.
#   restore: cat dump.sql | docker exec -i <c> psql -U postgres
#   Without -i the process gets instant EOF -> reads nothing -> silent no-op.
#   A dump (output redirected OUT) does NOT need -i.

-t / --tty           # allocate a pseudo-TTY. For an INTERACTIVE shell: docker exec -it <c> bash
#   FOOTGUN: a pty maps \n -> \r\n, so `docker exec -t <c> pg_dumpall > dump.sql` writes CRLF
#   and corrupts the dump (pg_restore can segfault). NEVER use -t when redirecting to a file.

-e / --env           # set an env var for THAT exec (fresh env each call -> repeats per command).
#   Secrets: -e MYSQL_PWD=<pw> passes the password via ENV, not argv. Why it matters:
#     /proc/<pid>/cmdline (argv)  is world-readable (0444) -> -p<pw> leaks to any user on the box
#     /proc/<pid>/environ (env)   is owner-only     (0400)
#   NUANCE: -e MYSQL_PWD=<literal> is STILL in the host `docker exec` argv while it runs, and in
#   shell history. Cleanest = bare `-e MYSQL_PWD` (NO value): docker forwards the host env var,
#   so the value is in NO argv; source it from a 0600 file to keep it out of history too:
#     umask 077; printf %s 'pw' > ~/.dbpw; export MYSQL_PWD="$(cat ~/.dbpw)"
#     docker exec -e MYSQL_PWD <c> mysqldump -u root <db> > db.sql
```

- `-it` together is the usual interactive-shell combo (`docker exec -it <c> bash`); keep it away
  from redirected dumps, where the `-t` half corrupts output.

## Database backups (logical vs file-level)

A running DB has buffered/in-flight writes, so a file-level tar of its volume can be torn
and inconsistent. Two safe options:

- LOGICAL dump while running (preferred) — transaction-consistent:

```bash
# postgres — all DBs + roles. NO -t: a pty rewrites \n->\r\n and corrupts the dump
# (pg_restore can segfault on it). Adjust -U to your DB user.
docker exec <pg_container> pg_dumpall -U postgres > dump-$(date +%F).sql
cat dump-YYYY-MM-DD.sql | docker exec -i <pg_container> psql -U postgres   # -i: feed stdin
# (single DB instead of all: pg_dump -U postgres <db>)

# mysql/mariadb — inline -p<pw> is world-readable in `ps` (argv); pass via env instead:
docker exec -e MYSQL_PWD=<pw> <mysql_container> mysqldump -u root <db> > db-$(date +%F).sql
cat db-YYYY-MM-DD.sql | docker exec -i -e MYSQL_PWD=<pw> <mysql_container> mysql -u root <db>
```

- FILE-LEVEL copy — only when the container is STOPPED; then the volume-tar pattern above
  is consistent.

## Legacy builder deprecation

- The old (non-BuildKit) builder is deprecated. Install buildx so BuildKit is default:
  sudo pacman -S docker-buildx # reverse: sudo pacman -R docker-buildx
