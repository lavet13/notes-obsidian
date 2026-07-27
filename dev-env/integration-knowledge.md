---
id: integration-knowledge
aliases: []
tags: []
---

## "Headless" — two senses, and why the distinction decides your architecture

Headless = runs with no display and no human driving a UI: a cron job, a bot handler, a
server-side script. Nobody clicks anything. Two ways to BE headless against a remote site,
and which one you're forced into is set entirely by how that site authenticates:

### Headless HTTP client — just `fetch`
Works when auth is a documented endpoint you can call directly.

    // 1. exchange credentials for a token
    const { tokenManager } = await fetch("https://host/api/auth/manager", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ login, password }),   // creds from env, never committed
    }).then(r => r.json());

    // 2. send it on every real request
    await fetch("https://host/api/upload", {
      method: "POST",
      headers: { Authorization: `Bearer ${tokenManager}` },
      body: form,
    });

No browser, no window, no cookie jar. Ships as nothing but code. This is the path you want.

### Headless browser — Playwright / Puppeteer
A real Chromium with no visible window, scripted. You're forced here ONLY when the site has
no usable API — you have to automate typing into the login form, keep whatever session cookie
it sets, and re-script when the page HTML shifts. Costs: a browser binary in your Docker image,
brittleness against markup changes, and far more surface to break.

### The decision
Before automating anything against an external site, find the auth mechanism FIRST — it dictates
everything downstream:

- **documented login endpoint returning a token** → headless HTTP client (`fetch`). Cheap,
  robust, unattended.
- **only a browser session cookie** (login is a form POST, no API) → either a headless browser,
  or a human pasting a fresh cookie whenever it expires. The second barely beats doing the job
  by hand — it's not really "delegated".

Real case (donbass-post ingestion): the open risk was that `workplace-post.ru` might accept
uploads only from a logged-in browser session. Discovering `POST /api/auth/manager` →
`{ tokenManager }` → `Authorization: Bearer` collapsed the whole task from "maybe needs a headless
browser or manual cookies" to "a plain `fetch` on a server" — the auth mechanism was the pivot,
not the upload logic.

### Token handling for an unattended job
- Credentials live in env, never in the repo.
- Until you've measured the token's lifetime, fetch a fresh one per run rather than caching —
  a cached token that silently expires fails the job at the worst time.
- A 401 mid-run means re-auth, not retry-the-same-token.
