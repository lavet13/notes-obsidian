---
id: track-global-knowledge
aliases: []
tags: []
---

# track.global proxy — feature patterns

## Cache-first proxy with found/miss TTL + stale-on-failure

The quota-defense shape for a rate-limited upstream (10/day):

1. Read cache by key (normalized: trim + uppercase the track). Fresh? serve, 0 upstream cost.
2. Miss/stale → fetch upstream ONCE (the only place quota is spent).
3. On 429 (quota gone) → serve stale cached row if any, else a friendly "limit" message.
4. On network reject → serve stale if any, else 503.
5. On success → upsert cache, serve.

- Two TTLs by outcome: found → 6h (history is append-only, moves slowly); miss → 30min (a "not found" is often "not registered yet" — re-check sooner so a real parcel isn't frozen as missing).
- `isFresh(row)` = `Date.now() - row.fetchedAt < (row.found ? 6h : 30min)`. Impure (reads the clock) but domain policy → lives in track-global/service.ts, not utils/.
- Only cache a successful HTTP response; never cache a network/error failure.

## Exact-match router has no path params — use searchParams

The hand-rolled router (router.ts) matches paths by exact Map lookup — no `:param` support. `/api/track-global/:track` silently 404s. Read the value from the query string inside the handler: `new URL(request.url).searchParams.get("track")`. (Adding real param matching = compile regex + precedence + a params bag on the immutable Request via WeakMap — not worth it for one dynamic route.)

## Empty-string vs missing in the payload

track.global sends `""` (not null/absent) for fields it doesn't populate — fromLocation/toLocation/location for aliexpress. `value || "?"` fires on "" (falsy) → renders "? → ?". Guard with `[a,b].filter(Boolean).join(" → ")` so blanks drop instead of showing placeholders.

## Slim the SEO blob before caching

Each service object carries a ~40KB `full_text` article, duplicated in found_in_services + checkedServices. UI needs only name + image_path. `slimTrackData()` keeps {id,name,alias,image_path}, drops checkedServices — ~230KB rows → ~5-9KB. Pure + domain-specific → feature folder.
