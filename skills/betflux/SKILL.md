---
name: betflux
description: Discover and query BetFlux sportsbook datasets through its CLI, Python SDK, or HTTP API. Use for BetFlux data access, historical odds analysis, closing line value, and dataset exploration.
---

# BetFlux

Normalized US sportsbook odds, served as per-game Parquet files. Leagues: NFL,
NBA, MLB, NHL, NCAAM. The audited odds history includes FanDuel, DraftKings, BetMGM,
and Pinnacle.

## Setup

Requires a shell or Python runtime with network access to the BetFlux API.
If the host cannot run the CLI or securely provide credentials, return commands
for the user to run; installing this skill does not create an MCP connection.

First run `betflux keys check` if the CLI is installed. A saved login may
already be available; do not require an environment variable when it works.

For a new installation, have the user run:

```bash
uv tool install betflux      # or: pip install betflux
betflux login
betflux keys check
```

The user opens the printed URL, signs in, enters the terminal code, and approves
access. The CLI stores the key in the system keychain; CLI commands and Python
`Client()` automatically use it for that API URL. Do not approve access on the
user's behalf. Install `betflux` separately in the project's Python environment
when using the SDK; `uv tool install` only installs the CLI environment.

For headless hosts, the user can choose `betflux login --no-browser
--credential-store file`, then approve in a browser. Existing explicit credentials
(`BETFLUX_API_KEY`, `--api-key-file`, or `Client(api_key=...)`) remain supported
and take precedence over saved login. Raw HTTP callers need an explicitly
configured key; browser login does not export an environment variable.

**Never print, echo, or copy the API key into the conversation.** Let the CLI or
SDK read saved credentials; do not inspect keychain entries or credential files.
For automation, have the user configure `BETFLUX_API_KEY` securely outside chat.
Manual keys are available at <https://betflux.ai/account/api-keys>.

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

A game that has not settled yet has no final `sportsbook-lines` file. `get`
still returns its rows — assembled from the live feed — and prints a
`provisional:` line on stderr. Say so when reporting such rows: the final
build may revise them.

## Historical coverage

Before choosing a historical date range, read
[references/coverage.md](references/coverage.md) for the verified start date,
season/phase, and NFL week **by league and operator**. Dates refer to game dates
in US Eastern. Do not assume an operator covers a league from that league's
first date, or that retained source data guarantees a downloadable artifact.
Discover games and confirm the requested dataset.

## Workflow

```bash
betflux keys check                                     # validate credentials
betflux datasets                                       # what's available
betflux games --league NBA --date-from 2026-04-01 --date-to 2026-04-07
betflux get closing-lines --game NBA_GSW_MIA_20260401  # one game first
betflux get closing-lines --league NBA --date-from 2026-04-01 --date-to 2026-04-07
betflux live-board MLB_WAS_DET_20260922                # in-progress game: current prices
```

## Datasets

| Dataset | Grain | Rows/game | Addressing |
|---|---|---|---|
| `closing-lines` | one row per (operator, market, selection, side) | ≈ 12,000 | range **or** `--game` |
| `market-results` | closing lines + graded outcome | ≈ 12,000 | range **or** `--game` |
| `sportsbook-lines` | every change-only line observation | **~190k** | `--game` only |
| `game-state-timeline` | flat `ts` / `field` / `value` / `source` observations | varies | `--game` only |

Row counts are sizing estimates and vary by league, operators, and market
coverage; local filters do not reduce the downloaded artifact's row count.

An unsettled game's `sportsbook-lines` is served live: `get` assembles it,
`betflux live-board GAME` shows what each operator is quoting now, and
`betflux live-tail GAME` follows new lines until interrupted. Live rows are
provisional. Live listings are free and each live row is charged once per
account per month, however often it is read: a repeated `get` on a live game
is charged only for rows published since, and a `live-tail` poll only for the
rows it brings in (both still re-download bytes). Keep `--every` at 5 s or more
(the API's cache; faster re-downloads the same bytes), and resume a tail with
`--cursor` (printed when it stops) or start one with `--from-now` — a bare
rerun backfills the whole history.

`market-results` includes NCAAM and NCAAF with beta grading limits:
box-score facts are available, but play-by-play is absent; missing facts and unsupported
questions remain `INDETERMINATE`. College overtime and operator rules are not
comprehensively verified. `game-state-timeline` does not cover NCAAM or NCAAF.

Per-dataset columns and the exact filter list: `references/datasets/*.md`.

## Game ids

`LEAGUE_AWAY_HOME_YYYYMMDD` — away team first, date in **US Eastern**, `_2`
suffix for the second game of a doubleheader. Case-insensitive. Use the
game id returned by `betflux games`.

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
| `--player-id` | `closing-lines`, `market-results`, `sportsbook-lines` |
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
| 401 | key missing, malformed, or unrecognized | check for an explicit credential override; have the user sign in with `betflux login` if no valid saved login exists |
| 402 `subscription-required` | key valid, but account has no active subscription | report the response details and `upgrade_url`; retry only after account access is restored |
| 403 `key-disabled` | key revoked or suspended | user must resolve account access and replace the revoked key; do not retry |
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

- **[references/coverage.md](references/coverage.md)** — league/operator start dates, seasons, NFL weeks, and source-history limits
- **[references/datasets/closing-lines.md](references/datasets/closing-lines.md)** — columns, filters, coverage
- **[references/datasets/market-results.md](references/datasets/market-results.md)** — graded outcomes and settlement columns
- **[references/datasets/sportsbook-lines.md](references/datasets/sportsbook-lines.md)** — full line history
- **[references/datasets/game-state-timeline.md](references/datasets/game-state-timeline.md)** — observation rows, field/source values
- **[references/cli.md](references/cli.md)** — every command and flag
- **[references/python.md](references/python.md)** — `Client`, dataset handles, typed errors, DuckDB
- **[references/http-api.md](references/http-api.md)** — raw endpoints, for non-Python callers
- **[references/recipes.md](references/recipes.md)** — worked examples: CLV, line movement, backtest frames

Docs: <https://betflux.ai/docs>
