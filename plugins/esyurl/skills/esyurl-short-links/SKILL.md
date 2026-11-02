---
name: esyurl-short-links
description: "Shorten URLs and manage short links with esyURL (https://esyurl.fyi): custom slugs, change where a link already shared points, smart redirects (iPhone to the App Store and Android to Google Play, or by country, language, device or time), click and QR scan analytics, campaign groups, and short links on your own branded domain. Use when the user wants to shorten a URL, make a short, branded or app download link, fix a link already printed or sent, route visitors by device or country, or see how many people clicked."
---

# Short links with esyURL

esyURL (https://esyurl.fyi) gives a URL a short, stable address
(`https://esyurl.fyi/{slug}`) that redirects to it. The destination can be
changed later; everything already shared keeps working.

## Signing in

This plugin connects the `esyurl` MCP server; use its tools. The first time a
tool is called, Claude Code asks the user to sign in to esyurl.fyi in the
browser (one-click account, email, or an existing key). If a tool answers
that authentication is needed, tell the user to run `/mcp`, pick `esyurl` and
authenticate. Never ask the user to paste a key into the chat.

## Shorten a URL

`create_short_link`:

```json
{"targetUrl": "https://example.com/spring-sale?utm_source=print", "slug": "spring-sale",
 "name": "Spring sale flyer", "tags": ["spring"]}
```

The response has `linkId` (used by the other tools) and `shortUrl`; give the
user the `shortUrl`.

- `targetUrl` must be an absolute `http(s)` URL.
- `slug` is optional (8 random characters otherwise): 3–64 of
  `[a-zA-Z0-9_-]`, globally unique (409 if taken), and **cannot be changed
  later**. Route names such as `v1`, `mcp`, `api`, `admin` are reserved (400).
- Query strings on the short URL are not forwarded: put UTM parameters in
  `targetUrl`.

## Manage links

| Task | Tool |
| --- | --- |
| List (newest first, filter by `tag` or `group`; follow `nextCursor`) | `list_short_links` |
| Get one | `get_short_link` |
| Change destination | `update_short_link` `{"linkId", "targetUrl"}` |
| Pause (answers 410) / resume | `update_short_link` `{"linkId", "status": "paused" \| "active"}` |
| Delete | `delete_short_link` |

`update_short_link` also takes `name`, `tags` (≤ 20), `metadata` (≤ 20 string pairs),
`groupIds` (replaces the set) and `rules`. Changing `targetUrl` is the point
of a short link: suggest it instead of making a new link when a destination
moves.

## Groups (campaigns)

`create_group` `{"name": "Spring campaign", "color"?: "#3b82f6"}`, then
`add_links_to_group` `{"groupId", "linkIds": [...]}` (up to 25, idempotent),
or pass `groupIds` when creating links.

## Redirect rules

One link can send different visitors to different places: iPhones to the
App Store and Android to Google Play, each country to its own page, a page
that switches on launch day.

### How rules work

`rules` on a link is an ordered list. On each visit the first enabled rule
whose `when` matches picks `targetUrl`; if none matches, the link's own
`targetUrl` is the fallback. Rule: `{"id"?, "name"?, "enabled"?: true, "when": <condition>, "targetUrl"}`.
`update_short_link` replaces the whole list (`[]` removes all); keep rule `id`s when
editing, since per-rule stats are keyed by them.

### App-store split

`update_short_link`:

```json
{"linkId": "...", "targetUrl": "https://example.com/app",
 "rules": [
   {"name": "iOS", "when": {"field": "os", "op": "eq", "value": "ios"},
    "targetUrl": "https://apps.apple.com/app/id123456789"},
   {"name": "Android", "when": {"field": "os", "op": "eq", "value": "android"},
    "targetUrl": "https://play.google.com/store/apps/details?id=com.example.app"}]}
```

`rules` can also be sent to `create_short_link`.

### Conditions

`{"field", "op", "value"}`, or combine with `{"all": [...]}`,
`{"any": [...]}`, `{"not": {...}}`.

| field | values | ops |
| --- | --- | --- |
| `os` | ios android windows macos linux chromeos other | eq neq in not_in |
| `device` | mobile tablet desktop bot | eq neq in not_in |
| `browser` | safari chrome firefox edge samsung opera in_app other | eq neq in not_in |
| `source` | qr direct bot | eq neq in not_in |
| `language` | primary tag; `pt` matches `pt-BR` | eq neq in not_in exists |
| `country` | ISO 3166-1 alpha-2, e.g. `GB` | eq neq in not_in |
| `referrer` | host, e.g. `l.instagram.com` | eq neq in not_in contains starts_with ends_with exists |
| `query` | a query parameter; needs `"key"` | eq neq in not_in contains starts_with ends_with exists |
| `time` | ISO 8601 | before after |
| `weekday` | sun..sat (UTC) | eq neq in not_in |
| `hour` | 0–23 (UTC) | eq neq in not_in gte lte |

`in`/`not_in` take arrays. A missing value (no referrer, unknown country)
fails `eq`/`in` and passes `neq`/`not_in`. iPads on Safari report as macOS
desktop. Up to 20 rules, 20 conditions each, nested 4 deep.

More examples:
- `{"field": "country", "op": "in", "value": ["BR", "PT"]}`
- `{"all": [{"field": "language", "op": "eq", "value": "pt"}, {"field": "device", "op": "eq", "value": "mobile"}]}`
- `{"field": "time", "op": "after", "value": "2026-12-01T00:00:00Z"}` (launch-day switch)
- `{"field": "browser", "op": "eq", "value": "in_app"}` (Instagram/TikTok in-app browsers)

### Test before sharing

`test_redirect_rules` `{"linkId", "userAgent": "Mozilla/5.0 (iPhone; CPU iPhone OS 18_0 like Mac OS X)", "country": "GB", "source": "qr"}`
returns the `targetUrl` and `matchedRule`; records nothing. Check each branch
plus the fallback.

Visits per rule: `byRule` in the link's visits (below).

## Visits (clicks and QR scans)

Every visit is counted by source: QR scans, direct clicks, and bots
(link-preview fetchers like Slackbot or WhatsApp, kept out of human counts).

`get_short_link_visits` `{"linkId", "days": 30, "recent": 50}`. The response has:

- `totals`: `{visits, qr, direct, bot}`. `visits` is humans only (qr + direct).
- `daily`: one row per UTC day.
- `recent`: the latest visit events, newest first.
- `byRule`: visits each redirect rule decided, and `fallback` for the rest.

Find a link's `linkId` with `list_short_links`; each link
also carries `visitCount` and `lastVisitedAt`.

### Visits for a group

`get_group_visits` `{"groupId", "days": 30}` sums the group's current links.

### Reading the numbers

- QR scans are counted only for codes rendered by esyURL (they encode
  `?s=qr`). A QR code of the raw destination URL can't be counted.
- HEAD requests aren't visits. `curl` and HTTP libraries count as direct.
- Individual events are kept 90 days; daily counts forever.
- When summarising, give totals, the QR vs direct split, the busiest days and
  the trend; mention bots separately.

## Custom domain

An account's short links can live on its own domain, such as
`go.acme.com/spring`. Once active, short URLs and QR codes use it; the
`esyurl.fyi` addresses keep working.

### Cost: ask first

For a self-signed-up account, setting up a domain costs **5.00 USDC on Base,
once per domain**, paid with x402. Reading, re-setting the same domain and
removing it are free; switching to another domain is a new payment.
Accounts whose key came from esyURL directly don't pay.

**Tell the user the price and get explicit approval before paying.** If they
have no x402 wallet, stop and explain.

### Set it up

1. `set_custom_domain` `{"domain": "go.acme.com"}`. One domain per account.
   Invalid or taken domains fail before any charge. Otherwise the tool
   returns an error whose `structuredContent` holds the x402 payment
   requirements.
2. With approval, sign the payment with an x402 client and call
   `set_custom_domain` again with the signed payload in the `payment`
   argument (or `_meta["x402/payment"]`). Success means set up; the payment
   error again means settlement failed and nothing was set up.
3. `get_custom_domain` returns the status and the exact DNS records to
   create. **Each call advances provisioning**, so call it again after DNS
   changes.

Status goes `pending_validation` (create the validation CNAME) →
`provisioning` → `pending_dns` (CNAME the domain to the given target) →
`active`. Give the user the records exactly as returned; DNS can take minutes
to hours.

Remove with `remove_custom_domain`.

Payment details: https://esyurl.fyi/llms.txt, "Pricing".

## Limits

Self-signed-up accounts hold up to 100 links (403 `quota_exceeded` beyond).
QR codes for these links: the `esyurl-qr-codes` skill.

## Reference

The live documentation wins over anything here:
https://esyurl.fyi/llms.txt (API, MCP tools, limits) and
https://esyurl.fyi/openapi.json (full schema). Errors are JSON:
`{"error": {"code", "message", "details"?}}`; retry 5xx with backoff, don't
repeat a 4xx unchanged.
