---
name: esyurl-custom-domains
description: "Put short links on your own branded domain with esyURL (https://esyurl.fyi), like go.yourcompany.com/offer: set up the domain, get the DNS records (CNAMEs) to create, follow provisioning to active. Use when the user wants branded short links, a vanity domain for links or QR codes, or asks why their short links don't use their domain. The agent can sign itself up; no human account is needed."
---

# Custom short-link domains with esyURL

esyURL (https://esyurl.fyi) can serve an account's short links on its own
domain, such as `go.acme.com/spring`. Once the domain is active, short URLs
and QR codes use it; the `esyurl.fyi` addresses keep working.

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

## Cost: ask first

For a self-signed-up account, setting up a domain costs **5.00 USDC on Base,
once per domain**, paid with x402. Reading, re-setting the same domain and
removing it are free; switching to another domain is a new payment.
Accounts whose key came from esyURL directly don't pay.

**Tell the user the price and get explicit approval before paying.** If they
have no x402 wallet, stop and explain.

## Set it up

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

## Reference

The live documentation wins over anything here:
https://esyurl.fyi/llms.txt (API, MCP tools, limits) and
https://esyurl.fyi/openapi.json (full schema). Errors are JSON:
`{"error": {"code", "message", "details"?}}`; retry 5xx with backoff, don't
repeat a 4xx unchanged.
