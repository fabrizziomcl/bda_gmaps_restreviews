# BDA: Google Maps Restaurant Reviews Pipeline

Two-stage pipeline that scrapes Peruvian restaurants from Google Maps and
feeds a Medallion (Bronze → Silver → Gold) architecture in Spark. The
parent repository wires together two independent engines as git submodules.

## Repository layout

| Path | Purpose |
|---|---|
| [mapScraper/](mapScraper/) | Phase 1 — country-wide place ingestion via Google's `tbm=map` endpoint. |
| [reviewsScraper/](reviewsScraper/) | Phase 2 — async Playwright extraction of user reviews. |
| [notebooks/](notebooks/) | Medallion-architecture transforms and analytics on the scraped data. |
| [docs/](docs/) | Technical report and architecture documentation. |
| [.pre-commit-config.yaml](.pre-commit-config.yaml) | Ruff (lint + format) and hygiene hooks applied at the parent level. |

Each submodule is self-contained: it has its own `pyproject.toml`, ruff/black
config, test suite, and console entry points. See its README for engine-level
details.

## Quickstart

```bash
git clone --recursive git@github.com:fabrizziomcl/bda_gmaps_restreviews.git
cd bda_gmaps_restreviews

# If the submodules came in empty (cloned without --recursive):
git submodule update --init --recursive
```

## End-to-end pipeline

```text
[ Ubigeo open data ]
      │  config/get_dist.py
      ▼
geo_ref_pe.csv  +  categories
      │  orchestrator_peru.py  (async aiohttp, atomic per-district writes,
      ▼                         retry on 429/5xx, JSON manifests)
data/{Dep}/{Prov}/{Dist}.csv
      │  python -m etl.pipeline  (Polars + ZSTD Parquet)
      ▼
data_parquet/Peru/Perú.parquet                                   ─ Phase 1 ─
      │
      ▼
reviewsScraper/orchestrator.py  (1 Chromium + N async workers,
                                 DOM-stable end-of-feed, fsync writes)
      │  python -m etl.pipeline
      ▼
data_parquet/Peru/reviews_peru.parquet                           ─ Phase 2 ─
      │
      ▼
notebooks/Medallon.ipynb                          (Bronze → Silver → Gold)
```

### Phase 1 — places

```bash
cd mapScraper
python orchestrator_peru.py                       # full Peru sweep, resumable
python orchestrator_peru.py --deps '["Lima"]'     # single department
python -m etl.pipeline                            # consolidate CSVs to Parquet
```

Each district produces `{Dist}.csv` plus a sibling `{Dist}.json` manifest
(start/finish timestamps, rows per category, run parameters). Writes are
atomic (`.partial → os.replace`), so resume after a crash is safe.

### Phase 2 — reviews

```bash
cd reviewsScraper
python orchestrator.py --input data/input/places_peru.csv --workers 8
python orchestrator.py ... --max-reviews 0        # unlimited (≥15k reviews)
python orchestrator.py ... --retry-non-terminal   # re-queue partial captures
python -m etl.pipeline --incremental              # skip if input unchanged
```

Worker termination is recorded with a reason
(`ok | no_reviews_tab | dom_stable | timeout | error`); only the first two
are terminal, so partial captures can be retried by intent.

### Picking the right worker count

```bash
cd reviewsScraper
python utils/benchmark_workers.py generate-golden          # ground truth
python utils/benchmark_workers.py bench --configs 1,4,8,12 \
    --min-recall 0.99                                      # recall-gated
```

The "optimal" config is the **fastest** run whose mean per-place recall
clears `--min-recall` *and* whose worst-place recall clears 0.95× that —
so a single botched place can't be hidden by a healthy average.

### Streaming hook

`reviewsScraper/monitor.py` polls already-mapped places on an interval and
emits only reviews newer than each place's watermark. Output is pluggable
via a `Sink` protocol (CSV today, Kafka hook ready):

```bash
python monitor.py --input data/input/places_peru.csv --interval 1800
```

## Quality

| | Tests | Lint |
|---|---|---|
| mapScraper | `pytest` — 18 cases (extractor shape, reviews-column regression, ETL roundtrip) | `ruff check .` clean |
| reviewsScraper | `pytest` — 29 cases (date parsing ES/EN, ETL, pre-filter, dedup) | `ruff check .` clean |

Set `BDA_LOG_JSON=1` to switch every entry point to single-line JSON logs
(Datadog / Loki / Log Analytics ready).

## Documentation

See [docs/bda_reporte.pdf](docs/bda_reporte.pdf) for the architectural
write-up and methodology. Per-engine usage details live in
[mapScraper/README.md](mapScraper/README.md) and
[reviewsScraper/README.md](reviewsScraper/README.md).
