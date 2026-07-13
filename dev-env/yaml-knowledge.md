---
id: yaml-knowledge
aliases: []
tags: []
---

# YAML Knowledge

## The whole data model: three node kinds
Everything in YAML is one of three, nested arbitrarily:
- **scalar** — a single value (string, number, bool, null)
- **sequence** — an ordered list
- **mapping** — key → value pairs

## Block vs flow style (same data, two spellings)
```yaml
# BLOCK — indentation + dashes (what Compose/Actions use)
ports:
  - "80:80"
  - "443:443"

# FLOW — inline, JSON-like: [] sequence, {} mapping
ports: ["80:80", "443:443"]
env: { NODE_ENV: production, PORT: 3000 }
```
YAML 1.2 is a SUPERSET of JSON: any JSON is valid flow-style YAML.

## Anchors (&), aliases (*), merge key (<<) — YAML's DRY
```yaml
x-defaults: &app_defaults   # &name = ANCHOR: label this node
  restart: unless-stopped
services:
  web:
    <<: *app_defaults       # <<: *name = MERGE the aliased mapping's keys in
    image: nginx
  api:
    <<: *app_defaults       # *name = ALIAS: reuse the labeled node
    image: node
```
> A value STARTING with `*` is read as an alias → why "*.md" needs quotes in GHA filters.

## Tags (!) — explicit type override
YAML implicitly types unquoted scalars (123→int, true→bool). A tag overrides that:
`port: !!str 8080` forces a string. Tools add custom tags (CloudFormation `!Ref`, `!GetAtt`).
> A value STARTING with `!` is read as a tag → why "!apps/**" needs quotes.

## Gotcha: implicit typing is aggressive ("the Norway problem")
```yaml
country: NO      # → boolean false, NOT "NO"   (Norway!)
answer: yes      # → true  (also on/off, y/n in YAML 1.1, which many parsers still use)
version: 1.10    # → float 1.1  (trailing zero dropped)
zip: 01234       # → may drop the leading zero
```
> Fix: quote anything you mean as a literal string — `"NO"`, `"1.10"`.

## When to quote a value
Quote if it STARTS with a syntax token: `* & ! [ { ? : -` (and to defuse implicit typing).
