---
id: devops-knowledge
aliases:
  - devops-knowledge
tags: []
---

# DevOps Knowledge

## Diagnosing a "silent" container (Up but doing nothing)

Symptom: container is `Up` (not crash-looping), process is alive, but logs stop
after an early line and none of your startup logs appear - no crash, no stack
trace. This is a HANG on an `await` to an unreachable external host, not a
crash. A crash leaves a stack trace; silence after startup means a promise that
never resolves. Do not assume it is a code bug.

## Confirm the layer before fixing

A production incident has several possible causes with different fixes. Gather
evidence before theorizing:
- `docker compose ... ps` - is it Up, Restarting, or exited? (crash loop vs hang)
- `... logs <svc> --tail 100` - where does startup stop? (not `-f`, which only
  shows new output and misses a past death)
- `... exec <svc> ps aux` - is the running process actually what you expect
  (e.g. `node dist/server.js`), and alive?
- `... exec <svc> sh -c 'echo ${VAR:+set}'` - are env vars actually injected?
- reachability test from INSIDE the container (`wget`/`curl` the external host).
Confirm which layer is broken, THEN fix it. Diagnose before you patch.

## Env comes from Compose env_file, not (necessarily) dotenv at runtime

`[dotenv] injecting env (0)` meaning "zero vars from .env" can be NORMAL: in a
Compose stack the vars come from the `env_file:` directive, which injects them
into the container ENVIRONMENT directly. Verify with
`exec <svc> sh -c 'echo ${TOKEN:+yes}'` (checks the environment, independent of
dotenv reading a file). Do not chase the `(0)` as a bug before confirming the
environment itself is empty.

## Container localhost != host localhost

Inside a container, `localhost` is the container itself, not the host. A service
on the host bound to `127.0.0.1` will not accept connections from a container.
Fixes: bind the host service to `0.0.0.0`, and from the container reach the host
via its container-visible address (e.g. `host.docker.internal` under Docker
Desktop). Same class of problem as the DooD socket-mount: a service on one side,
a client on the other, and `localhost` meaning different things on each.
(WSL2-hosted Neovim reaching a Windows-host service on localhost can work
directly - the boundary depends on where each process actually runs.)

## Egress blocks are routing, not DNS config

If a container cannot reach an external API (e.g. Telegram from a Russian
network), and the cause is the network BLOCKING that destination, a `dns:` entry
in compose will not help - it is a routing/egress problem, not name resolution.
The fix is to route egress through a reachable path (a proxy, or a WG tunnel to
a VPS in an unblocked region), not to tweak DNS. Confirm by whether name
resolution works but the connection hangs (routing) vs the name not resolving
(DNS).

## Polling vs webhook (outbound vs inbound dependency)

- Polling: the bot PULLS updates (outbound connection). Robust - only needs
  OUTBOUND reachability to the API; no inbound DNS/TLS required. A deploy flood
  is just queued updates draining, zero loss.
- Webhook: the API PUSHES to you (inbound). Depends on inbound DNS + TLS being
  reachable, so it is sensitive to dynamic-DNS flakiness. For an egress-only
  problem, polling means you only need to fix OUTBOUND reachability.

## Fail loud, not silent (timeout external calls)

An external call that can hang forever (e.g. an SDK init that pings a remote
API) should be raced against a timeout, so it rejects with a LOGGED error and
exits non-zero (container restarts) instead of becoming a silent zombie:
`Promise.race([promise, rejectAfter(ms)])`. This mitigates the SYMPTOM (makes
failure visible and self-healing via restart); it does not fix the underlying
reachability. Note: the timer keeps running after the promise wins - harmless
for one-shot startup, but `clearTimeout` in a `.finally` if used on a hot path.

## Docker multi-stage: builder vs runner, production-only deps

A builder stage installs all deps and compiles; the runner stage copies only the
built output plus production deps (`yarn workspaces focus <pkg> --production`).
Keeps the final image small. A `YN0028`/lockfile error in the builder often
means a workspace `package.json` is missing for a package not represented in the
builder stage - the lockfile expects a workspace that was not copied in.

## Compose depends_on cascades and bind-mount wipes (two real deploy footguns)

- Restarting a service that others `depends_on` can cascade-restart them
  unintentionally. To reload config without a cascade, signal the process
  directly (`nginx -s reload`), not a container restart.
- A "clean" step that removes a directory which is ALSO a bind-mount target can
  wipe host files (e.g. `nginx/conf.d/`). Remove selectively; never blow away a
  directory that the host has mounted in.

## expose vs ports (documentation vs security)

On a modern bridge network, `expose` is documentation-only - it does not publish
a port to the host. `ports: []` is what actually controls host exposure. A
service reached only via internal DNS (e.g. `http://telegram-bot:3000` from
nginx) needs NO published ports at all; omit them so nothing is reachable from
outside the Docker network.

## Idempotent seed (safe to run every deploy)

A seed run on every deploy must be idempotent: `upsert` everywhere (including
join tables via their composite unique key), and gate any one-time backfill /
env-bootstrap behind an emptiness check so it does not re-assert or resurrect
state on repeat runs. Call the COMPILED seed directly in the container
(`node dist/prisma/seed.js`) - no tsx, no network installs at deploy time.

## GitHub Actions path filters (`on.push.paths`)

`paths:` is an allowlist — the workflow runs if AT LEAST ONE changed file matches a pattern.

````yaml
on:
  push:
    branches: ["main"]
    paths:
      - "apps/telegram-bot/**"      # include everything under the app…
      - "!apps/telegram-bot/*.md"   # …EXCEPT top-level docs. The negation MUST come AFTER the
                                    # positive glob it carves out of — last matching pattern wins.
````

Rules:
- Can't mix `paths` and `paths-ignore` for one event. To include AND exclude, use `paths` + `!`.
- Evaluated against the WHOLE changeset: a push touching a doc AND a source file still runs (the
  source matches). Exclusion only skips pushes that are EXCLUSIVELY excluded files.
- A push skipped by path filtering creates NO run at all (don't hunt for a "skipped" entry).
- `workflow_dispatch` ignores paths — always manually runnable.

GHA filter glob is its OWN dialect (not shell glob, not full regex):
- `*` = zero+ chars but NOT `/`;  `**` = zero+ chars INCLUDING `/` (recursive).
- `?` = zero-or-one of the PRECEDING char (regex-style quantifier!), `+` = one-or-more.
  → `*.jsx?` matches BOTH `page.js` and `page.jsx` (trailing x optional). NOT shell "any char".
- Patterns are ANCHORED to the whole path from repo root (why negations spell the full path).
- No brace expansion — list `**/*.ts` and `**/*.tsx` separately.
- `*`, `[`, `!` are special in YAML → quote patterns starting with them.

```yaml
# No brace expansion. The shell turns {ts,tsx} into "ts tsx"; GHA does NOT — it reads the braces
# LITERALLY (a file named "…{ts,tsx}"), so the pattern matches nothing. List each on its own line:
paths:
  - "**/*.ts"
  - "**/*.tsx"       # NOT "**/*.{ts,tsx}" — that never matches anything

# YAML quoting. A value that STARTS with a syntax token is parsed as that token, not as text:
#   *  = alias    &  = anchor    !  = tag    [ or { = flow collection    (? : - in places too)
# So an unquoted pattern starting with one is a parse error or mis-parse. Quote it → literal string:
paths:
  - "!apps/telegram-bot/*.md"   # unquoted !… → YAML reads a tag → error
  - "*.md"                      # unquoted *…  → YAML reads an alias → error
  - "[a-z]*.ts"                 # unquoted [   → YAML reads a flow sequence → wrong
```

## nginx micro-caching a proxied endpoint (proxy_cache)

Two-layer cache model: nginx 30s microcache (collapses request bursts) OVER the node DB cache (saves upstream quota). Different jobs — nginx protects against concurrent duplicate work, node protects the quota.

```nginx
# http context (by limit_req_zone):
proxy_cache_path /var/cache/nginx/track_global levels=1:2 keys_zone=track_global:1m max_size=50m inactive=10m;
#   levels=1:2       nest cache files in subdirs (filesystems hate flat huge dirs)
#   keys_zone:NAME:1m  shared-mem for KEYS+metadata (not bodies); 1m ≈ 8000 keys
#   max_size         disk cap for bodies; LRU-evicts past it
#   inactive         drop entries not READ within this window

# location = /api/track-global (exact match beats the /api/ prefix):
proxy_cache track_global;
proxy_cache_key "$request_uri";      # path+query → one entry per parcel; query is the identity
proxy_cache_valid 200 30s;           # only cache 200s, briefly
proxy_cache_lock on;                 # burst-collapse: N simultaneous misses on the SAME key → 1 upstream call, the rest wait for it
add_header X-Cache-Status $upstream_cache_status always;  # HIT/MISS/EXPIRED visible in DevTools
```
- `EXPIRED` is normal, not an error: entry aged past proxy_cache_valid, nginx revalidates by re-asking upstream.
- add_header in a location REPLACES inherited add_headers → re-declare server-level security headers in the block.
- Vary: Origin from upstream is honored automatically — nginx keys per-origin on its own. (Our node cors already sends it, so no need to fold origin into proxy_cache_key.)
- nginx won't cache if upstream sends Set-Cookie / Cache-Control:no-cache. Our node returns a fresh Response.json (none of those), so it caches. Only add `proxy_ignore_headers Cache-Control Set-Cookie;` if a HIT never appears.

## Test nginx config locally without ACME/prod
`nginx -t` never contacts Let's Encrypt — it only parses. Danger is only in `docker compose up` of the certbot image. Validate in a throwaway plain-nginx container; two things trip -t because it FULLY loads config:
```bash
# fake certs at the paths the config references
CERTDIR=/tmp/le/live/donbass-post.duckdns.org; mkdir -p "$CERTDIR"
openssl req -x509 -newkey rsa:2048 -nodes -days 1 -keyout "$CERTDIR/privkey.pem" -out "$CERTDIR/fullchain.pem" -subj "/CN=localhost"
cp "$CERTDIR/fullchain.pem" "$CERTDIR/chain.pem"
docker run --rm -v "$PWD/nginx/conf.d:/etc/nginx/conf.d:ro" -v /tmp/le:/etc/letsencrypt:ro \
  --add-host telegram-bot:127.0.0.1 nginx:alpine nginx -t
#   --add-host  → plain proxy_pass hostnames resolve at config load; without this, -t errors "host not found in upstream"
```

## docker compose file auto-loading
Plain `docker compose up` loads docker-compose.yml ALWAYS + docker-compose.override.yml IF present. Any `-f` flag disables ALL auto-loading — only the listed files, in flag order, later overriding earlier. So `-f docker-compose.prod.yml` ALONE is incomplete (no base) → prod always needs `-f docker-compose.yml -f docker-compose.prod.yml`.
