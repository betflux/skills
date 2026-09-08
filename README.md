# BetFlux Skills

An [Agent Skill](https://agentskills.io) for querying [BetFlux](https://betflux.ai) —
normalized US sportsbook odds (NFL, NBA, MLB, NHL, NCAAM) served as per-game
Parquet files.

The skill teaches an agent our cost model, which is easy to get wrong: **quota
meters rows downloaded, filters run locally, and one `sportsbook-lines` game is
~190k rows.** Without it, an agent will happily spend a month's quota answering
one question.

## Installing

Works with any agent that supports the Agent Skills standard — Claude Code,
Codex, Cursor, OpenCode, Copilot, Gemini CLI, and others.

### From the betflux CLI

If you already have the CLI, it ships the skill:

```bash
pip install betflux        # or: uv tool install betflux
betflux skill install      # --codex, --project also available
```

### Claude Code

```
/plugin marketplace add betflux/skills
/plugin install betflux@betflux
```

### Clone / copy

Copy `skills/betflux/` into your agent's skill directory:

| Agent | Skill directory |
|---|---|
| Claude Code | `~/.claude/skills/` |
| Cursor | `~/.cursor/skills/` |
| OpenAI Codex | `~/.agents/skills/` |
| OpenCode | `~/.config/opencode/skills/` |

## Skills

| Skill | Useful for |
|---|---|
| `betflux` | Querying closing lines, graded market results, full line history, and game state timelines; cost-aware query planning |

## Getting a key

The skill needs `BETFLUX_API_KEY` set in the environment. Mint one at
<https://betflux.ai/account/api-keys>. An entitled account is required.

### ChatGPT and Codex plugins

The package includes `.codex-plugin/plugin.json` for OpenAI and a portable
Agent Plugins 1.0 manifest at `plugin.json`, alongside Claude and Cursor
manifests. All formats share `skills/betflux/`.

This is a skills-only plugin: it requires a shell/Python environment with
network access and `BETFLUX_API_KEY` supplied outside the conversation. It
does not add an MCP server or a hosted connection to the API. Hosts without
those capabilities can use the skill to prepare commands for you to run.

Publishing this GitHub repository does not list it in a vendor's public
directory. OpenAI supports skills-only submissions through its
[plugin submission portal](https://developers.openai.com/plugins/deploy/submission).
For persistent authenticated ChatGPT access, OpenAI's
[migration guidance](https://developers.openai.com/plugins/guides/submit-claude-plugin)
requires a remote MCP integration; that service is not included in this package.
Claude Code marketplace installation above uses this repository directly.

## Source

This directory is generated from the private `bet-data` monorepo and mirrored
here on release. File issues at <https://betflux.ai/developer> or in the
BetFlux Discord; pull requests against this repository are not merged upstream.

Docs: <https://betflux.ai/developer>
