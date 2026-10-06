---
name: esyurl-short-links
description: "Shorten URLs and manage short links with esyURL (https://esyurl.fyi): custom slugs, change where a link already shared points, smart redirects (iPhone to the App Store and Android to Google Play, or by country, language, device or time), click and QR scan analytics, campaign groups, and short links on your own branded domain. Use when the user wants to shorten a URL, make a short, branded or app download link, fix a link already printed or sent, route visitors by device or country, or see how many people clicked. The agent can sign itself up; no human account is needed."
---

# Short links with esyURL

esyURL (https://esyurl.fyi) gives a URL a short, stable address
(`https://esyurl.fyi/{slug}`) that redirects to it. The destination can be
changed later; everything already shared keeps working.

## Get an API key

Keys look like `esy_<keyId>_<secret>` and do not expire. Use the first of:

1. The `ESYURL_API_KEY` environment variable.
2. `~/.config/esyurl/credentials.json` (`{"api_key": "...", "identity_assertion": "..."}`).
3. An `esyurl` MCP server already configured in this client: use its tools.

If there is none, sign up. Each sign-up creates a new, empty account, so do
it **once** and save the result:

```sh
BASE=https://esyurl.fyi
REG=$(curl -s $BASE/agent/identity -H 'content-type: application/json' -d '{"type":"anonymous"}')
ASSERTION=$(echo "$REG" | jq -r .identity_assertion)
KEY=$(curl -s $BASE/oauth2/token \
  --data-urlencode grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer \
  --data-urlencode assertion="$ASSERTION" | jq -r .access_token)
mkdir -p ~/.config/esyurl && chmod 700 ~/.config/esyurl
printf '{"api_key":"%s","identity_assertion":"%s"}\n' "$KEY" "$ASSERTION" > ~/.config/esyurl/credentials.json
chmod 600 ~/.config/esyurl/credentials.json
```

Tell the user an account was created and where the key is; never print the
key in full. A 401 on a key that used to work: exchange the saved
`identity_assertion` at `/oauth2/token` again, and sign up only if that
answers `invalid_grant`. Details: https://esyurl.fyi/auth.md.

Send the key as `Authorization: Bearer <key>` to `https://esyurl.fyi/v1/...`,
or connect the MCP server (Streamable HTTP):

```sh
claude mcp add --transport http esyurl https://esyurl.fyi/mcp --header "Authorization: Bearer $KEY"
```

A newly added MCP server may only load in a new session; use REST until then.

## Shorten a URL

```sh
curl -s https://esyurl.fyi/v1/links -H "Authorization: Bearer $KEY" -H 'content-type: application/json' \
  -d '{"targetUrl": "https://example.com/spring-sale?utm_source=print", "slug": "spring-sale", "name": "Spring sale flyer", "tags": ["spring"]}'
```

MCP: `create_short_link`. The response has `linkId` (the `{id}` below) and `shortUrl`; give the
user the `shortUrl`.

- `targetUrl` must be an absolute `http(s)` URL.
- `slug` is optional (8 random characters otherwise): 3–64 of
  `[a-zA-Z0-9_-]`, globally unique (409 if taken), and **cannot be changed
  later**. Route names such as `v1`, `mcp`, `api`, `admin` are reserved (400).
- Query strings on the short URL are not forwarded: put UTM parameters in
  `targetUrl`.

## Manage links

| Task | REST | MCP |
| --- | --- | --- |
| List (newest first) | `GET /v1/links?limit=20&tag=...&group=...`, follow `nextCursor` | `list_short_links` |
| Get one | `GET /v1/links/{id}` | `get_short_link` |
| Change destination | `PATCH /v1/links/{id}` `{"targetUrl": "..."}` | `update_short_link` |
| Pause (answers 410) / resume | `PATCH /v1/links/{id}` `{"status": "paused" \| "active"}` | `update_short_link` |
| Delete | `DELETE /v1/links/{id}` | `delete_short_link` |

`PATCH` also takes `name`, `tags` (≤ 20), `metadata` (≤ 20 string pairs),
`groupIds` (replaces the set) and `rules`. Changing `targetUrl` is the point
of a short link: suggest it instead of making a new link when a destination
moves.

## Groups (campaigns)

`POST /v1/groups` `{"name": "Spring campaign", "color"?: "#3b82f6"}`, then
`POST /v1/groups/{id}/links` `{"linkIds": [...]}` (up to 25, idempotent), or
pass `groupIds` when creating links. MCP: `create_group`,
`add_links_to_group`.

## Redirect rules

One link can send different visitors to different places: iPhones to the
App Store and Android to Google Play, each country to its own page, a page
that switches on launch day.

### How rules work

`rules` on a link is an ordered list. On each visit the first enabled rule
whose `when` matches picks `targetUrl`; if none matches, the link's own
`targetUrl` is the fallback. Rule: `{"id"?, "name"?, "enabled"?: true, "when": <condition>, "targetUrl"}`.
`PATCH` replaces the whole list (`[]` removes all); keep rule `id`s when
editing, since per-rule stats are keyed by them.

### App-store split

```sh
curl -s -X PATCH https://esyurl.fyi/v1/links/$ID -H "Authorization: Bearer $KEY" -H 'content-type: application/json' -d '{
  "targetUrl": "https://example.com/app",
  "rules": [
    {"name": "iOS", "when": {"field": "os", "op": "eq", "value": "ios"},
     "targetUrl": "https://apps.apple.com/app/id123456789"},
    {"name": "Android", "when": {"field": "os", "op": "eq", "value": "android"},
     "targetUrl": "https://play.google.com/store/apps/details?id=com.example.app"}]}'
```

`rules` can also be sent when creating the link (`POST /v1/links`). MCP:
`create_short_link` / `update_short_link`.

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

```sh
curl -s https://esyurl.fyi/v1/links/$ID/resolve -H "Authorization: Bearer $KEY" -H 'content-type: application/json' \
  -d '{"userAgent": "Mozilla/5.0 (iPhone; CPU iPhone OS 18_0 like Mac OS X)", "country": "GB", "source": "qr"}'
```

Returns the `targetUrl` and `matchedRule`; records nothing. MCP:
`test_redirect_rules`. Check each branch plus the fallback.

Visits per rule: `byRule` in the link's visits (below).

## Visits (clicks and QR scans)

Every visit is counted by source: QR scans, direct clicks, and bots
(link-preview fetchers like Slackbot or WhatsApp, kept out of human counts).

```sh
curl -s "https://esyurl.fyi/v1/links/$ID/visits?days=30&recent=50" -H "Authorization: Bearer $KEY"
```

MCP: `get_short_link_visits`. The response has:

- `totals`: `{visits, qr, direct, bot}`. `visits` is humans only (qr + direct).
- `daily`: one row per UTC day.
- `recent`: the latest visit events, newest first.
- `byRule`: visits each redirect rule decided, and `fallback` for the rest.

Find a link's `linkId` with `GET /v1/links` (MCP `list_short_links`); each link
also carries `visitCount` and `lastVisitedAt`.

### Visits for a group

`GET /v1/groups/{id}/visits?days=30` sums the group's current links. MCP:
`get_group_visits`.

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

1. `PUT /v1/domain` `{"domain": "go.acme.com"}` (MCP `set_custom_domain`).
   One domain per account. Invalid or taken domains fail with 400/409 before
   any charge. Otherwise the answer is 402 with the payment requirements
   (body and `PAYMENT-REQUIRED` header).
2. With approval, sign the payment with an x402 client and repeat the PUT
   with the base64 payload in the `PAYMENT-SIGNATURE` header. MCP: pass it in
   `params._meta["x402/payment"]`, or the `payment` argument. 200 means set
   up; 402 again means settlement failed and nothing was set up.
3. `GET /v1/domain` (MCP `get_custom_domain`) returns the status and the exact
   DNS records to create. **Each call advances provisioning**, so call it
   again after DNS changes.

Status goes `pending_validation` (create the validation CNAME) →
`provisioning` → `pending_dns` (CNAME the domain to the given target) →
`active`. Give the user the records exactly as returned; DNS can take minutes
to hours.

Remove with `DELETE /v1/domain` (MCP `remove_custom_domain`).

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
