---
name: esyurl-short-links
description: "Shorten URLs and manage short links with esyURL (https://esyurl.fyi): create a short link with a custom slug, change where a link already shared points, pause or delete links, tag them and group them into campaigns. Use when the user wants to shorten a URL, make a short or branded link, fix a link that was already printed or sent, or organise links. The agent can sign itself up; no human account is needed."
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
pass `groupIds` when creating links. `GET /v1/groups/{id}/visits?days=30`
sums visits over the group. MCP: `create_group`, `add_links_to_group`,
`get_group_visits`.

## Related

QR codes for these links: the `esyurl-qr-codes` skill. Different destinations
per device or country: `esyurl-redirect-rules`. Click and scan stats:
`esyurl-link-analytics`. Your own domain: `esyurl-custom-domains`.

Self-signed-up accounts hold up to 100 links (403 `quota_exceeded` beyond).

## Reference

The live documentation wins over anything here:
https://esyurl.fyi/llms.txt (API, MCP tools, limits) and
https://esyurl.fyi/openapi.json (full schema). Errors are JSON:
`{"error": {"code", "message", "details"?}}`; retry 5xx with backoff, don't
repeat a 4xx unchanged.
