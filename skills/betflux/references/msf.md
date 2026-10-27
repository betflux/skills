# MSF snapshots and revision feeds

`game-state`, `plays`, `player-stats`, and `team-stats` take one public game ID.
`injuries` takes one league. Check `betflux datasets` first: these feeds appear
when serving is enabled for the API environment. The CLI commands need betflux
0.7.0 or later; an older CLI rejects these dataset names locally even when the
listing shows them. The source is MySportsFeeds;
`game-state-timeline` is a separate beta dataset: our estimate of the state
from every source we hold, with confidence.

`betflux datasets` (`GET /v1/datasets`) lists every league a feed covers, and
each covered game and league is published from its first observation; the
dates are in [coverage.md](coverage.md#msf-feed-history). A covered scope with
nothing published yet, usually a game that has not started, answers `404` with
problem type `…/problems/not-yet-available` and the detail "No validated
`<dataset>` snapshot has been published for this scope" — on `get`,
`snapshot()`, `freshness()`, `changes` and archive listings alike. It is
distinct from an unknown game (`404 Game not found`) or an uncovered league
(`404`, reason `league_not_covered`). When you meet it, wait for publication;
do not retry the same request in a loop. A game that started before
2025-01-01 00:00 UTC is the exception: it is listed but never will be
published, so the answer is final.
`live-tail` and `follow()` started without a cursor are the exception by
design: they wait for a first publication themselves (see below).

```sh
betflux get game-state --game NFL_IND_KC_20260920
betflux get player-stats --game NFL_IND_KC_20260920 --metric FOOTBALL_RUSHING_YARDS
betflux get injuries --league NFL --output injuries.parquet
betflux live-tail NFL_IND_KC_20260920 --dataset plays --output plays.jsonl
betflux live-tail --dataset injuries --league NFL --from-now --output injuries.jsonl
```

`get` downloads a current snapshot. The snapshot manifest identifies a frozen
schema version, source observation, stream epoch and revision, coverage, status,
the Parquet files with hashes/counts, and a cursor. Snapshot exports support
Parquet and CSV. Filters run locally after download and do not reduce quota.
In the stat datasets, `stat_key` is stable but opaque for most stats (`msf:`
plus an encoded source name); `source_stat_category` and `source_stat_name`
name the stat. `metric` is set only on stats mapped to a canonical name, about
a quarter to a third of rows today, so `--metric` cannot select the rest: filter
those on `source_stat_name`. One metric can match more than one source stat.
`unit`, `value_kind`, `stat_scope` and `aggregation_role` are null on almost
every row; null means not yet reviewed. `--metric` omits a reviewed
`restatement` row when the matching canonical alternative is present in the
same entity, scope and unit; no row carries that role yet.

Snapshot and changes responses have a separate `freshness` object with nullable
`newest_applied_source_updated_at`, `last_changed_at`,
`latest_request_started_at`, `last_pull_success_observed_at`,
`last_normalized_at`, and
`health_observed_at`. Null means independent health evidence is unavailable,
and today every field is null on every manifest: the evidence is not published
yet, so do not build on these fields.
`source_observation` identifies the source behind the published revision; a
later unchanged observation may have been applied, so its timestamp is not a
newest-applied checkpoint or evidence of a recent completed fetch. Current
snapshot `published_at` is the data head adoption time, preserved through
archive/retention maintenance; a historical snapshot uses its
manifest upload time. The Python SDK exposes `snapshot.freshness` and
`bf.plays.freshness(game)`; the latter fetches only metadata, without Parquet
files.

`live-tail` writes **one complete JSONL object per revision**, including an
empty `rows` list. Start without `--cursor` to emit a baseline; use
`--from-now` to start at the current cursor and emit only later revisions.
`--from-now` with filters still loads the baseline internally so a correction
that removes a row from the filter produces a delete. A managed output file
gets `<output>.cursor.json`; resume with the same scope and filters. Do not
apply a partial trailing JSONL line.

Started without `--cursor` on a scope with nothing published yet (`404`
`not-yet-available`), `live-tail` prints a `waiting:` line on stderr and checks
again every poll interval — 30 s, 300 s for injuries, or `--every` — until a first snapshot appears; any other 404 ends it. Between polls it waits the `poll_after` the server
sends with each listing, a resumed tail included.

Other manifest fields: `is_baseline` is true on a stream's first revision,
whose change files hold every row. `history_origin` is `captured` for a
revision published as the source changed and `reconstructed` for one rebuilt
later from stored source responses. `live_capture_started_at` is when live
capture of the scope began; null means every revision so far is a
reconstruction.

Revision kinds have distinct state transitions:

| Kind | Apply |
|---|---|
| `patch` | Upsert or delete each named row key. Used by game state, stats and injuries. |
| `append` | Add the listed plays to the prior complete play set. |
| `replace` | Clear the prior play set, then insert the listed plays; an empty list clears it. |

The feed's `status` is `verified` once MySportsFeeds reports the game
`COMPLETED` and, for plays and stats, that dataset's snapshot is complete;
until then, and while MSF holds a final under review, it is `provisional`.
A league has no completed state, so `injuries` is always `provisional`.
Separate datasets do not form an atomic game bundle. Carried rows keep their
original row-level source provenance.

`injuries` mirrors MySportsFeeds' injury report, which usually changes about
once a day. Between reports the head does not move, so a player ruled out after
the last report keeps the earlier status until the next one. The source reports
only `out` and `questionable` today, so a player listed as doubtful appears as
`questionable`. The report is not a complete injury list: it can miss players
other outlets list as out, it sometimes drops a player for one report and
restores the player in the next (a delete, then an upsert), MLB's list empties after
the regular season, and NCAAM's holds only a handful of players.

```python
from betflux import Client

with Client() as bf:
    snapshot = bf.plays.snapshot("NFL_IND_KC_20260920")
    for revision in bf.plays.changes(snapshot.cursor, "NFL_IND_KC_20260920"):
        print(revision.metadata["revision_kind"], revision.rows)

    injuries = bf.injuries.snapshot(league="NFL")
```

The SDK also has `bf.game_state`, `bf.player_stats`, and `bf.team_stats`.
`snapshot().df()` returns a pandas DataFrame, including columns for an empty
snapshot. `follow()` polls continuously; `changes(cursor)` returns the finite
set published through a fixed high-water revision (over HTTP, the first
page's `high_water_revision`, sent as `until_revision` on later pages). For a
key whose history window ends before the newest revision (the demo key's sample
month), the high-water mark is the last revision that key may read, so the
listing still ends and `follow()` then waits there. Read and verify all files
for a revision before saving its cursor. A `409` with problem type
`…/problems/stream-reset` requires a new snapshot and state replacement. A
`410` with problem type `…/problems/cursor-expired` means the prior resume
history is unavailable. A filtered resume, `follow(cursor=..., filters=...)`
or a filtered `live-tail` resuming from `--cursor` or its output checkpoint,
downloads the whole historical snapshot at the cursor's revision, charged at
its full row count, to rebuild which rows pass the filter. Before that download
it checks that the cursor names the requested dataset and scope and makes one
free changes listing from it, so a cursor for another stream (`ValueError`), a
stream reset (`409`) or an expired cursor (`410`) fails without a charge. Once
the cursor's snapshot is outside retained history it fails with the same `410`
cursor-expired error, one revision before an unfiltered resume would. For raw
HTTP paths and file authentication, see [http-api.md](http-api.md).

A demo key cannot read the current injuries head. Its history window is the
demo sample month, May 2026, and every current read of a league dataset is
checked against it: `betflux get injuries`, `betflux live-tail --dataset
injuries` (with or without `--from-now`), and `bf.injuries`
`.snapshot(league=...)`, `.follow(league=...)` and `.freshness(league=...)`
answer `403` with problem type `…/problems/history-restricted`. A demo key
reads injuries only through a historical snapshot,
`bf.injuries.snapshot(league=..., at_revision=..., epoch=...)`, whose source
date falls in that month, or through the archives `.archives(league=...)` lists
for it. Do not retry the `403`; use those reads, or a paid key for current
injuries. Game
datasets follow the usual rule: a demo key reads games dated in its sample
month.

The generated dataset pages list each frozen snapshot column and change
envelope: [game-state](datasets/game-state.md), [plays](datasets/plays.md),
[player-stats](datasets/player-stats.md), [team-stats](datasets/team-stats.md),
and [injuries](datasets/injuries.md).

Archive discovery is explicit and paginated. No archives are published yet:
every listing is empty today, and every published revision can still be read
directly with `at_revision` and `epoch`. Discovery returns only independently
verified blocks; do not silently use an archive to resume an expired cursor.
Select a block and reconstruct its baseline plus changes locally:

```python
with Client() as bf:
    for archive in bf.plays.archives("NFL_IND_KC_20260920"):
        snapshot = archive.snapshot(at_revision=archive.descriptor["last_revision"])
        print(snapshot.metadata["revision"], len(snapshot.rows))
    for archive in bf.injuries.archives(league="NFL"):
        snapshot = archive.snapshot()
```

The SDK verifies every reconstructed state digest and original row provenance.
Metadata remains free; baseline and change downloads charge downloaded rows.
Archive access uses the same historical entitlement as other historical reads.
A league change listing serves the revisions inside the key's history window
and reports the last of them as `high_water_revision` on every page, so paging
to that mark finishes; a listing from a cursor at the edge is an empty `200`.
Only a cursor the key could never read is refused with a `history-restricted`
problem.
A deliberately retired direct snapshot returns `404 revision_not_retained`
with archive discovery; an expired cursor returns `410` with a fresh snapshot
and archive link. Missing files still promised by a retained manifest are errors.
