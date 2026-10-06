---
name: esyurl-link-analytics
description: "Click and QR scan analytics for esyURL short links (https://esyurl.fyi): visits per link or per campaign group, split into QR scans, direct clicks and bots, daily trends, recent visits and per-redirect-rule counts. Use when the user asks how many people clicked a link or scanned a QR code, wants to compare campaigns, or wants a traffic report for their short links. The agent can sign itself up; no human account is needed."
---

# Link analytics with esyURL

esyURL (https://esyurl.fyi) counts every visit to a short link and splits it
by source: QR scans, direct clicks, and bots (link-preview fetchers like
Slackbot or WhatsApp, kept out of the human counts).

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

## Visits for a link

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

## Visits for a campaign

`GET /v1/groups/{id}/visits?days=30` sums the group's current links. MCP:
`get_group_visits`.

## Reading the numbers

- QR scans are counted only for codes rendered by esyURL (they encode
  `?s=qr`). A QR code of the raw destination URL can't be counted.
- HEAD requests aren't visits. `curl` and HTTP libraries count as direct.
- Individual events are kept 90 days; daily counts forever.
- When summarising, give totals, the QR vs direct split, the busiest days and
  the trend; mention bots separately.

## Related

Making links: `esyurl-short-links`. QR codes that count scans:
`esyurl-qr-codes`. Per-rule routing: `esyurl-redirect-rules`.

## Reference

The live documentation wins over anything here:
https://esyurl.fyi/llms.txt (API, MCP tools, limits) and
https://esyurl.fyi/openapi.json (full schema). Errors are JSON:
`{"error": {"code", "message", "details"?}}`; retry 5xx with backoff, don't
repeat a 4xx unchanged.
