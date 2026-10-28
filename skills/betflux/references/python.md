# betflux Python client

```bash
pip install betflux            # core client + CLI
pip install "betflux[pandas]"  # adds .df()
pip install "betflux[polars]"
```

`pyarrow` is a core dependency, not an extra — dataset payloads are Parquet and
the client parses them locally.

## Client

```python
from betflux import Client

with Client() as bf:          # uses explicit environment credentials or saved login
    ...
```

`Client(api_key=..., base_url=..., timeout=30.0, user_agent=...)` overrides the
environment. Without an explicit key or `BETFLUX_API_KEY`, the client
uses the credential saved by `betflux login` for the selected API URL
(`base_url`, `BETFLUX_BASE_URL`, or the production default). Install the SDK in
your project's Python environment even if the CLI is installed with `uv tool`.
Always use it as a context manager, or call `.close()`.

### Reference data

```python
bf.leagues()
bf.teams(league="NBA")
bf.players(league="NBA", team_id=...)
bf.games(league="NBA", date_from="2026-05-01", date_to="2026-05-07")
bf.game("NBA_ORL_DET_20260503")
bf.datasets()
bf.check_key()
bf.usage()                                   # the key's plan, limits and the period's games
bf.preview_games(["NBA_ORL_DET_20260503"])   # what reading these would cost
bf.counted_games(league="NBA")               # games already counted: free to read again
```

Rows are plain dicts. A game carries its id, schedule (`start_time`, `status`,
`season`, `week`), both teams, the score (`home_score`, `away_score`,
`current_period`, `period_scores_home/away`), venue (`venue_*`) and game-time
weather (`weather_*`); the venue and weather fields are null when unknown
(indoor games, pre-game). Every reference row is current state, not history —
columns and the rule are in [reference-data.md](reference-data.md).

### Datasets

Handles hang off the client — `bf.closing_lines`, `bf.market_results`,
`bf.sportsbook_lines`, `bf.game_state_timeline` — or `bf.dataset("closing-lines")`
by public name. A sports dataset by name is `bf.sports_dataset("game-state")`;
`bf.dataset()` does not accept sports dataset names.

Sports data handles are `bf.game_state`, `bf.plays`, `bf.player_stats`,
`bf.team_stats` and `bf.injuries`. They use the snapshot/revision protocol
below, rather than the settled-artifact methods in this table. Before betflux
0.10.0 the sports data names were `bf.msf_dataset()`, `MsfSnapshot`,
`MsfRevision` and `MsfIntegrityError`; they still work but warn.

| Method | Returns |
|---|---|
| `.game(game_id, **filters)` | list of row dicts for one game |
| `.query_game(game_id, **filters)` | `(rows, hint)` — hint explains a zero-row result; also `.provisional` and `.cost` |
| `.preview(league=, date_from=, date_to=, team=)` or `.preview(games=[...])` | `GamesPreview`: what the pull would cost, nothing downloaded or counted |
| `.iter(**kwargs)` | streaming iterator over a range, or over explicit ids with `games=[...]`; its `.games()` is a per-game cost tally afterwards |
| `.rows(**kwargs)` | list over a range |
| `.df(**kwargs)` | pandas DataFrame (needs the `pandas` extra) |
| `.validate_filters(**filters)` | raises `ValueError` for a filter the dataset lacks, without fetching |
| `.raw(game_id)` | the Parquet bytes — verbatim for a settled game; a live game's are its segments stitched and re-encoded by the SDK |
| `.is_live(game_id)` | `sportsbook_lines` only: is the game being served live (any 404 reads as no) |
| `.live_segments(game_id, cursor)` | `sportsbook_lines` only: one poll step |
| `.live_cursor(game_id)` | `sportsbook_lines` only: the current position, one free listing |
| `.live_board(game_id, operator=None)` | `sportsbook_lines` only: the current board |
| `.live_urls(game_id)` | `sportsbook_lines` only: segment URLs for DuckDB |

```python
for row in bf.closing_lines.iter(
    league="NBA", date_from="2026-05-01", date_to="2026-06-30",
    operator="FANDUEL",     # local filter
    max_rows=10_000,        # matching output rows, not downloaded rows
):
    print(row["market_key"], row["closing_odds"])
```

`df()` over a range downloads the selected games before returning a DataFrame.
`max_rows` is the Python equivalent of `--limit`: it caps matching output rows,
not downloaded games. Local filters may match no rows in a file. Team and league
filters also narrow game discovery on range queries.

### Limits and cost

Usage is metered in distinct games per billing period: a game counts once, on
its first successful read of any of its datasets, and every later read of it
is free. The rules are in `SKILL.md`; the numbers are the key's and come from
`/v1/me` at runtime, so never assume them.

```python
with Client() as bf:
    usage = bf.usage()                      # typed /v1/me; free, never counts a game
    usage.plan.label, usage.limits.games_per_period, usage.limits.live
    usage.games_used, usage.games_remaining, usage.resets_at, usage.upgrade_url

    preview = bf.closing_lines.preview(league="NBA", date_from="2026-05-01", date_to="2026-05-07")
    preview.new, preview.already, preview.exempt, preview.fits, preview.over_by
    if preview.fits:
        rows = bf.closing_lines.rows(games=list(preview.games))   # exactly what was previewed
    else:
        rows = bf.closing_lines.rows(games=preview.fitting())      # the part the allowance covers

    for game in bf.counted_games(league="NBA"):                     # free to read again
        game.game_id, game.first_read_at, game.first_dataset, game.cost   # "counted" | "exempt"
```

- `Usage` (`bf.usage()`): `tier`, `plan` (`label`, `size`), `limits`
  (`rate_limit_per_minute`, `games_per_period` — `None` when uncapped —
  `free_recent_game_days`, `max_keys`, `live`, `league_datasets`, `sample`),
  `period_start`, `resets_at`, `games_used`, `games_allowance`,
  `games_remaining`, `games_exempt`, `rows_used`, `counted_by_league`,
  `upgrade_url`, and `raw` (the body as served). `metered` is `False` on a
  gateway from before games metering, and then every games field is `None`.
  `check_key()` returns the same facts as a plain dict beside its legacy keys.
- `GamesPreview` (`bf.preview_games(ids)`, `Dataset.preview(...)`): `games`
  maps each id to `new` (would count), `already` (counted this period, free),
  `exempt` (within the plan's recent-game window, free), `restricted` (outside
  the free sample, refused) or `invalid`; `new`/`already`/`exempt`/
  `restricted`/`invalid` are the counts; `fits` and `over_by`
  compare the new games with `games_remaining`; `fitting()` is the ids the
  allowance covers, in order. Ids are posted to `/v1/me/games/preview` in
  chunks of at most 5,000 and nothing is counted by asking.
- `CountedGame` (`bf.counted_games(league=, date_from=, date_to=)`): pages
  through `GET /v1/me/games`, newest read first; `cost` is `counted` or
  `exempt`.
- What a read cost, from the `x-betflux-game` and `x-betflux-games-remaining`
  response headers: `GameCost` (`outcome` is `new`, `already` or `exempt`,
  `counted` is the boolean form, `games_remaining`) as `.cost` on `GameRows`
  (`query_game`), `GameTable`, `LiveSegments`, `LiveBoard`, `SportsDataSnapshot` and
  `SportsDataRevision`; on progress events `GameDone.counted` (`True` for `new`,
  `False` for `already` or `exempt`, `None` when the gateway sent no header —
  an uncapped plan or an older gateway) and `GameDone.games_remaining`. After
  a range pull, `iter(...).games()` tallies `new`, `already`, `exempt` and
  `unknown` per distinct game.
- `Dataset.iter(games=[...])` pulls explicit ids instead of discovering a
  range; `_iter_rows` is the only multi-game loop, so `rows()` and `df()`
  take `games=` too. A live game whose segments the plan excludes (403
  `plan-excludes`) is skipped by `iter` like a game without data
  (`GameSkipped(reason="plan-excludes")`, counted in `.games().plan_excluded`)
  rather than aborting the range; a single-game read raises
  `PlanRequiredError`.

On a gateway without the preview or counted-games endpoints `preview_games`
and `counted_games` raise `NotFoundError`; `usage()` raises it on a gateway
without `/v1/me`, where `check_key()` still answers from a `/v1/leagues` probe.

### Sports data snapshots and changes

The four game handles take a public game ID; `bf.injuries` takes `league=`.
`snapshot()` returns rows, metadata, a cursor and the downloaded Arrow schema.
`changes(cursor, ...)` yields a finite sequence of complete revisions through
the first listing's high-water mark. `follow()` polls continuously, and waits
at that edge. A game's snapshot and revision files count the game once per
period like any other read of it (free once it is counted, and `SportsDataSnapshot`
/ `SportsDataRevision` carry `.cost`); `injuries` counts nothing but needs a plan
with `limits.league_datasets`. Each
revision has `metadata`, `rows` and `.cursor`, even when `rows` is empty.

```python
with Client() as bf:
    snapshot = bf.player_stats.snapshot("NFL_IND_KC_20260920")
    for revision in bf.player_stats.changes(snapshot.cursor, "NFL_IND_KC_20260920"):
        print(revision.metadata["revision"], revision.rows)

    injuries = bf.injuries.snapshot(league="NFL")
```

Use `follow(..., from_now=True)` to skip baseline output. With filters, the SDK
still loads the baseline so rows leaving a filter can produce deletes; resuming
a filtered `follow(cursor=...)` downloads the whole snapshot at the cursor's
revision (a full re-download; the game is counted already, so it costs no
game). It first checks the cursor against the dataset and scope and makes one
free changes listing, so a cursor for another stream (`ValueError`), a reset
(409) or an expired cursor (410) fails before any download. `follow()` waits
the `poll_after` the server advises between polls (`every=` overrides), from
the first listing on. `SportsDataIntegrityError` means a downloaded file, hash, row
count, cursor or revision envelope failed validation; do not advance the saved
cursor. An epoch reset (409) or expired cursor (410) requires a new snapshot;
a filtered resume whose cursor snapshot is no longer retained raises the same
410. On a plan without `limits.league_datasets` (a trial key) every
`bf.injuries` read — `snapshot(league=...)`, `follow(league=...)`,
`freshness(league=...)` and the archives — raises `PlanRequiredError` (403
`plan-excludes`); a trial key reads game datasets only for first-weekend games
and gets `ForbiddenError` `history-restricted` for any other date. See
[sports-data.md](sports-data.md) for revision application and the generated dataset pages for
frozen column types.

Each game and league a dataset listed by `bf.datasets()` covers is published
from its first observation ([coverage.md](coverage.md#sports-data-feed-history)). A
covered scope with nothing published yet, usually a game that has not started,
raises `NotFoundError` whose `.type` is
`https://api.betflux.ai/problems/not-yet-available`, from `snapshot()`,
`freshness()`, `changes()` and `archives()`. Wait for publication; do not loop
on it. A game that started before 2025-01-01 00:00 UTC is listed but never
will be published. `follow()` without a cursor instead
waits for the first publication itself — every poll interval, calling
`on_wait(error, seconds)` before each wait — and raises any other 404.

### Types

Timestamp and date columns come back as Python `datetime` / `date` objects —
pyarrow decodes the Parquet types — **not** ISO strings. Compare these values using their native Python types.

## Filters

Passed as keyword arguments; evaluated locally after download. A filter naming
a column the dataset lacks raises `ValueError` **before** fetching. `team` matches home or away; `player_id` matches the
market/selection player-id list columns; `mainline=True` keeps only the operator's primary full-game
spread / moneyline / total (`is_mainline`; pre-flag NULL rows never match).

`query_game()` returns a hint alongside the rows when a filter excluded
everything — it names the offending filter and sample values it did see. Use it
before concluding that data is missing.

## Errors

Common errors subclass `BetfluxError`; catch that base class for other API failures:

| Exception | Cause |
|---|---|
| `AuthError` | 401 — key missing, malformed, unrecognized |
| `PaymentRequiredError` | 402 `subscription-required` — key valid, but account has no active subscription; response includes `upgrade_url` |
| `ForbiddenError` | 403 — key revoked or suspended (`key-disabled`), or a game the plan may not read (`history-restricted`: a trial key asked for a game outside its first-weekend sample; `.extra["upgrade_url"]`) |
| `PlanRequiredError` | 403 `plan-excludes` — a `ForbiddenError` subclass: the plan does not include `.feature` (live lines, league datasets); `.required_tiers` names the plans that do and `.upgrade_url` where to change plan. Never worth retrying on this key |
| `NotFoundError` | 404 — unknown game/dataset, or league not covered; for a sports dataset, `.type` `…/problems/not-yet-available` means nothing is published yet for that game or league |
| `GameLiveError` | 404 `not-final` — the game is still live. A `NotFoundError` subclass; the dataset methods handle it for you |
| `LiveCursorResetError` | 409 — the live log was rebuilt under the cursor; start over without one |
| `LiveSegmentBehindError` | 409 on a listed segment URL — behind its listing; the dataset methods re-list for you |
| `LiveInconsistentError` | a live step's listing and bodies kept disagreeing; nothing returned, cursor unmoved — call again |
| `RateLimitError` | 429 `rate-limited` — per-minute limit; retried for you after `Retry-After` |
| `QuotaExceededError` | 429 `quota-exhausted` — the period's games allowance is spent and `.game_id` would be a new game; games already counted stay readable. `.games_used`, `.games_allowance`, `.reset` (`YYYY-MM-DD`), `.reset_at` (instant) and `.upgrade_url`. Not retried |
| `NetworkError` | transport failure |
| `APIError` | other non-2xx |
| `SportsDataIntegrityError` | a sports data file, hash, row count, cursor or revision envelope failed validation; do not advance the saved cursor |

Rate limits, 5xx, and transport failures retry automatically with backoff that
honors `Retry-After` (capped). Access errors are
terminal until their cause is resolved — do not retry them.

## Live games

An unsettled game has no final `sportsbook-lines` file. Every method above
handles that without being asked: the client follows the API's live pointer,
downloads the Parquet segments and concatenates them into the same columns the
final file would have. The only difference is that the rows are **provisional**
— `query_game()` reports it as `.provisional`. Every such call downloads every
segment again (listings are free, and the game counts once per period on its
first read, so a repeat costs no game but all of the bytes), so rather than
looping `game()` on a live game, poll:

```python
cursor = None                     # or bf.sportsbook_lines.live_cursor(game_id): skip the history
while True:
    step = bf.sportsbook_lines.live_segments(game_id, cursor)
    cursor = step.cursor          # e.g. "v5:0:0" until a first segment exists
    handle(step.rows)             # what is new; also step.table (pyarrow)
    time.sleep(5)                 # listings are edge-cached for 5 s
```

The listing is free and incremental; the segments it names are downloaded
whole — the open tail on every poll in which it grew — and the game counts
once, on the first segment read, so later polls cost no game (`step.cost`
reports `already`). Polling faster than the ~5 s cache re-downloads the same
bytes. A first poll without a cursor backfills the whole history. Live lines
need a plan with `limits.live`: on any other key the segment listing raises
`PlanRequiredError`.

The cursor is an opaque position, not a timestamp: a slower operator's file can
land after a faster one's later-stamped rows, so a timestamp would drop them.
`LiveCursorResetError` means the log was rebuilt under it — start again
without a cursor; `LiveInconsistentError` means the listing and a segment body
kept disagreeing through the SDK's bounded retry — nothing was returned and
the cursor did not move, so call again.

`live_board(game_id, operator=None)` is the current snapshot — one row per
selection believed quotable, adding `quote_state` (`live` / `stale_suspect`),
`last_changed_at` and `operator_observed_through`. `live_urls(game_id)` returns
the segment URLs for DuckDB `read_parquet`, verbatim — `?rows=N` included; a
segment behind that count answers 409 rather than a short file.

## Progress

`betflux.progress` exposes a callback hook for download progress; the CLI uses
it to render its stderr bar. Pass `on_progress=` to `Client`.

## DuckDB

Download one file with the CLI, then use DuckDB locally to reuse it across queries.

```bash
betflux get closing-lines --game NBA_ORL_DET_20260503 --output closing.parquet
```

```sql
SELECT * FROM read_parquet('closing.parquet');
```

Files for different games can be built at different versions of a dataset's
compatibility line: newer versions only add columns or carry a change
documented under `versions` in `betflux datasets`. Read several files with
`union_by_name`, and check `dataset_version` before comparing across games:

```sql
SELECT dataset_version, count(*) FROM read_parquet('closing-*.parquet', union_by_name = true) GROUP BY 1;
```
