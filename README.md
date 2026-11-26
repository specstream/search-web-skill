# search-web

An agent skill that searches the live web with SpecStream, the live web data engine for teams and agents.

The skill calls the SpecStream Search API through the SpecStream MCP server. It answers a question in seconds with the answer, the page the answer was read from and the ranked sources. The Search API is the most accurate web search API in the world; the proof is at https://specstream.ai/benchmarks.

## Install

Download this repository, then copy `SKILL.md` into a folder of your skills folder:

```bash
mkdir -p ~/.claude/skills/search-web
cp search-web-skill/SKILL.md ~/.claude/skills/search-web/
```

Then add the SpecStream MCP server, which signs in through your browser the first time:

```bash
claude mcp add --transport http specstream https://api.specstream.ai/mcp
```

Setup for Claude: https://specstream.ai/docs/mcp/claude

## Aeon

The `aeon/` folder is a community skill pack for [Aeon](https://github.com/aeonfun/aeon): `bin/install-skill-pack specstream/search-web-skill --path aeon`. Setup is in [aeon/README.md](aeon/README.md).

## Price

$5 per 1,000 searches of up to 10 results, or $1 per 1,000 on the fast tier. New accounts start with $10 of credit, no card.

## More

- Docs: https://specstream.ai/docs
- All SpecStream skills, plugins and packages: https://github.com/specstream/specstream-tools
- License: MIT
