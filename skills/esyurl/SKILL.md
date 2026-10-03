---
name: esyurl
description: Create and manage short links and QR codes with esyURL (https://esyurl.fyi). Use when the user wants to shorten a URL, make a QR code (plain or styled, with a logo), change where an already-shared link or printed QR code points, send iOS and Android visitors to different app stores, route by country, language, device or time, group links into a campaign, track clicks or QR scans, or put short links on their own domain. The agent can sign itself up; no human account is needed.
---

# esyURL

esyURL is an API for short links. A link has a stable short URL
(`https://esyurl.fyi/{slug}`) that redirects to its `targetUrl`. The
`targetUrl` can be changed at any time, and links already shared and QR codes
already printed keep working. Visits are counted by source: QR scan, direct
or bot.

The API is the product: there is no dashboard. The canonical documentation is
always live and wins over anything here:

- https://esyurl.fyi/llms.txt — the API, MCP tools, QR styles, redirect rules, limits
- https://esyurl.fyi/auth.md — agent sign-up, step by step
- https://esyurl.fyi/openapi.json — the full schema

Fetch `llms.txt` before doing anything beyond the basics below.

## 1. Get an API key

Keys look like `esy_<keyId>_<secret>` and do not expire. Look for one in this
order and use the first you find:

1. The `ESYURL_API_KEY` environment variable.
2. `~/.config/esyurl/credentials.json` (`{"api_key": "...", "identity_assertion": "..."}`).
3. An `esyurl` MCP server already configured in this client: just use its tools.

If there is none, sign up. This creates a new, empty account, so do it
**once** and save the result; signing up again gives a different account
without the user's links.

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
curl -s $BASE/v1/me -H "Authorization: Bearer $KEY"
```

Tell the user an account was created and where the key was saved. Never
print the key in full in chat. If the user already has a key from esyURL, use
theirs instead of signing up.

Errors: 429 `rate_limited` means wait `Retry-After` seconds;
`anonymous_not_enabled` means ask the user for a key. A 401 on a key that used
to work means it was revoked: exchange the saved `identity_assertion` again at
`/oauth2/token` (it lasts 30 days); if that answers `invalid_grant`, sign up
again. Full procedure: https://esyurl.fyi/auth.md.

## 2. Connect

**MCP (preferred when the client supports it).** Streamable HTTP at
`https://esyurl.fyi/mcp`, with the header `Authorization: Bearer <key>`. In
Claude Code:

```sh
claude mcp add --transport http esyurl https://esyurl.fyi/mcp --header "Authorization: Bearer $KEY"
```

Tools: `create_short_link`, `list_short_links`, `get_short_link`,
`update_short_link`, `delete_short_link`, `get_short_link_visits`,
`render_qr_code`, `test_redirect_rules`, group tools (`create_group`,
`add_links_to_group`, `get_group_visits`, ...) and custom domain tools
(`get_custom_domain`, `set_custom_domain`, `remove_custom_domain`).

A newly added MCP server may only load in a new session. Until then, or in
clients without MCP, use REST.

**REST.** Same key, `Authorization: Bearer <key>`, JSON bodies.

| Task | Call |
| --- | --- |
| Shorten | `POST /v1/links` `{"targetUrl": "https://...", "slug"?: "spring-sale", "name"?, "tags"?: [..]}` |
| List | `GET /v1/links?limit=20` (follow `nextCursor`) |
| Change destination | `PATCH /v1/links/{id}` `{"targetUrl": "https://..."}` |
| Pause | `PATCH /v1/links/{id}` `{"status": "paused"}` |
| QR code | `GET /v1/links/{id}/qr?format=svg` (or `png&size=1024`) |
| Stats | `GET /v1/links/{id}/visits?days=30` |
| QR of any text, not tracked | `POST /v1/render` `{"data": "...", "format": "png"}` |

## 3. Things to get right

- `targetUrl` must be an absolute `http(s)` URL. Slugs are 3–64 characters of
  `[a-zA-Z0-9_-]`, unique and **cannot be changed later**; omit the slug for a
  random one. 409 means it is taken.
- QR images from `/v1/links/{id}/qr` encode `{shortUrl}?s=qr`, so scans are
  counted apart from clicks. Prefer them over a QR code of the raw destination:
  the destination can then change after printing.
- Styled QR codes are free with a key: set `qrStyle` on the link, e.g.
  `{"modules": "dot", "eyeFrame": "rounded", "eyeBall": "circle", "dark": "#0a1630"}`,
  or add a logo with `"image": {"mode": "logo", "data": "data:image/png;base64,..."}`.
  Shapes, colours and contrast rules are in llms.txt under "QR styles".
- Redirect rules (`"rules"` on a link) send visitors to different destinations
  by OS, device, browser, country, language, referrer, query parameter or
  time; `targetUrl` stays the fallback. Test with `POST /v1/links/{id}/resolve`
  (or `test_redirect_rules`) before sharing. Syntax and examples are in
  llms.txt under "Redirect rules".
- Save QR images to a file and give the user the path; don't paste base64 into chat.
- Self-signed-up accounts hold up to 100 links (403 `quota_exceeded` beyond).

## 4. Costs

Links, QR codes (styled included), redirect rules, groups and stats are free.
A custom domain (e.g. `go.acme.com`) for a self-signed-up account costs
5.00 USDC on Base, once per domain, paid with x402: the first call answers 402
with the price. **Never pay without the user's explicit approval**, and tell
them the amount first. Procedure: llms.txt under "Pricing".
