---
name: betflux
description: Discover and query BetFlux sportsbook and MSF game and injury datasets through its CLI, Python SDK, or HTTP API. Use for BetFlux data access, live game state, plays, stats, injuries, historical odds analysis, closing line value, dataset exploration, and checking what a BetFlux key may read and what a pull will cost.
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

## Limits on this key

Usage is metered in **distinct games per billing period**, never in rows. The
rules are the same for every key; the numbers belong to the key and come from
`/v1/me` at runtime — never assume an allowance, rate or price from a plan
name.

- A game counts once per period, on its first successful read of any of its
  datasets, a partial (Range) read included. Re-reads of a counted game, its
  other datasets, listings, reference data and `injuries` are free.
- Games dated within `limits.free_recent_game_days` are free (`exempt`) on
  plans that have that window.
- At the cap only a *new* game is refused (429 `quota-exhausted`); every game
  already counted stays readable until `usage.resets_at`.
- On a trial key (`limits.sample` is set) only first-weekend games can be read:
  the first Saturday of a month and the Sunday after it. Any other date is
  403 `history-restricted`.
- A feature the plan excludes (live lines when `limits.live` is false, league
  datasets when `limits.league_datasets` is false) is 403 `plan-excludes` and
  is never worth retrying on this key. A rate-limit 429 (`rate-limited`) is
  retried after `Retry-After`.

`betflux usage`, `keys check --json`, `get --dry-run` and `--allow-partial`,
and `games --cost` need betflux 0.9.0 or later (`betflux --version`); an older
CLI rejects them as unknown commands or options. Upgrade it with
`uv tool upgrade betflux` (or `pip install -U betflux`) before relying on them.

Habits:

1. Run `betflux keys check --json` when a task starts and before any
   multi-game job; budget against `usage.games_remaining` (null means no cap).
2. Run `betflux get … --dry-run` before a range pull. It prints what the pull
   would cost (new, already counted, exempt) and whether it fits, and
   downloads nothing. Ask the user before spending more than a quarter of what
   is left. A pull that would not fit aborts by itself unless
   `--allow-partial` is given.
3. On a trial key, choose first-weekend games and tell the user the rest needs
   a plan (`upgrade_url` in `/v1/me`).
4. Prefer games already counted this period, since they are free: `betflux
   usage games` lists them and `betflux games --cost` marks them
   (`free_to_read`). One game answers many questions at no extra cost.
5. Skip `live-board`, `live-tail` and live `sportsbook-lines` reads when
   `limits.live` is false, and `get injuries` when `limits.league_datasets`
   is false.
6. Stop on 429 `quota-exhausted` instead of retrying. Keep working with the
   games already counted, and report `resets_at` and `upgrade_url`.

After `get`, `live-board` and `live-tail` the CLI prints one stderr line,
`games: N new, M already counted · X of Y left · resets YYYY-MM-DD`
(`--quiet` silences it). Report it when the user is budgeting.

## Query behavior

The artifact endpoint streams one whole Parquet file per game; row filters
are evaluated locally after download. On range queries, team and league also
narrow game discovery, reducing the number of files fetched.

`--limit N` stops fetching once N matching rows have been yielded on a range
query. It limits output, not the number of downloaded games: if local filters
match nothing, every game in the range can still be downloaded, and each one
counts. Results may end mid-game or mid-date; do not treat a limited sample as
a complete period.

Start with one game or day to confirm the shape before widening the query.
`sportsbook-lines` and `game-state-timeline` support `--game` only.
MSF `game-state`, `plays`, `player-stats`, and `team-stats` use one `--game`;
`injuries` uses one `--league`. They serve current snapshots and complete
revision feeds, with local filters. They exist only where MSF serving is
enabled: if `betflux datasets` does not list one, its commands fail with a
plain 404 (`No route matches …`), exactly as a path that does not exist. They need betflux 0.7.0 or later (`betflux --version`); an
older CLI rejects them locally even when `betflux datasets` lists them. See
[references/msf.md](references/msf.md).

MSF data is published game by game (league by league for injuries), each
from its first observation; the start dates are in
[references/coverage.md](references/coverage.md#msf-feed-history). A covered
game or league with nothing published yet, usually a game that has not
started, answers `404` with problem type `not-yet-available` ("No validated …
snapshot has been published for this scope"). The request is valid; that scope
has no data yet. Come back after publication — do not retry the same request in
a loop. The exception is a game that started before 2025-01-01 00:00 UTC: it is
listed but never will be published. `live-tail` already waits for a first publication by itself (it prints
a `waiting:` line each interval).

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
betflux keys check --json                              # validate credentials; plan, limits, games left
betflux datasets                                       # what's available
betflux games --league NBA --date-from 2026-05-02 --date-to 2026-05-03 --cost
betflux get closing-lines --game NBA_ORL_DET_20260503  # one game first
betflux get closing-lines --league NBA --date-from 2026-05-02 --date-to 2026-05-03 --dry-run
betflux get closing-lines --league NBA --date-from 2026-05-02 --date-to 2026-05-03
betflux live-board MLB_WAS_DET_20260922                # in-progress game: current prices
```

## Datasets

| Dataset | Grain | Rows/game | Addressing |
|---|---|---|---|
| `closing-lines` | one row per (operator, market, selection, side) | ≈ 12,000 | range **or** `--game` |
| `market-results` | closing lines + graded outcome | ≈ 12,000 | range **or** `--game` |
| `sportsbook-lines` | every change-only line observation | **~190k** | `--game` only |
| `game-state-timeline` (beta) | our estimate of the score, period, clock and situation, one row per change, with confidence | ≈ 200–1,500 | `--game` only |
| `game-state` | latest MSF score, status, clock and period state | one | `--game` only |
| `plays` | latest complete MSF play set | varies | `--game` only |
| `player-stats` | game, player and stable stat key | varies | `--game` only |
| `team-stats` | game, team and stable stat key | varies | `--game` only |
| `injuries` | league and player injury state | league-wide | `--league` only |

Rows per game is download size, not cost. Cost is per game, not per row: a
game counts once per period whichever dataset is read first, a
`sportsbook-lines` read costs the same as a `game-state` read of the same
game, and the other datasets of a counted game are free. Row counts are
sizing estimates and vary by league, operators, and market coverage; local
filters do not reduce the downloaded artifact's row count.

## Reference data

`betflux games`, `leagues`, `teams`, `players` (and `bf.games()` etc.) return
the JSON reference data the datasets key on. A game row is the full game
record — schedule, teams, score, venue, game-time weather — not just an id.
It is **current state**: the latest value, refreshed a few times an hour,
with no earlier values kept. Use it to look a game up, not to reconstruct
what happened: for a game's state over time use `game-state-timeline`, for
its odds history `sportsbook-lines`. Columns and filters are in
[references/reference-data.md](references/reference-data.md).

An unsettled game's `sportsbook-lines` is served live: `get` assembles it,
`betflux live-board GAME` shows what each operator is quoting now, and
`betflux live-tail GAME` follows new lines until interrupted. Live rows are
provisional. Live listings are free, and a live game counts once per period
like any other game: the first segment or board read counts it, and every
later poll, repeated `get` or board read of it is free (bytes are still
re-downloaded). Live lines need a plan with `limits.live`; on any other key
they are 403 `plan-excludes`. Keep `--every` at 5 s or more (the API's cache;
faster re-downloads the same bytes), and resume a tail with `--cursor`
(printed when it stops) or start one with `--from-now` — a bare rerun
backfills the whole history.

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
NBA_ORL_DET_20260503
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
| `--player-id` | `closing-lines`, `market-results`, `sportsbook-lines`, MSF `player-stats` and `injuries` |
| `--outcome` (`WON`, `LOST`, `PUSH`, `INDETERMINATE`) | `market-results` |
| `--metric` | `player-stats`, `team-stats`; applies reviewed restatement policy |
| `--team-id` | MSF `player-stats`, `team-stats` and `injuries` |
| `--availability` | MSF injuries in `live-tail` |

`--team` matches home **or** away.

## Output

Default is a compact table showing a curated column subset — the gold datasets
are wide.

- `--wide` — every column
- `--columns game_date,operator,side,closing_odds` — pick columns
- `--format record` — vertical `key: value`, good for one wide row
- `--format json | jsonl | csv` — machine formats, full fidelity; `jsonl` and
  `csv` stream as each game's file arrives
- `--output PATH` — save a final odds artifact with `--game`, or an MSF snapshot as `.parquet`/`.csv` with its game or league scope

Prefer `--columns` plus `--format jsonl` when you intend to process rows
yourself; it keeps the transcript small. Timestamps render as ISO 8601.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 401 | key missing, malformed, or unrecognized | check for an explicit credential override; have the user sign in with `betflux login` if no valid saved login exists |
| 402 `subscription-required` | key valid, but account has no active subscription | report the response details and `upgrade_url`; retry only after account access is restored |
| 403 `key-disabled` | key revoked or suspended | user must resolve account access and replace the revoked key; do not retry |
| 403 `history-restricted` | a trial key asked for a game outside its sample: only games from the first weekend of a month (the first Saturday and the Sunday after it) can be read | do not retry; choose first-weekend games (`betflux games` lists dates) and tell the user other dates need a plan (`upgrade_url`) |
| 403 `plan-excludes` | the plan does not include this feature: live lines (`limits.live`) or a league dataset such as `injuries` (`limits.league_datasets`); the body names `feature` and `required_tiers` | never retry on this key; skip the feature and tell the user which plans include it (`upgrade_url`) |
| 429 `quota-exhausted` | the period's games allowance is spent and this game would be a new one; the body carries `games_used`, `games_allowance`, `reset_at` and `upgrade_url` | do not retry; keep working with the games already counted (`betflux usage games`), report when it resets and where to upgrade |
| 429 `rate-limited` | too many requests this minute | respect `Retry-After` (the CLI and SDK retry it for you); if the client surfaces an error, report it rather than looping |
| 429 `throttled` | an operations stop on this key | do not retry; report it |
| 404 `not-yet-available` | MSF only: nothing has been published yet for that game or league, though `betflux datasets` lists its league | not a bad request and not a reason to loop: wait for publication (a game that started before 2025-01-01 00:00 UTC never will be) |
| 404 | unknown game, dataset, or league not covered; `No route matches` on an MSF command means MSF serving is not enabled there | verify with `betflux games` / `betflux datasets` |

Errors are RFC 9457 problem+json; inspect the `type` and `detail` fields.
Do not retry access errors without resolving their cause.

## Python

```python
from betflux import Client

with Client() as bf:
    games = bf.games(league="NBA", date_from="2026-05-02", date_to="2026-05-03")
    rows = bf.closing_lines.game(games[0]["id"], operator="FANDUEL")
    df = bf.closing_lines.df(league="NBA", date_from="2026-05-02", date_to="2026-05-03")
```

`df()` over a wide range downloads the selected games before returning, and
each counts; `bf.closing_lines.preview(league=..., date_from=..., date_to=...)`
says what it would cost first and `bf.usage()` is `/v1/me` typed. Details in
`references/python.md`.

## Reference index

Read these only when the task needs them.

- **[references/coverage.md](references/coverage.md)** — league/operator start dates, seasons, NFL weeks, and source-history limits
- **[references/datasets/closing-lines.md](references/datasets/closing-lines.md)** — columns, filters, coverage
- **[references/datasets/market-results.md](references/datasets/market-results.md)** — graded outcomes and settlement columns
- **[references/datasets/sportsbook-lines.md](references/datasets/sportsbook-lines.md)** — full line history
- **[references/datasets/game-state-timeline.md](references/datasets/game-state-timeline.md)** — beta: estimated state rows, confidence columns, as-of joins
- **[references/msf.md](references/msf.md)** — snapshot and revision protocol, CLI and SDK examples
- **[references/datasets/game-state.md](references/datasets/game-state.md)** — MSF game status and score columns
- **[references/datasets/plays.md](references/datasets/plays.md)** — complete play snapshots and append/replace revisions
- **[references/datasets/player-stats.md](references/datasets/player-stats.md)** — player stat keys, values and metric filtering
- **[references/datasets/team-stats.md](references/datasets/team-stats.md)** — team stat keys, values and metric filtering
- **[references/datasets/injuries.md](references/datasets/injuries.md)** — league injury state and availability
- **[references/reference-data.md](references/reference-data.md)** — leagues, teams, players, games: columns, filters, and the current-state rule
- **[references/cli.md](references/cli.md)** — every command and flag, including `keys check --json`, `--dry-run`, `--allow-partial` and `usage games`
- **[references/python.md](references/python.md)** — `Client`, dataset handles, `usage()` / `preview()` / `counted_games()`, typed errors, DuckDB
- **[references/http-api.md](references/http-api.md)** — raw endpoints, the `/v1/me` shape, the preview endpoint, the cost headers and the 429/403 bodies, for non-Python callers
- **[references/recipes.md](references/recipes.md)** — worked examples: CLV, line movement, backtest frames

Docs: <https://betflux.ai/docs>
