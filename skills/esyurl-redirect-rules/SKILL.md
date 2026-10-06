---
name: esyurl-redirect-rules
description: "Smart redirects with esyURL (https://esyurl.fyi): one short link or QR code that sends iPhone users to the App Store and Android users to Google Play, or routes visitors by country, language, device, browser, referrer, query parameter or time of day. Use when the user wants an app download link or QR code for both stores, geo-targeted or language-specific links, a link that switches destination on a date, or A/B-style routing. The agent can sign itself up; no human account is needed."
---

# Smart redirects with esyURL

An esyURL (https://esyurl.fyi) short link can send different visitors to
different places: iPhones to the App Store and Android to Google Play,
each country to its own store, Portuguese speakers to the Portuguese page, a
campaign page that switches on launch day. One link, one QR code.

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

## How rules work

`rules` on a link is an ordered list. On each visit the first enabled rule
whose `when` matches picks `targetUrl`; if none matches, the link's own
`targetUrl` is the fallback. Rule: `{"id"?, "name"?, "enabled"?: true, "when": <condition>, "targetUrl"}`.
`PATCH` replaces the whole list (`[]` removes all); keep rule `id`s when
editing, since per-rule stats are keyed by them.

## App-store split

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

## Conditions

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

## Test before sharing

```sh
curl -s https://esyurl.fyi/v1/links/$ID/resolve -H "Authorization: Bearer $KEY" -H 'content-type: application/json' \
  -d '{"userAgent": "Mozilla/5.0 (iPhone; CPU iPhone OS 18_0 like Mac OS X)", "country": "GB", "source": "qr"}'
```

Returns the `targetUrl` and `matchedRule`; records nothing. MCP:
`test_redirect_rules`. Check each branch plus the fallback.

Visits per rule: `GET /v1/links/{id}/visits` → `byRule` (see the
`esyurl-link-analytics` skill).

## Reference

The live documentation wins over anything here:
https://esyurl.fyi/llms.txt (API, MCP tools, limits) and
https://esyurl.fyi/openapi.json (full schema). Errors are JSON:
`{"error": {"code", "message", "details"?}}`; retry 5xx with backoff, don't
repeat a 4xx unchanged.
