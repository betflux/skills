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

Credentials are resolved in order: `--api-key`, `BETFLUX_API_KEY`,
`--api-key-file`, then saved login for the selected API URL.

## Commands

### `betflux login`

Opens the browser and prints a URL and code. The user
signs in, enters the code, and approves access. Saves a new installation key in
the system keychain for CLI commands and Python `Client()`. Existing saved login
is reused without creating another key. Onboarding and account key limits apply.

- `--no-browser`: print the link without opening a browser.
- `--credential-store keyring|file`: default `keyring`; explicitly select `file`
  for protected local file storage on headless hosts.
- `--name TEXT`: override the default installation name.
- `--auth-url URL`: website origin; required for a custom API URL, or set
  `BETFLUX_AUTH_URL`. Put global `--base-url` before `login`.

### `betflux logout`

Revokes the saved installation key and removes local login. Explicit keys are
unaffected. Failed revocation retains the local key for retry; successful
revocation may take five minutes to propagate through the API edge cache.

`--local-only` clears local login without contacting the server. It can recover
malformed metadata or an unavailable keychain reference, and warns if a keychain
entry may remain. Revoke the old key through the account page separately. Use
the same `--base-url` when resetting a custom API login.

### `betflux plugin install --codex | --claude | --cursor`

Specify one agent per invocation. Run the command separately for each agent
you use; installing in both Claude Code and Codex is supported.
Claude Code and Codex register `betflux/skills` and install through
their native plugin managers; Claude uses user scope. The host CLI must already
be installed. An API key is not needed for plugin installation.

Rerunning refreshes the repository, updates the plugin, and enables it.
`--cursor` prints manual setup
instructions and exits nonzero without installing. The command never copies
the SDK's bundled skill or deletes existing files. Start a new
agent session after installing or updating; use the native manager for removal.

### `betflux agent-guide`

Prints the guide bundled with the installed SDK to stdout, without network
access or installation. This snapshot may differ from an independently
released plugin.

### `betflux keys check`

Validates the key and reports the API's account metadata.

### `betflux datasets [--format]`

Lists datasets with title, leagues, and column count. No
auth needed. The table view flattens the nested column list; `--format json`
or `record` keeps the full detail.

### `betflux games`

Lists games — the id plus the full game record: schedule (`start_time`,
`status`, `season`, `week`, `broadcasters`), both teams, the score
(`home_score`, `away_score`, `current_period`, `current_period_seconds_remaining`,
`period_scores_home/away`), the venue (`venue_name`, `venue_city`,
`venue_latitude/longitude`, `venue_capacity`, `venue_surface`,
`venue_has_roof`) and game-time weather (`weather_type`,
`weather_temperature_f`, `weather_wind_speed_mph`, `weather_precipitation_type`, …).
Every field beyond the teams and schedule is nullable.

| Option | Notes |
|---|---|
| `--league` | e.g. `NBA` |
| `--date-from`, `--date-to` | Inclusive ET game date, `YYYY-MM-DD` |
| `--team` | Canonical team name; matches home or away |
| `--status` | `SCHEDULED`, `STARTED`, `COMPLETED` |
| `--id` | Fetch one game by id instead of listing |
| `--format`, `--columns`, `--wide` | Output control; the default table is a curated subset, `--wide` shows every column |

`--id` is exclusive: pairing it with any filter is an error. For one game's
whole record use `--id <id> --format record`. The record is current state
(latest score, status, roster), not history — see
[reference-data.md](reference-data.md).

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
| `--player-id` | `closing-lines`, `market-results`, `sportsbook-lines` |
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

**Live games.** A `sportsbook-lines` game that has not settled has no final
file; `get` assembles its rows from the live feed and prints
`provisional: <game> is live; …` on stderr. stdout is unchanged, so pipes are
unaffected. Report the provisional status when you pass such rows on. Each run
downloads every live segment again, but each live row is charged once per
account per month, so a rerun is charged only for rows published since; still,
use `live-tail` rather than looping `get`. `--output` on a live game saves the
segments re-encoded by the SDK, not a server file.

### `betflux live-board <game>`

The current board for a live game: one row per selection each operator is
believed to be quoting, with `quote_state` (`live` / `stale_suspect`),
`last_changed_at` and `operator_observed_through` on top of the
`sportsbook-lines` columns. A snapshot, not history. Board reads are metered
at a weight the API sets (free during Beta). Once the game has settled it
exits 0 and points at `betflux get sportsbook-lines --game <game>`.

| Option | Notes |
|---|---|
| `--operator` | One operator's board file instead of every operator's |
| `--output PATH` | Save as Parquet instead of rendering rows |
| `--columns`, `--wide`, `--format` | As `get` |

### `betflux live-tail <game>`

Follows a live game's new lines until Ctrl-C. Each poll lists what is new
since the cursor (a position, not a timestamp; the listing is free) and
downloads the segments it names — the open tail on every poll in which it
grew. Each live row is charged once per account per month, so a poll costs only
the rows it brings in. A first poll with neither `--cursor` nor `--from-now`
backfills the whole history published so far. Stops by itself once the game
settles, pointing at
`betflux get sportsbook-lines --game <game>` for the settled history.

| Option | Notes |
|---|---|
| `--every N` | Seconds between polls, default `5.0`; the API caches live listings for 5 s, so anything faster re-downloads the same bytes |
| `--cursor CURSOR` | Resume from the cursor the previous tail printed when it stopped (e.g. `v5:3:1200`), instead of backfilling |
| `--from-now` | Take the current position from one free listing and follow only rows published after it |
| `--output PATH` | Append new rows to a file instead of printing them; resume with `--cursor` so the file holds the history once |
| `--columns`, `--format` | As `get`, except `--format` defaults to `jsonl` — rows arrive per poll — and `json` is refused (an array per poll is not one document) |

Ctrl-C is the normal way to end it and exits 0, not 130; the line it prints
carries the cursor reached. A live log rebuilt under the cursor (409
`live-cursor-reset`) ends the tail with exit 1 — restart without `--cursor`,
or with `--from-now`; rows already written may be superseded. A dataset
version bump rebuilds every in-flight game's live log at the new version, so
a cursor from the old version ends the tail the same way. A poll whose
listing and segment bodies keep disagreeing prints a `note:` and polls again
with the cursor unchanged.

## Output notes

- Default table shows a curated column subset; the gold datasets are wide.
- Timestamp columns are real Parquet timestamps and render as ISO 8601 in every
  format.
- Progress renders on stderr, so `--format json > out.json` stays clean.

## Exit behavior

Failures print `Error: …` to stderr and exit 1. `BETFLUX_DEBUG=1` adds a
traceback. Ctrl-C exits 130.
