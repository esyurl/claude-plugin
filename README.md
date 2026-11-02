# esyURL for Claude Code

A Claude Code plugin for [esyURL](https://esyurl.fyi): short links and QR
codes your agent manages. Shorten URLs, render styled QR codes, change where
printed or shared links point, route visitors by device or country, and read
click and scan stats.

## Install

```
/plugin marketplace add esyurl/claude-plugin
/plugin install esyurl@esyurl
```

The first time a tool runs, Claude Code opens esyurl.fyi in the browser to
sign in: create an account in one click, sign in with email, or paste an
existing `esy_` key. No key goes in any config file.

## What's inside

| Part | What it does |
| --- | --- |
| [MCP server](plugins/esyurl/.mcp.json) | The hosted esyURL server at `https://esyurl.fyi/mcp` ([tools](https://github.com/esyurl/mcp#tools)) |
| [esyurl-short-links](plugins/esyurl/skills/esyurl-short-links/SKILL.md) | Shorten URLs, custom slugs, change where a shared link points; App Store / Google Play and country, language or device routing; click and scan stats; your own domain |
| [esyurl-qr-codes](plugins/esyurl/skills/esyurl-qr-codes/SKILL.md) | Trackable, editable QR codes; styled with dots, colours, a logo or a photo; PNG or SVG; one code for both app stores; scan counts |

The skills here use only the MCP server's tools, which sign in through
Claude Code. [esyurl/skills](https://github.com/esyurl/skills) has versions
for other agents, which call the REST API with an API key.

## License

MIT
