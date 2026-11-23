---
name: betflux
description: Discover and query BetFlux sportsbook and prediction-market (Kalshi, Polymarket) datasets and sports data (game state, plays, stats, injuries) through its CLI, Python SDK, or HTTP API. Use for BetFlux data access, live game state, plays, stats, injuries, historical odds analysis, closing line value, dataset exploration, and checking what a BetFlux key may read and what a pull will cost.
---

# BetFlux

Normalized US sportsbook odds, served as per-game Parquet files. Leagues: NFL,
NBA, MLB, NHL, NCAAM, NCAAF. The audited odds history includes FanDuel, DraftKings, BetMGM,
and Pinnacle; the prediction-market datasets cover the exchanges Kalshi and
Polymarket.

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
   would cost (new or already counted) and whether it fits, and
   downloads nothing. Ask the user before spending more than a quarter of what
   is left. A pull that would not fit aborts by itself unless
   `--allow-partial` is given.
3. On a trial key, choose first-weekend games and tell the user the rest needs
   a plan (`upgrade_url` in `/v1/me`).
4. Prefer games already counted this period, since they are free: `betflux
   usage games` lists them and `betflux games --cost` marks them
   (`free_to_read`). One game answers many questions at no extra cost.
5. Skip `live-board`, `live-tail` and live lines reads when
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

A range query skips, without an error, every game in the range that has no
file for that dataset (not settled yet, or not served for it); only a
single-game `--game` read of such a game fails (404). Do not treat a range
result as complete: compare the games it returned against `betflux games`
for the same range.

A range is discovered through `/v1/games`, which takes at most 366 days per
query (400 `validation-error` beyond that). The CLI and SDK do not split a
longer range: split it into pieces of a year or less yourself, for `games` and
`get` alike.

Start with one game or day to confirm the shape before widening the query.
`sportsbook-lines`, `prediction-market-lines` and `game-state-timeline` support `--game` only.
The sports datasets `game-state`, `plays`, `player-stats`, and `team-stats` use one `--game`;
`injuries` uses one `--league`. They serve current snapshots and complete
revision feeds, with local filters. They exist only where sports data is
enabled: if `betflux datasets` does not list one, its commands fail with a
plain 404 (`No route matches …`), exactly as a path that does not exist. They need betflux 0.7.0 or later (`betflux --version`); an
older CLI rejects them locally even when `betflux datasets` lists them. See
[references/sports-data.md](references/sports-data.md).

Sports data is published game by game (league by league for injuries), each
from its first observation; the start dates are in
[references/coverage.md](references/coverage.md#sports-data-feed-history). A covered
game or league with nothing published yet, usually a game that has not
started, answers `404` with problem type `not-yet-available` ("No validated …
snapshot has been published for this scope"). The request is valid; that scope
has no data yet. Come back after publication — do not retry the same request in
a loop. The exception is a game that started before 2025-01-01 00:00 UTC: it is
listed but never will be published. `live-tail` already waits for a first publication by itself (it prints
a `waiting:` line each interval).

A game that has not settled yet has no final `sportsbook-lines` or
`prediction-market-lines` file. `get`
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
betflux get sportsbook-closing-lines --game NBA_ORL_DET_20260503  # one game first
betflux get sportsbook-closing-lines --league NBA --date-from 2026-05-02 --date-to 2026-05-03 --dry-run
betflux get sportsbook-closing-lines --league NBA --date-from 2026-05-02 --date-to 2026-05-03
betflux live-board MLB_WAS_DET_20260922                # in-progress game: current prices
```

## Datasets

| Dataset | Grain | Rows/game | Addressing |
|---|---|---|---|
| `sportsbook-closing-lines` | one row per (operator, market, selection, side); none for a selection quoted on more than one line pregame | ≈ 12,000 | range **or** `--game` |
| `sportsbook-market-results` | one settled row per offer (`offer_key`), with its closing price | ≈ 12,000 | range **or** `--game` |
| `sportsbook-lines` | every change-only line observation | **~190k** | `--game` only |
| `prediction-market-closing-lines` | one row per (exchange, contract, side) | ≈ 100–2,000 | range **or** `--game` |
| `prediction-market-results` | every contract side + the exchange's settlement | ≈ 100–2,000 | range **or** `--game` |
| `prediction-market-lines` | every change-only contract observation | up to ~200k | `--game` only |
| `game-state-timeline` (beta) | our estimate of the score, period, clock and situation, one row per change, with confidence | ≈ 200–1,500 | `--game` only |
| `game-state` | latest source score, status, clock and period state | one | `--game` only |
| `plays` | latest complete play set | varies | `--game` only |
| `player-stats` | game, player and stable stat key | varies | `--game` only |
| `team-stats` | game, team and stable stat key | varies | `--game` only |
| `injuries` | league and player injury state | league-wide | `--league` only |

A missing or null close is expected, not a data error: from v6 a selection first
seen in play has no `sportsbook-closing-lines` row, and from v8 a book not
observed for the game in the 24 hours before the start has null closing columns
(judged per operator event from v10). Games built before v8 carry that stale
quote as the close. See `closing_odds` in
[references/datasets/sportsbook-closing-lines.md](references/datasets/sportsbook-closing-lines.md).

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

An unsettled game's `sportsbook-lines` and `prediction-market-lines` are served
live: `get` assembles them, `betflux live-board GAME` shows what each operator
is quoting now (`--dataset prediction-market-lines` for the exchanges), and
`betflux live-tail GAME` follows new lines until interrupted. Live rows are
provisional. Live listings are free, and a live game counts once per period
like any other game: the first segment or board read counts it, and every
later poll, repeated `get` or board read of it is free (bytes are still
re-downloaded). Live lines need a plan with `limits.live`; on any other key
they are 403 `plan-excludes`. Keep `--every` at 5 s or more (the API's cache;
faster re-downloads the same bytes), and resume a tail with `--cursor`
(printed when it stops) or start one with `--from-now` — a bare rerun
backfills the whole history.

`sportsbook-market-results` includes NCAAM and NCAAF with beta settlement limits:
box-score facts are available, but play-by-play is absent; missing facts and unsupported
questions remain `INDETERMINATE`. College overtime and operator rules are not
comprehensively verified. `game-state-timeline` does not cover NCAAM or NCAAF.

Prediction-market prices are not book odds. `implied_probability` / `odds` are
the all-in cost of one contract (ask plus taker fee), comparable with a book's
posted price; `mid_fair_*` and `microprice_fair_*` are the exchange's fair
value, comparable with a book's de-vigged fair price, and are never de-vigged.
`prediction-market-results` settles from the exchange's payout, so a
fair-price or 50-50 settlement is `PARTIAL`. Line an exchange up against a
book on `outcome_key` ([Comparing venues](#comparing-venues)). Polymarket prices
(`prediction-market-lines`, `-closing-lines`) start at 2026-10-16; its results
reach back to 2025. Start dates by league:
[references/coverage.md](references/coverage.md).

Per-dataset columns and the exact filter list: `references/datasets/*.md`.

## Comparing venues

`market_key` and `offer_key` never match across venues. To find the same bet at
several books, or at a book and an exchange, use the keys every market dataset
carries (the three `sportsbook-*` and three `prediction-market-*` datasets):

- Equal **`outcome_key`**: the rows pay on exactly the same game result,
  whatever the wording. One book's Over 25.5 and another's 26+ share it, as do
  a book's 3-way first-quarter home pick and Kalshi's "home wins Q1" contract
  bought Yes. Over 25 (a push at 25) keys apart, and so does a 2-way moneyline
  that refunds a tie.
- Equal **`settlement_key`**: the same result *and* the same rules for an
  inactive player, overtime, a tie and a game cut short — the identical
  contract. Null when a venue's rule for a needed case is not on record.
  `terms_status` (`verified`, `inferred`, `unknown`) says how firmly the rules
  are known.
- `outcome_min`, `outcome_max` and `outcome_push` spell the result out
  (`outcome_parity` on odd/even markets).

Group on `outcome_key` to compare prices; use `settlement_key` when the rare
cases matter. The keys are null for first-scorer, most-of, race-to and
method-of-score markets, "either team by n", quarter lines (x.25, x.75), Tie No,
and games built before them (check `dataset_version`); only for those, fall back
to `market_type`, `period`, `side` and the line. The CLI has no key filters:
download the files and join locally ([references/recipes.md](references/recipes.md#the-same-bet-at-every-venue)),
or pass `outcome_key=` in Python.

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
| `--league`, `--operator`, `--market-type`, `--team` | the six lines datasets |
| `--side` | the six lines datasets |
| `--player-id` | the six lines datasets, `player-stats` and `injuries` |
| `--outcome` (`WON`, `LOST`, `PUSH`, `VOID`, `PARTIAL`, `INDETERMINATE`) | `sportsbook-market-results`, `prediction-market-results` |
| `--metric` | `player-stats`, `team-stats`; applies reviewed restatement policy |
| `--team-id` | `player-stats`, `team-stats` and `injuries` |
| `--availability` | `injuries` in `live-tail` |

`--team` matches home **or** away.

## Output

Default is a compact table showing a curated column subset — the gold datasets
are wide.

- `--wide` — every column
- `--columns game_date,operator,side,closing_odds` — pick columns
- `--format record` — vertical `key: value`, good for one wide row
- `--format json | jsonl | csv` — machine formats, full fidelity; `jsonl` and
  `csv` stream as each game's file arrives
- `--output PATH` — save a final odds artifact with `--game`, or a sports data snapshot as `.parquet`/`.csv` with its game or league scope

Prefer `--columns` plus `--format jsonl` when you intend to process rows
yourself; it keeps the transcript small. Timestamps render as ISO 8601.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 401 | key missing, malformed, or unrecognized | check for an explicit credential override; have the user sign in with `betflux login` if no valid saved login exists |
| 402 `subscription-required` | rare and brief: the key's record was cached before the account's plan was known (every account has at least the free sample) | retry once after a few minutes; if it persists, report the response details |
| 403 `key-disabled` | key revoked or suspended | user must resolve account access and replace the revoked key; do not retry |
| 403 `history-restricted` | a trial key asked for a game outside its sample: only games from the first weekend of a month (the first Saturday and the Sunday after it) can be read | do not retry; choose first-weekend games (`betflux games` lists dates) and tell the user other dates need a plan (`upgrade_url`) |
| 403 `plan-excludes` | the plan does not include this feature: live lines (`limits.live`), a league dataset such as `injuries` (`limits.league_datasets`) or the prediction-market datasets (`limits.prediction_markets`); the body names `feature` and `required_tiers` | never retry on this key; skip the feature and tell the user which plans include it (`upgrade_url`) |
| 429 `quota-exhausted` | the period's games allowance is spent and this game would be a new one; the body carries `games_used`, `games_allowance`, `reset_at` and `upgrade_url` | do not retry; keep working with the games already counted (`betflux usage games`), report when it resets and where to upgrade |
| 429 `rate-limited` | too many requests this minute | respect `Retry-After` (the CLI and SDK retry it for you); if the client surfaces an error, report it rather than looping |
| 429 `throttled` | an operations stop on this key | do not retry; report it |
| 400 `validation-error` | a parameter failed validation, e.g. a `/v1/games` date range over 366 days (also from a long range `get`, which lists its games there) or only one of `date_from` / `date_to` | fix the request; split a long range into pieces of a year or less |
| 404 `not-yet-available` | Sports data only: nothing has been published yet for that game or league, though `betflux datasets` lists its league | not a bad request and not a reason to loop: wait for publication (a game that started before 2025-01-01 00:00 UTC never will be) |
| 404 | unknown game, dataset, or league not covered; `No route matches` on a sports data command means sports data is not enabled there | verify with `betflux games` / `betflux datasets` |

Errors are RFC 9457 problem+json; inspect the `type` and `detail` fields.
Do not retry access errors without resolving their cause.

## Python

```python
from betflux import Client

with Client() as bf:
    games = bf.games(league="NBA", date_from="2026-05-02", date_to="2026-05-03")
    rows = bf.sportsbook_closing_lines.game(games[0]["id"], operator="FANDUEL")
    df = bf.sportsbook_closing_lines.df(league="NBA", date_from="2026-05-02", date_to="2026-05-03")
```

`df()` over a wide range downloads the selected games before returning, and
each counts; `bf.sportsbook_closing_lines.preview(league=..., date_from=..., date_to=...)`
says what it would cost first and `bf.usage()` is `/v1/me` typed. Details in
`references/python.md`.

## Reference index

Read these only when the task needs them.

- **[references/coverage.md](references/coverage.md)** — league/operator start dates, seasons, NFL weeks, and source-history limits
- **[references/datasets/sportsbook-closing-lines.md](references/datasets/sportsbook-closing-lines.md)** — columns, filters, coverage
- **[references/datasets/sportsbook-market-results.md](references/datasets/sportsbook-market-results.md)** — settled outcomes and settlement columns
- **[references/datasets/sportsbook-lines.md](references/datasets/sportsbook-lines.md)** — full line history
- **[references/datasets/prediction-market-closing-lines.md](references/datasets/prediction-market-closing-lines.md)** — exchange opens, closes, CLV on the all-in price and the mid
- **[references/datasets/prediction-market-results.md](references/datasets/prediction-market-results.md)** — exchange settlement from the reported payout
- **[references/datasets/prediction-market-lines.md](references/datasets/prediction-market-lines.md)** — full exchange price history: all-in price, book, mid and microprice
- **[references/datasets/game-state-timeline.md](references/datasets/game-state-timeline.md)** — beta: estimated state rows, confidence columns, as-of joins
- **[references/sports-data.md](references/sports-data.md)** — snapshot and revision protocol, CLI and SDK examples
- **[references/datasets/game-state.md](references/datasets/game-state.md)** — source game status and score columns
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
