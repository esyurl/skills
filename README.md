# esyURL agent skills

Skills that let AI agents create and manage short links and QR codes with
[esyURL](https://esyurl.fyi): shorten URLs, render styled QR codes, change
where shared links point, route visitors by device or country, and read
click and scan stats. The agent can sign itself up; no human account is
needed.

## Install

```sh
npx skills add esyurl/skills                          # both
npx skills add esyurl/skills --skill esyurl-qr-codes  # just one
```

Works with Claude Code, Cursor, Codex and the other agents
[skills](https://skills.sh) supports.

## Skills

| Skill | What it does |
| --- | --- |
| [esyurl-short-links](skills/esyurl-short-links/SKILL.md) | Shorten URLs, custom slugs, change where a shared link points; App Store / Google Play and country, language or device routing; click and scan stats; your own domain |
| [esyurl-qr-codes](skills/esyurl-qr-codes/SKILL.md) | Trackable, editable QR codes; styled with dots, colours, a logo or a photo; PNG or SVG; one code for both app stores; scan counts |

## Without a skill

Agents can also start from [esyurl.fyi/llms.txt](https://esyurl.fyi/llms.txt)
and [esyurl.fyi/auth.md](https://esyurl.fyi/auth.md), or connect the MCP
server at `https://esyurl.fyi/mcp`.

## License

MIT
