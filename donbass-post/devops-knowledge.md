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

# Anki's cards

Card 29 - silent hang diagnosis (application)

Front: Ops - a container is Up, the process is alive, env is injected, but logs stop after one line and none of your startup logs appear (no crash, no stack trace). What class of failure is this?
Back: A hang on an await to an unreachable external host - not a crash. A crash leaves a stack trace; silence after startup means a promise that never resolves (e.g. bot.init() calling a blocked API). Confirm by testing reachability from inside the container before touching code.

Card 30 - confirm the layer (recall)

Front: Debugging - what is the discipline before applying a fix to a production incident?
Back: Confirm which layer is actually broken with evidence (ps, env echo, reachability test) before theorizing a fix. Different causes have different fixes; diagnose before you patch.

Card 31 - fail loud not silent (application)

Front: Resilience - an external call (e.g. bot.init to a network API) can hang forever with no log. How do you make the failure diagnosable?
Back: Race it against a timeout (withTimeout(promise, ms, label) using Promise.race) so it rejects with a logged error and exits non-zero (container restarts) instead of becoming a silent zombie. Mitigates the symptom; does not fix the underlying reachability.
