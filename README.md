# BetFlux Skills

An [Agent Skill](https://agentskills.io) for querying [BetFlux](https://betflux.ai) —
normalized US sportsbook odds (NFL, NBA, MLB, NHL, NCAAM) served as per-game
Parquet files.

The skill teaches an agent how to discover games, choose datasets, apply
filters, and work with Parquet files through the CLI, Python SDK, or HTTP API.

## Installing

Works with any agent that supports the Agent Skills standard — Claude Code,
Codex, Cursor, OpenCode, Copilot, Gemini CLI, and others.

### From the betflux CLI

Install the BetFlux CLI, then run the command for each agent you use:

```bash
pip install betflux        # or: uv tool install betflux
betflux plugin install --claude
betflux plugin install --codex
# Cursor setup instructions: betflux plugin install --cursor
```

The wrapper delegates to the installed agent's native plugin manager, which
fetches this repository and owns updates and removal. It requires network
access but no BetFlux API key for installation. Rerun the command to refresh
the repository, update the plugin, and enable it. Specify one agent per invocation;
there is no implicit default. You can install in both Claude Code and Codex by
running both commands.

You can also install directly through your agent using the commands below.

### Claude Code

```
/plugin marketplace add betflux/skills
/plugin install betflux@betflux
```

### Codex

Run these commands in your terminal with Codex CLI installed:

```bash
codex plugin marketplace add https://github.com/betflux/skills.git
codex plugin add betflux@betflux
```

To update an existing installation, refresh the marketplace and add the plugin
again. This also enables the plugin if it is disabled:

```bash
codex plugin marketplace upgrade betflux
codex plugin add betflux@betflux
```

Start a new Codex session after installing or updating. Use `/plugins` inside
Codex to inspect your installed plugins. See
[OpenAI's plugin documentation](https://learn.chatgpt.com/docs/plugins).

### Cursor

`betflux plugin install --cursor` reports manual setup instructions and exits
nonzero; it does not pretend to install the plugin. For local testing, clone
this repository into a new `~/.cursor/plugins/local/betflux` directory, reload
Cursor, and confirm the skill in Customize. Do not overwrite an existing
directory. Managed workspaces may require Allow Local Plugin Imports.
See [Cursor's plugin documentation](https://cursor.com/docs/plugins).

### Clone / copy (advanced compatibility option)

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
| `betflux` | Querying closing lines, graded market results, full line history, and game state timelines |

## Getting a key

The skill needs `BETFLUX_API_KEY` set in the environment. Mint one at
<https://betflux.ai/account/api-keys>.

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
