---
name: betflux
description: Discover and query BetFlux sportsbook datasets through its CLI, Python SDK, or HTTP API. Use for BetFlux data access, historical odds analysis, closing line value, and dataset exploration.
---

# BetFlux

Normalized US sportsbook odds, served as per-game Parquet files. Leagues: NFL,
NBA, MLB, NHL, NCAAM. Operators include FanDuel, DraftKings, Caesars, BetMGM,
and Pinnacle.

## Setup

Requires a shell or Python runtime with network access to the BetFlux API.
If the host cannot run the CLI or securely provide credentials, return commands
for the user to run; installing this skill does not create an MCP connection.

```bash
pip install betflux          # or: uv tool install betflux
export BETFLUX_API_KEY=bfx_live_...
betflux keys check
```

Keys are minted at <https://betflux.ai/account/api-keys> and shown once.

**Never print, echo, or write the API key.** Read it from the environment and
leave it there. If it is missing, tell the user to set `BETFLUX_API_KEY` — do
not ask them to paste it into the conversation.

## Query behavior

The artifact endpoint streams one whole Parquet file per game; row filters
are evaluated locally after download. On range queries, team and league also
narrow game discovery, reducing the number of files fetched.

`--limit N` stops fetching once N matching rows have been yielded on a range
query. It limits output, not the number of downloaded rows: if local filters
match nothing, every game in the range can still be downloaded. Results may
end mid-game or mid-date; do not treat a limited sample as a complete period.

Start with one game or day to confirm the shape before widening the query.
`sportsbook-lines` and `game-state-timeline` support `--game` only.

## Workflow

```bash
betflux keys check                                     # validate credentials
betflux datasets                                       # what's available
betflux games --league NBA --date-from 2026-04-01 --date-to 2026-04-07
betflux get closing-lines --game NBA_GSW_MIA_20260401  # one game first
betflux get closing-lines --league NBA --date-from 2026-04-01 --date-to 2026-04-07
```

## Datasets

| Dataset | Grain | Rows/game | Addressing |
|---|---|---|---|
| `closing-lines` | one row per selection, final price | hundreds | range **or** `--game` |
| `market-results` | closing lines + graded outcome | hundreds | range **or** `--game` |
| `sportsbook-lines` | every change-only line observation | **~190k** | `--game` only |
| `game-state-timeline` | flat `ts` / `field` / `value` / `source` observations | thousands | `--game` only |

`market-results` and `game-state-timeline` do not cover NCAAM.

Per-dataset columns and the exact filter list: `references/datasets/*.md`.

## Game ids

`LEAGUE_AWAY_HOME_YYYYMMDD` — away team first, date in **US Eastern**, `_2`
suffix for the second game of a doubleheader. Case-insensitive. Internal UUIDs
work anywhere a game id does.

```
NBA_GSW_MIA_20260401
MLB_BOS_NYY_20260715_2
```

Do not construct these from guessed abbreviations — discover them with
`betflux games`, which returns game metadata as JSON.

## Filters

All local. A filter naming a column the dataset does not have is rejected up
front, before any download.

| Filter | Datasets |
|---|---|
| `--league`, `--operator`, `--market-type`, `--team` | the three lines datasets |
| `--side` | the three lines datasets |
| `--player-id` | `closing-lines`, `sportsbook-lines` |
| `--outcome` (`WON`, `LOST`, `PUSH`, `INDETERMINATE`) | `market-results` |
| `--field`, `--source` | `game-state-timeline` |

`--team` matches home **or** away.

## Output

Default is a compact table showing a curated column subset — the gold datasets
are wide.

- `--wide` — every column
- `--columns game_date,operator,side,closing_odds` — pick columns
- `--format record` — vertical `key: value`, good for one wide row
- `--format json | jsonl | csv` — machine formats, full fidelity; `jsonl` and
  `csv` stream as each game's file arrives
- `--output PATH` (with `--game`) — save the raw Parquet, no parsing

Prefer `--columns` plus `--format jsonl` when you intend to process rows
yourself; it keeps the transcript small. Timestamps generally render as ISO 8601;
`game-state-timeline.ts` is epoch milliseconds and needs explicit conversion.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 401 | key missing, malformed, or unrecognized | check `BETFLUX_API_KEY` is set and unabridged |
| 402 | access unavailable for this key | report the API's error details; do not retry |
| 403 | key revoked or suspended | user must mint a new key; do not retry |
| 429 | request rejected by a service limit | respect `Retry-After`; if the client surfaces an error, report it rather than looping |
| 404 | unknown game, dataset, or league not covered | verify with `betflux games` / `betflux datasets` |

Errors are RFC 9457 problem+json; inspect the `type` and `detail` fields.
Do not retry access errors without resolving their cause.

## Python

```python
from betflux import Client

with Client() as bf:
    games = bf.games(league="NBA", date_from="2026-04-01", date_to="2026-04-07")
    rows = bf.closing_lines.game(games[0]["id"], operator="FANDUEL")
    df = bf.closing_lines.df(league="NBA", date_from="2026-04-01", date_to="2026-04-07")
```

`df()` over a wide range downloads the selected games before returning. Details in `references/python.md`.

## Reference index

Read these only when the task needs them.

- **[references/datasets/closing-lines.md](references/datasets/closing-lines.md)** — columns, filters, coverage
- **[references/datasets/market-results.md](references/datasets/market-results.md)** — graded outcomes and settlement columns
- **[references/datasets/sportsbook-lines.md](references/datasets/sportsbook-lines.md)** — full line history
- **[references/datasets/game-state-timeline.md](references/datasets/game-state-timeline.md)** — observation rows, field/source values
- **[references/cli.md](references/cli.md)** — every command and flag
- **[references/python.md](references/python.md)** — `Client`, dataset handles, typed errors, DuckDB
- **[references/http-api.md](references/http-api.md)** — raw endpoints, for non-Python callers
- **[references/recipes.md](references/recipes.md)** — worked examples: CLV, line movement, backtest frames

Docs: <https://betflux.ai/developer>
