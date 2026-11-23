# Recipes

Worked examples. See `SKILL.md` for query behavior and supported datasets.

## Closing line value for one team over a week

CLV is **precomputed**. Do not derive it from `opening_odds` / `closing_odds`
yourself:

- `clv_odds` = `closing_odds − opening_odds` in American odds cents, counting
  −100 and +100 as one price (−102 → +100 is 2; before v9 it read 202)
- `clv_implied` = `opening_implied − closing_implied` (implied probability,
  vig included)

Both are null when either end is missing. From v10 the opening is the quote
at the league's opening cutoff (`opening_method`, `opening_cutoff_ts`), not a
look-ahead price posted weeks early; `first_quote_odds` keeps the first quote
ever seen.

```bash
betflux get sportsbook-closing-lines \
  --league NBA --date-from 2026-05-02 --date-to 2026-05-03 \
  --team "Detroit Pistons" \
  --columns game_date,operator,market_key,side,opening_odds,closing_odds,clv_odds \
  --format csv > det-clv.csv
```

## Closing price and result of the same bet

An offer is one selection at one line. FanDuel (main, period and over/under
markets) and Pinnacle (player props) keep one selection id while the line
moves, so one selection id can be several offers. `sportsbook-market-results`
has a row per offer; `sportsbook-closing-lines` has a row only for a
selection quoted on one line before the game started, keyed by that line's
`offer_key`. Join the closing lines, the results and the line history on
`offer_key`, never on `market_key` plus `selection_external_id`, which pairs
a close at one line with a result at another. `market_key` is for
market-level joins.

```sql
SELECT c.operator, c.market_key, c.side, c.closing_spread, c.closing_total,
       c.clv_implied, r.outcome
FROM read_parquet('closing.parquet') c
JOIN read_parquet('results.parquet') r USING (offer_key)
```

Rows built before `sportsbook-closing-lines` v14 and
`sportsbook-market-results` v16 carry no `offer_key`. Up to closing-lines
v14 a moving selection is one row whose opening and close can be on
different lines, and up to market-results v16 it is one result, at its
closing line (see `dataset_version`).

## The same bet at every venue

`outcome_key` lines up quotes that pay on the same result, across books and
exchanges; `market_key` and `offer_key` never match across venues. Download one
game from each venue and join locally:

```bash
betflux get sportsbook-closing-lines --game NBA_ORL_DET_20260503 --output books.parquet
betflux get prediction-market-closing-lines --game NBA_ORL_DET_20260503 --output exchanges.parquet
```

```sql
-- Book against exchange: the book's closing price and the exchange's all-in cost
-- and fair mid on the same result; an equal settlement_key is the identical contract.
SELECT b.operator AS book, b.side, b.closing_odds AS book_odds, b.closing_implied AS book_implied,
       x.operator AS exchange, x.closing_implied AS exchange_cost, x.closing_mid,
       b.settlement_key = x.settlement_key AS identical_contract
FROM read_parquet('books.parquet') b
JOIN read_parquet('exchanges.parquet') x ON b.outcome_key = x.outcome_key;

-- Book against book: every pair of books quoting the same result, however worded.
SELECT a.operator, a.selection_name, a.closing_odds, b.operator, b.selection_name, b.closing_odds
FROM read_parquet('books.parquet') a
JOIN read_parquet('books.parquet') b ON a.outcome_key = b.outcome_key AND a.operator < b.operator;
```

The keys are null for markets without a single range of results (first scorer,
race to, "either team by n", quarter lines, Tie No) and on games built before
them, so those rows drop out of the join; compare them on `market_type`,
`period`, `side` and the line instead.

## Line movement within a single game

`sportsbook-lines` is the full change-only history — ~190k rows for one game.
It is `--game`-only. Save the Parquet rather than rendering it:

```bash
betflux get sportsbook-lines --game NBA_ORL_DET_20260503 --output movement.parquet
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
betflux get sportsbook-market-results \
  --league MLB --date-from 2026-07-01 --date-to 2026-07-07 \
  --columns game_date,offer_key,side,outcome,settled_value,closing_odds,stake_won_fraction,stake_refunded_fraction
```

`outcome` is one of:

- `WON` / `LOST`: settled against the result.
- `PUSH`: the bet had action and the stake is returned (the value landed on a
  whole-number line, or a two-way market ended level).
- `VOID`: the bet never had action and the stake is returned (for example, a
  player who did not take part).
- `PARTIAL`: part of the stake won or was returned (a dead heat, a split
  line); read the stake fractions.
- `INDETERMINATE`: no trusted source could settle that exact question. It is
  not a loss, and dropping those rows silently will bias any hit-rate you
  compute.

Price every settled row with the stake fractions rather than the label. A
bet's return (stake included) is

```
stake × (stake_won_fraction × decimal_odds + stake_refunded_fraction)
```

with `decimal_odds` the price you took (`closing_odds` converted, for a bet at
the close). `WON` is `1`/`0`, `LOST` `0`/`0`, `PUSH` and `VOID` `0`/`1`, and a
`PARTIAL` row carries the split, e.g. two players tied for the lead of a
most-of market is `0.5`/`0` at most books and `0.5`/`0.5` at DraftKings. Profit is the return minus the stake. Both fractions are null on
`INDETERMINATE` rows and on games built before v10 (`dataset_version`); leave
those out of ROI and report how many there were.

```python
def american_to_decimal(odds):
    return 1 + (odds / 100 if odds > 0 else 100 / -odds)

def bet_return(row, stake=1.0):
    if row["stake_won_fraction"] is None or row["closing_odds"] is None:
        return None   # INDETERMINATE, built before v10, or no pre-game close
    return stake * (row["stake_won_fraction"] * american_to_decimal(row["closing_odds"])
                    + row["stake_refunded_fraction"])
```

`sportsbook-market-results` includes NCAAM and NCAAF with beta settlement limits.
Box-score facts are available, but play-by-play is absent. Missing facts and unsupported
questions remain `INDETERMINATE`; college overtime and operator rules are not
comprehensively verified.

## Model-ready frames

A supervised setup wants three aligned pieces: features, binary outcomes, and
the odds you would have gotten. `sportsbook-market-results` carries the outcomes and the
closing price in one row, so it is the natural starting point:

```python
from betflux import Client

with Client() as bf:
    rows = bf.sportsbook_market_results.rows(
        league="NBA", date_from="2026-05-03", date_to="2026-05-03",
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

- `Y` — `outcome` mapped to `{WON: 1, LOST: 0}`. `PUSH` and `VOID` returned the
  stake and `INDETERMINATE` was not settled: drop or hold them out deliberately.
  For `PARTIAL`, use `stake_won_fraction` as a fractional label or hold it out;
  for profit, use the return formula in [Did these bets win?](#did-these-bets-win)
- `O` — `closing_odds` (or `opening_odds` if you are modelling the open)
- `X` — join `game-state-timeline` per game for in-play state, or bring your own
  features

Live-only selections are still settled, but their v3 `closing_odds`,
`closing_implied`, and `closing_ts` are null. Use `priced_rows` for
closing-price ROI or model evaluation; do not treat a missing close as zero
odds or drop it from outcome-only settlement summaries.

Fetch one day first and confirm the join before widening the range. A full
season across all leagues is a large download.

## Scoreboard state over time

```bash
betflux get game-state-timeline --game NBA_ORL_DET_20260503 --format jsonl
```

Beta, and every value is an estimate: say so, with the field's confidence,
when you report one. One row each time the estimated state changes; a row
holds until the next row's `ts`. Each field has a calibrated confidence beside it (the clock has a
p90 error in seconds). To get the state behind each line move, as-of join on
`ts`:

```sql
SELECT l.timestamp, l.odds, s.home_score, s.away_score, s.period_number, s.clock_seconds_remaining
FROM read_parquet('lines.parquet') l
ASOF JOIN read_parquet('state.parquet') s ON l.game_id = s.game_id AND l.timestamp >= s.ts
```

A row is the best estimate of the state true at `ts`, built with reports that
arrived later (including the game's full play-by-play), so it is not what was
knowable at that moment.
Allow for that before backtesting a strategy that reacts to the game state.
