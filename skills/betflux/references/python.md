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
bf.games(league="NBA", date_from="2026-04-01", date_to="2026-04-07")
bf.game("NBA_GSW_MIA_20260401")
bf.datasets()
bf.check_key()
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
by public name.

| Method | Returns |
|---|---|
| `.game(game_id, **filters)` | list of row dicts for one game |
| `.query_game(game_id, **filters)` | `(rows, hint)` — hint explains a zero-row result; also `.provisional` |
| `.iter(**kwargs)` | streaming iterator over a range |
| `.rows(**kwargs)` | list over a range |
| `.df(**kwargs)` | pandas DataFrame (needs the `pandas` extra) |
| `.raw(game_id)` | the Parquet bytes — verbatim for a settled game; a live game's are its segments stitched and re-encoded by the SDK |
| `.is_live(game_id)` | `sportsbook_lines` only: is the game being served live (any 404 reads as no) |
| `.live_segments(game_id, cursor)` | `sportsbook_lines` only: one poll step |
| `.live_cursor(game_id)` | `sportsbook_lines` only: the current position, one free listing |
| `.live_board(game_id, operator=None)` | `sportsbook_lines` only: the current board |
| `.live_urls(game_id)` | `sportsbook_lines` only: segment URLs for DuckDB |

```python
for row in bf.closing_lines.iter(
    league="NBA", date_from="2026-04-01", date_to="2026-06-30",
    operator="FANDUEL",     # local filter
    max_rows=10_000,        # matching output rows, not downloaded rows
):
    print(row["market_key"], row["closing_odds"])
```

`df()` over a range downloads the selected games before returning a DataFrame.
`max_rows` is the Python equivalent of `--limit`: it caps matching output rows,
not downloaded rows. Local filters may match no rows in a file. Team and league
filters also narrow game discovery on range queries.

### Types

Timestamp and date columns come back as Python `datetime` / `date` objects —
pyarrow decodes the Parquet types — **not** ISO strings. Compare these values using their native Python types.

One exception: `game-state-timeline`'s `ts` is a plain **int64 of epoch
milliseconds (UTC)**, not a native timestamp. Convert it yourself.

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
| `ForbiddenError` | 403 — key revoked or suspended (`key-disabled`) |
| `NotFoundError` | 404 — unknown game/dataset, or league not covered |
| `GameLiveError` | 404 `not-final` — the game is still live. A `NotFoundError` subclass; the dataset methods handle it for you |
| `LiveCursorResetError` | 409 — the live log was rebuilt under the cursor; start over without one |
| `LiveSegmentBehindError` | 409 on a listed segment URL — behind its listing; the dataset methods re-list for you |
| `LiveInconsistentError` | a live step's listing and bodies kept disagreeing; nothing returned, cursor unmoved — call again |
| `RateLimitError` | 429 — per-minute limit |
| `NetworkError` | transport failure |
| `APIError` | other non-2xx |

Rate limits, 5xx, and transport failures retry automatically with backoff that
honors `Retry-After` (capped). Access errors are
terminal until their cause is resolved — do not retry them.

## Live games

An unsettled game has no final `sportsbook-lines` file. Every method above
handles that without being asked: the client follows the API's live pointer,
downloads the Parquet segments and concatenates them into the same columns the
final file would have. The only difference is that the rows are **provisional**
— `query_game()` reports it as `.provisional`. Every such call downloads every
segment again (listings are free, and each live row is charged once per account
per month, so a repeat is charged only for rows published since), so rather than
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
whole — the open tail on every poll in which it grew — but each row is charged
once per account per month, so a poll costs only the rows it brings in. Polling
faster than the ~5 s cache re-downloads the same bytes. A first poll without a
cursor backfills the whole history.

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
betflux get closing-lines --game NBA_GSW_MIA_20260401 --output closing.parquet
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
