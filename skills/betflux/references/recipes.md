# Recipes

Worked examples. See `SKILL.md` for query behavior and supported datasets.

## Closing line value for one team over a week

CLV is **precomputed**. Do not derive it from `opening_odds` / `closing_odds`
yourself:

- `clv_odds` = `closing_odds − opening_odds` (American odds points)
- `clv_implied` = `opening_implied − closing_implied` (implied probability,
  vig included)

Both are null when either end is missing.

```bash
betflux get closing-lines \
  --league NBA --date-from 2026-04-01 --date-to 2026-04-07 \
  --team "Golden State Warriors" \
  --columns game_date,operator,market_key,side,opening_odds,closing_odds,clv_odds \
  --format csv > gsw-clv.csv
```

## Line movement within a single game

`sportsbook-lines` is the full change-only history — ~190k rows for one game.
It is `--game`-only. Save the Parquet rather than rendering it:

```bash
betflux get sportsbook-lines --game NBA_GSW_MIA_20260401 --output movement.parquet
```

Then query locally:

```python
import pyarrow.parquet as pq
import pyarrow.compute as pc

t = pq.read_table("movement.parquet")
ml = t.filter(pc.equal(t["market_type"], "MONEYLINE"))
```

For cross-book fair value, `sportsbook-lines` carries devigged columns from
the supported de-vig methods — use those instead of removing vig yourself.
Read the generated column reference; the ratio method has been removed.

## Did these bets win?

```bash
betflux get market-results \
  --league MLB --date-from 2026-07-01 --date-to 2026-07-07 \
  --outcome WON \
  --columns game_date,market_key,side,outcome,settled_value
```

`outcome` is `WON`, `LOST`, `PUSH`, or `INDETERMINATE`. `INDETERMINATE` means
no trusted source could settle that exact question — it is not a loss, and
dropping those rows silently will bias any hit-rate you compute.

`market-results` includes NCAAM and NCAAF with beta grading limits.
Box-score facts are available, but play-by-play is absent. Missing facts and unsupported
questions remain `INDETERMINATE`; college overtime and operator rules are not
comprehensively verified.

## Model-ready frames

A supervised setup wants three aligned pieces: features, binary outcomes, and
the odds you would have gotten. `market-results` carries the outcomes and the
closing price in one row, so it is the natural starting point:

```python
from betflux import Client

with Client() as bf:
    rows = bf.market_results.rows(
        league="NBA", date_from="2026-04-01", date_to="2026-04-01",
    )

# Require a real pre-game close. The timestamp check also protects reads of
# historical v2 artifacts while the v3 rebuild and serving cutover complete.
priced_rows = [
    row for row in rows
    if row["closing_odds"] is not None
    and row["closing_ts"] is not None
    and row["closing_ts"] < row["game_start"]
]
```

- `Y` — `outcome` mapped to `{WON: 1, LOST: 0}`, `PUSH`/`INDETERMINATE` dropped
  or held out deliberately
- `O` — `closing_odds` (or `opening_odds` if you are modelling the open)
- `X` — join `game-state-timeline` per game for in-play state, or bring your own
  features

Live-only selections remain graded, but their v3 `closing_odds`,
`closing_implied`, and `closing_ts` are null. Use `priced_rows` for
closing-price ROI or model evaluation; do not treat a missing close as zero
odds or drop it from outcome-only grading summaries.

Fetch one day first and confirm the join before widening the range. A full
season across all leagues is a large download.

## Scoreboard state over time

```bash
betflux get game-state-timeline --game NBA_GSW_MIA_20260401 --field home_score
```

Flat observation rows: `ts` / `field` / `value` / `source`. `ts` is an **int64
of epoch milliseconds (UTC)** — the one timestamp column that is not a native
Parquet timestamp. Multiple sources report the same field; `--source` picks one.
