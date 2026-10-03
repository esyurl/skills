# esyURL agent skills

Skills that let AI agents create and manage short links and QR codes with
[esyURL](https://esyurl.fyi): shorten URLs, render styled QR codes, change
where shared links point, route visitors by device or country, and read
click and scan stats. The agent can sign itself up; no human account is
needed.

## Install

```sh
npx skills add esyurl/skills
```

Works with Claude Code, Cursor, Codex and the other agents
[skills](https://skills.sh) supports.

## Skills

| Skill | What it does |
| --- | --- |
| [esyurl](skills/esyurl/SKILL.md) | Short links, QR codes, redirect rules and visit stats through the esyURL MCP server or REST API |

## Without a skill

Agents can also start from [esyurl.fyi/llms.txt](https://esyurl.fyi/llms.txt)
and [esyurl.fyi/auth.md](https://esyurl.fyi/auth.md), or connect the MCP
server at `https://esyurl.fyi/mcp`.

## License

MIT
