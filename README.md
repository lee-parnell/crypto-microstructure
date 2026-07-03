# crypto-microstructure

This is a market microstructure research project I built on live exchange
data. I captured one night of Binance.US (top-20 book snapshots about once
a second, plus the trade tape), and then a longer dual-venue run on Kraken
and Coinbase (book first, trades added later as separate loggers). All of
it feeds a typed, tested pipeline you can rerun end to end.

It's not a trading strategy and I'm not pretending it is. The main result
is a pre-registered cross-venue replication. Night one's headline was SOL
trade-sign order-flow memory, and I froze that hypothesis in
`REPLICATION.md` before I touched the new data. Then I reran it on Kraken
and Coinbase and it held on both (SOL ACF(1-5) +0.26 and +0.33, Ljung-Box
p about 0, versus +0.15 on night one). That's independent confirmation of a
known effect (order-flow autocorrelation, Cont et al.), not something new I
discovered. What I actually care about is doing it carefully: a frozen
protocol, a trade-side convention I validated instead of assuming (without
it Coinbase's result would have silently flipped), and a print-fragmentation
confound I caught in a bug hunt. Once I collapsed that, Kraken lands at
+0.159, basically the same +0.15 from night one, so it's robust and not an
artifact.

Everything else is a negative result or a hedged one, and I kept those
separate from the replication. No strategy in the family I tested survives
retail fees (round-trip cost is roughly 1000x the per-trade edge).
Cross-venue mid divergence never beats a round trip. And the earlier
book-only night-two analyses (a tick-sign analogue and the Epps curve) came
back mixed, which I'm reporting straight, with one venue voided by a
data-quality freeze the pipeline caught on its own.

So the real point is the methodology: clean data layers, feature
definitions I checked by hand, statistics that punish multiple testing, a
backtester that physically can't look ahead, and a pre-registered
replication I reported either way. I also left the messy progression in on
purpose (book-only "not testable", then trade capture added, then
replicated) instead of tidying it up after the fact.

## Architecture

```
            live capture (runs unattended)
  ┌─────────────────────────────────────────────┐
  │  logger.py            trades_logger.py      │
  │  book snapshots 1/s   every trade           │
  │        └──────┬──────────────┘              │
  │               ▼                             │
  │  data/book-*.jsonl.gz, trades-*.jsonl.gz    │  hourly-rotated, gzip,
  └───────────────┬─────────────────────────────┘  flushed every write
                  ▼
        src/microstructure/
  ┌─────────────────────────────────────────────┐
  │ ingest.py    typed parsing, live-file safe  │
  │ quality.py   gaps / crossed books / report  │
  │ parquet.py   symbol-hour partitioned store  │
  │ features.py  microprice, OFI, imbalance,    │
  │              RV / Parkinson / bipower       │
  │ anomaly.py   spread/depth/vol episode flags │
  │ backtest.py  event replay, walk-the-book    │
  │              fills, deflated Sharpe         │
  │ analysis.py  shared plotting/loading        │
  └───────────────┬─────────────────────────────┘
                  ▼
              scripts/            output/
  build_data.py ──────────────▶ data_quality.md
  analysis_*.py (7 studies) ──▶ *.png + *.md
  run_backtest.py ────────────▶ backtest.md
  build_findings.py ──────────▶ ../FINDINGS.md
```

## Quick start

```bash
python3 -m venv .venv && .venv/bin/pip install -e '.[dev]'
make all     # parquet + quality report + 7 studies + backtest + FINDINGS.md
make check   # ruff + mypy --strict + pytest
```

The loggers (`logger.py`, `trades_logger.py`, stdlib + `websockets` only)
run separately: `caffeinate -i python3 logger.py`.

## What's inside

| module | what it does | the detail that matters |
|---|---|---|
| `ingest.py` | jsonl.gz to typed dataclasses | tolerates truncated gzip tails from live files |
| `quality.py` | validation | gaps, crossed books, backwards timestamps, all known before any analysis |
| `parquet.py` | columnar store | idempotent; only rewrites partitions that grew |
| `features.py` | feature library | every formula stated in the docstring, unit-tested against hand-computed numbers |
| `anomaly.py` | episode detection | trailing-robust thresholds tuned on real tick-quantized data, not textbook z-scores |
| `backtest.py` | event-driven replay | decisions at snapshot *t* execute against snapshot *t+1*, so lookahead is structurally impossible |

## Honesty constraints

- Imbalance/return regressions use Newey-West errors and a Bonferroni
  bar across all 27 hypotheses tested.
- The backtest pays taker fees, crosses the spread, and cannot fill more
  than displayed size; its Sharpe is deflated for the 9 configs tried
  and compared against 50 random-strategy seeds.
- Every report carries a data-span note; `FINDINGS.md` opens with what
  one night of single-venue snapshots cannot support.

## Results

Full detail is in [FINDINGS.md](FINDINGS.md) (regenerated by `make all`).
The headlines:

- H1, SOL trade-sign memory, replicated across venues. Kraken ACF(1-5)
  +0.26, Coinbase +0.33, both Ljung-Box p about 0 (night one was +0.15).
  Still holds after collapsing print fragmentation (+0.159 / +0.291). It's
  a known effect confirmed on independent venues, not a novel finding.
- Economic null, strategies lose to fees. The best backtest config has a
  deflated Sharpe of 0.00, inside the random-strategy null, and cross-venue
  divergence never beats a round trip. Statistically real does not mean
  tradable.
- Night-two book-only (dated, kept in on purpose): the tick-sign analogue
  didn't replicate, and the Epps curve replicated on Kraken but failed the
  criterion on Coinbase. Mixed verdict, reported as-is.

Every result keeps its caveats attached: 1 second book cadence,
retail-venue data, and a trade-side convention (`SIDE_CONVENTION.md`) that
the H1 verdict depends on.
