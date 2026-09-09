# betflux CLI reference

Every command and flag. For query behavior and planning, see
`SKILL.md` — this file is the surface, not the strategy.

## Global options

These go **before** the command: `betflux --timeout 5 games`, not
`betflux games --timeout 5`.

| Option | Env | Default | Notes |
|---|---|---|---|
| `--api-key` | `BETFLUX_API_KEY` | — | Prefer the env var or `--api-key-file`; a key on the command line is visible in `ps` and shell history |
| `--api-key-file PATH` | — | — | Reads the first line of the file |
| `--base-url` | `BETFLUX_BASE_URL` | `https://api.betflux.ai` | |
| `--timeout` | — | `30.0` | Per-request network timeout, seconds |
| `--version` | — | — | Reads installed distribution metadata |

## Commands

### `betflux keys check`

Validates the key and reports the API's account metadata.

### `betflux datasets [--format]`

Lists datasets with title, leagues, column count, and `history_sensitive`. No
auth needed. The table view flattens the nested column list; `--format json`
or `record` keeps the full detail.

### `betflux games`

Lists games and their public ids as JSON reference data.

| Option | Notes |
|---|---|
| `--league` | e.g. `NBA` |
| `--date-from`, `--date-to` | Inclusive ET game date, `YYYY-MM-DD` |
| `--team` | Canonical team name; matches home or away |
| `--status` | `SCHEDULED`, `STARTED`, `COMPLETED` |
| `--id` | Fetch one game by public id or UUID instead of listing |
| `--format`, `--columns` | Output control |

`--id` is exclusive: pairing it with any filter is an error.

### `betflux leagues` / `betflux teams` / `betflux players`

Reference data. `teams` takes `--league`; `players` takes `--league` and/or
`--team-id`. All support `--format` and `--columns`.

### `betflux get <dataset>`

Downloads dataset files and returns matching rows.

**Addressing** — exactly one of:

| Form | Datasets |
|---|---|
| `--date-from` + `--date-to` (+ optional `--league`) | `closing-lines`, `market-results` |
| `--game <id>` | all four |

`sportsbook-lines` and `game-state-timeline` are `--game`-only.

**Filters** (rows are filtered locally after download; team and league also
narrow game discovery on range queries, reducing downloads):

| Option | Applies to |
|---|---|
| `--operator` | the three lines datasets |
| `--market-type` | the three lines datasets |
| `--team` | the three lines datasets; matches home or away |
| `--side` | the three lines datasets, e.g. `HOME`, `OVER` |
| `--player-id` | `closing-lines`, `sportsbook-lines` |
| `--outcome` | `market-results` only: `WON`, `LOST`, `PUSH`, `INDETERMINATE` |
| `--field` | `game-state-timeline` only, e.g. `home_score` |
| `--source` | `game-state-timeline` only, e.g. `draftkings` |

A filter naming a column the dataset lacks is rejected before any download.

**Bounding and output**

| Option | Notes |
|---|---|
| `--limit N` | Range queries only: stop after N matching rows. Does not cap downloaded rows; sparse filters can download the entire range |
| `--output PATH` | With `--game`: save the raw Parquet file, no parsing. Prints one summary line |
| `--columns a,b,c` | Override the default projection |
| `--wide` | Table only: every column instead of the curated subset |
| `--format` | `table` (default), `record`, `json`, `jsonl`, `csv` |
| `--no-progress` | Also `BETFLUX_NO_PROGRESS=1`; auto-disabled when stderr is not a terminal |

`jsonl` and `csv` stream rows as each game's file arrives. `record` prints
vertical `key: value` blocks — the readable choice for a single wide row.

## Output notes

- Default table shows a curated column subset; the gold datasets are wide.
- Timestamp columns are real Parquet timestamps and render as ISO 8601 in every
  format.
- Progress renders on stderr, so `--format json > out.json` stays clean.

## Exit behavior

Failures print `Error: …` to stderr and exit 1. `BETFLUX_DEBUG=1` adds a
traceback. Ctrl-C exits 130.
