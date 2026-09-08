---
name: betflux
description: Discover and query BetFlux sportsbook datasets through its CLI, Python SDK, or HTTP API. Use for BetFlux data access, historical odds analysis, closing line value, and quota-aware download planning.
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

## The cost model — read this before any query

This is the part that is easy to get wrong, and getting it wrong spends the
user's money.

1. **Quota meters rows *downloaded*, not rows returned to you.**
2. **The artifact endpoint never filters rows.** It streams one whole Parquet file per game.
   Every `--operator`, `--market-type`, `--team`, `--side`, `--player-id`,
   `--outcome`, `--field`, `--source` filter is evaluated **locally, after
   download**. These row filters do not reduce an individual file's quota cost.
   On range queries, `--team` and `--league` also narrow game discovery, reducing
   the number of files downloaded.
3. **The only lever that reduces spend is fetching fewer games** — a narrower
   `--date-from`/`--date-to`, discovery by team/league, or a single `--game`.
4. `--limit N` stops *fetching* once N rows have been yielded on a range query,
   but **does not cap downloaded rows or quota**. Each file is billed in full;
   if local filters match nothing, every game in the range can be downloaded.
   It truncates arbitrarily, mid-league and mid-date. Use it to sample output,
   not to answer a question about a full period or enforce a spending limit.
5. A partial read is a full read: HTTP Range requests debit the artifact's
   entire row count.

Practical rules:

- Start with `betflux keys check` so you know the tier, rows used, and reset date.
- Start narrow — one day, or one game — confirm the shape, then widen.
- Before a range query, estimate: *games in range × rows per game* (see the
  table below). If that is a large fraction of the user's remaining quota,
  **say so and ask** before running it.
- Never fetch `sportsbook-lines` across a date range. It is ~190k rows for a
  single game; the CLI restricts it to `--game` for exactly this reason.

## Workflow

```bash
betflux keys check                                     # tier + remaining quota
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
`betflux games`, which is a cheap JSON call that does not debit row quota.

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
| 402 | no active subscription for this key | user must subscribe; do not retry |
| 403 | key revoked or suspended | user must mint a new key; do not retry |
| 429 | rate limit, row quota, or ops throttle | the client already retries with backoff honoring `Retry-After`; on quota, **stop** |
| 404 | unknown game, dataset, or league not covered | verify with `betflux games` / `betflux datasets` |

Errors are RFC 9457 problem+json; the `type` URI distinguishes the three 429
causes. Do not paper over a 402 or 403 by retrying — they are terminal until
the user acts.

## Tiers

Rate limits and a monthly row quota apply per tier. `betflux keys check` and
`GET /v1/me` report the live numbers — read them rather than assuming.

**DEMO is pinned to a fixed sample month.** A date range outside that month
returns nothing useful, and DEMO cannot mint keys of its own. If a demo user
asks for last week's games, explain the limitation instead of retrying.

## Python

```python
from betflux import Client

with Client() as bf:
    games = bf.games(league="NBA", date_from="2026-04-01", date_to="2026-04-07")
    rows = bf.closing_lines.game(games[0]["id"], operator="FANDUEL")
    df = bf.closing_lines.df(league="NBA", date_from="2026-04-01", date_to="2026-04-07")
```

The same cost model applies — `df()` over a wide range downloads every game in
it. Details in `references/python.md`.

## Reference index

Read these only when the task needs them; they cost nothing until you do.

- **[references/datasets/closing-lines.md](references/datasets/closing-lines.md)** — columns, filters, coverage
- **[references/datasets/market-results.md](references/datasets/market-results.md)** — graded outcomes and settlement columns
- **[references/datasets/sportsbook-lines.md](references/datasets/sportsbook-lines.md)** — full line history; the expensive one
- **[references/datasets/game-state-timeline.md](references/datasets/game-state-timeline.md)** — observation rows, field/source values
- **[references/cli.md](references/cli.md)** — every command and flag
- **[references/python.md](references/python.md)** — `Client`, dataset handles, typed errors, DuckDB
- **[references/http-api.md](references/http-api.md)** — raw endpoints, for non-Python callers
- **[references/recipes.md](references/recipes.md)** — worked examples: CLV, line movement, backtest frames

Docs: <https://betflux.ai/developer>
