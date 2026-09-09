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

with Client() as bf:          # reads BETFLUX_API_KEY / BETFLUX_BASE_URL
    ...
```

`Client(api_key=..., base_url=..., timeout=30.0, user_agent=...)` overrides the
environment. Always use it as a context manager, or call `.close()`.

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

### Datasets

Handles hang off the client — `bf.closing_lines`, `bf.market_results`,
`bf.sportsbook_lines`, `bf.game_state_timeline` — or `bf.dataset("closing-lines")`
by public name.

| Method | Returns |
|---|---|
| `.game(game_id, **filters)` | list of row dicts for one game |
| `.query_game(game_id, **filters)` | `(rows, hint)` — hint explains a zero-row result |
| `.iter(**kwargs)` | streaming iterator over a range |
| `.rows(**kwargs)` | list over a range |
| `.df(**kwargs)` | pandas DataFrame (needs the `pandas` extra) |
| `.raw(game_id)` | the Parquet bytes, verbatim |

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
pyarrow decodes the Parquet types — **not** ISO strings. Plan comparisons
accordingly.

One exception: `game-state-timeline`'s `ts` is a plain **int64 of epoch
milliseconds (UTC)**, not a native timestamp. Convert it yourself.

## Filters

Passed as keyword arguments; evaluated locally after download. A filter naming
a column the dataset lacks raises `ValueError` **before** fetching. `team` matches home or away; `player_id` matches the
market/selection player-id list columns.

`query_game()` returns a hint alongside the rows when a filter excluded
everything — it names the offending filter and sample values it did see. Use it
before concluding that data is missing.

## Errors

Common errors subclass `BetfluxError`; catch that base class for other API failures:

| Exception | Cause |
|---|---|
| `AuthError` | 401 — key missing, malformed, unrecognized |
| `ForbiddenError` | 403 — key revoked or suspended |
| `NotFoundError` | 404 — unknown game/dataset, or league not covered |
| `RateLimitError` | 429 — per-minute limit |
| `NetworkError` | transport failure |
| `APIError` | other non-2xx |

Rate limits, 5xx, and transport failures retry automatically with backoff that
honors `Retry-After` (capped). Access errors are
terminal until their cause is resolved — do not retry them.

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
