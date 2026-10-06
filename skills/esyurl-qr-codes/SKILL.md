---
name: esyurl-qr-codes
description: "Generate QR codes with esyURL (https://esyurl.fyi): trackable QR codes that can be edited after printing, styled QR codes with dots, rounded corners, brand colours, a logo in the middle or a photo behind, as PNG or SVG, plus static QR codes for any text. Use when the user wants a QR code for a menu, flyer, poster, packaging, business card, event or Wi-Fi, wants to add a logo to a QR code, or needs a printed QR code to point somewhere new. The agent can sign itself up; no human account is needed."
---

# QR codes with esyURL

esyURL (https://esyurl.fyi) renders QR codes, plain or styled (dot modules,
rounded corner markers, colours, a logo in the middle, or a photo behind the
code). A QR code of an esyURL short link is **editable after printing**: change
the link's destination and every printed code follows. Scans are counted
apart from clicks.

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

## Pick the right kind

- **Trackable and editable (recommended for anything printed):** make a short
  link, then render its QR code. The image encodes `{shortUrl}?s=qr`.
- **Static, not tracked:** `POST /v1/render` encodes any text (a URL, Wi-Fi
  string, vCard). Nothing is saved; it can't be changed later.

## QR code of a link

```sh
ID=$(curl -s https://esyurl.fyi/v1/links -H "Authorization: Bearer $KEY" -H 'content-type: application/json' \
  -d '{"targetUrl": "https://example.com/menu", "qrStyle": {"modules": "dot", "eyeFrame": "rounded", "eyeBall": "circle", "dark": "#0a1630"}}' | jq -r .linkId)
curl -s "https://esyurl.fyi/v1/links/$ID/qr?format=svg" -H "Authorization: Bearer $KEY" -o menu-qr.svg
curl -s "https://esyurl.fyi/v1/links/$ID/qr?format=png&size=1024" -H "Authorization: Bearer $KEY" -o menu-qr.png
```

MCP: `create_short_link`, then `render_qr_code` with the `linkId`.
Query options: `format=png|svg`, `size=64..2048`, `margin=0..16`,
`dark`/`light` (hex), `ecl=L|M|Q|H`. SVG is best for print.

Save images to a file and give the user the path; don't paste base64.

## Styles

Styled codes are free with a key. Set `qrStyle` on the link (`PATCH` replaces
the whole `qrStyle`), or pass the same fields to `/v1/render`. All optional:

| Field | Values |
| --- | --- |
| `modules` | `square` `rounded` `dot` `row` `column` |
| `eyeFrame`, `eyeBall` | `square` `rounded` `circle` (corner markers) |
| `dark`, `light`, `eyeColor` | hex colours |
| `errorCorrectionLevel` | `L` `M` `Q` `H` |
| `margin` | 0–16 |
| `image` | `{"mode": "logo", "data": "data:image/png;base64,...", "size"?: 0.1–0.3}` or `{"mode": "halftone", "data": ...}` |

- Logos go in the middle (error correction becomes H). Halftone draws the code
  as dots over a photo. png, jpeg, webp or gif, up to 2 MB.
- To change other fields and keep the picture, send back the `imageId` the
  response shows instead of `data`.
- Dark on light, contrast at least 3:1 (400 otherwise). Don't invert colours.

## Static QR code (any text)

```sh
curl -s https://esyurl.fyi/v1/render -H "Authorization: Bearer $KEY" -H 'content-type: application/json' \
  -d '{"data": "WIFI:T:WPA;S:Cafe;P:secret;;", "format": "png", "size": 800, "modules": "rounded"}' -o wifi.png
```

## Without an account

`POST https://esyurl.fyi/x402/qr` takes the `/v1/render` body and costs
0.01 USDC per image, paid with x402 (402 first, repeat with
`PAYMENT-SIGNATURE`). **Only pay with the user's explicit approval.** With a
key, everything above is free, so prefer signing up.

## Related

Different destinations per phone OS from one code (App Store / Google Play):
the `esyurl-redirect-rules` skill. Scan counts: `esyurl-link-analytics`.

## Reference

The live documentation wins over anything here:
https://esyurl.fyi/llms.txt (API, MCP tools, limits) and
https://esyurl.fyi/openapi.json (full schema). Errors are JSON:
`{"error": {"code", "message", "details"?}}`; retry 5xx with backoff, don't
repeat a 4xx unchanged.
