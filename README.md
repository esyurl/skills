# esyURL agent skills

Skills that let AI agents create and manage short links and QR codes with
[esyURL](https://esyurl.fyi): shorten URLs, render styled QR codes, change
where shared links point, route visitors by device or country, and read
click and scan stats. The agent can sign itself up; no human account is
needed.

## Install

```sh
npx skills add esyurl/skills                          # all of them
npx skills add esyurl/skills --skill esyurl-qr-codes  # just one
```

Works with Claude Code, Cursor, Codex and the other agents
[skills](https://skills.sh) supports.

## Skills

| Skill | What it does |
| --- | --- |
| [esyurl](skills/esyurl/SKILL.md) | Everything below in one skill |
| [esyurl-short-links](skills/esyurl-short-links/SKILL.md) | Shorten URLs, custom slugs, change where a shared link points, groups |
| [esyurl-qr-codes](skills/esyurl-qr-codes/SKILL.md) | Trackable, editable QR codes; styled with dots, colours, a logo or a photo; PNG or SVG |
| [esyurl-redirect-rules](skills/esyurl-redirect-rules/SKILL.md) | One link or QR code for the App Store and Google Play; route by country, language, device or time |
| [esyurl-link-analytics](skills/esyurl-link-analytics/SKILL.md) | Clicks and QR scans per link or campaign, bots kept apart |
| [esyurl-custom-domains](skills/esyurl-custom-domains/SKILL.md) | Short links on your own domain (go.yourcompany.com) |

## Without a skill

Agents can also start from [esyurl.fyi/llms.txt](https://esyurl.fyi/llms.txt)
and [esyurl.fyi/auth.md](https://esyurl.fyi/auth.md), or connect the MCP
server at `https://esyurl.fyi/mcp`.

## License

MIT
