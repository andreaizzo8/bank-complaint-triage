# Proposal: add-data-ingestion

Roadmap: row 1 of `docs/ROADMAP.md` (`add-data-ingestion`).

## Why

Every later capability (classification, evaluation, RAG) needs a reproducible, local
copy of real complaints with their ground-truth labels in PostgreSQL. Nothing exists
yet: the repository has no package, no tooling and no database.

**Data-source finding (checked 2026-11-08).** The CFPB stopped publishing complaint
narratives in the live Consumer Complaint Database (release 24, September 2026). The
live bulk file `https://files.consumerfinance.gov/ccdb/complaints.csv.zip` (~365 MB)
now has **no** `Consumer complaint narrative` column, and the search API rejects
automated clients ("Access Denied"). The narratives are still public. The CFPB
moved them to the **FOIA Reading Room "Consumer Complaint Database Narratives
Archive"**: 21 static ZIP exports (one CSV each) covering complaints received from
2011-12-01 to 2026-08-14. This change uses that archive. Being frozen also makes it
better for reproducibility.

Archive CSV columns (identical header verified on the 2025 files):
`Date received, Product, Sub-product, Issue, Sub-issue, Consumer complaint narrative,
Company public response, Company, State, ZIP code, Tags, Submitted via,
Date sent to company, Company response to consumer, Timely response?, Complaint ID`.
Some rows have an empty narrative.

## What Changes

- Project skeleton: `src/complaint_triage/` package managed with `uv`, `ruff` and `pytest`
  configured, `pyproject.toml` with pinned dependencies.
- `docker-compose.yml` with a `db` service (`pgvector/pgvector:pg16`), a persistent volume
  and credentials taken from `.env`.
- SQL migration creating the `complaints` table (structured fields + narrative).
- An ingestion CLI that:
  - downloads the archive exports covering the configured date window into `data/raw/`
    (skipped if already present),
  - keeps only rows with a non-empty narrative inside the window (default: calendar year 2025),
  - takes a reproducible sample (configurable size and seed),
  - loads it into `complaints` idempotently (re-runs never duplicate rows).
- `.env.example`, `README.md` (quick start) and `docs/GETTING_STARTED.md` updated to
  describe the archive source and the ingestion command.

## Capabilities

### New Capabilities
- `complaint-ingestion`: obtaining CFPB complaints from the public archive, filtering
  them to a consistent labelled subset, sampling reproducibly and storing them in the
  database without duplicates.

### Modified Capabilities
- None (no existing specs).

## Out of scope

- LLM calls, embeddings and the `pgvector` extension usage (only the image is chosen now
  so later changes do not need a new database). These belong to changes 2 and 4.
- The API container, FastAPI app and CI workflow (changes 2 and 5).
- Normalising or merging labels (e.g. mapping old product names). The date window keeps
  the label set consistent instead.
- Choosing the evaluation subset or the train/test split for the baseline (change 3).
- Any use of the live CFPB API or the live bulk file.

## Impact

- New: `pyproject.toml`, `uv.lock`, `src/complaint_triage/`, `tests/`, `scripts/`,
  `migrations/`, `docker-compose.yml`.
- Updated: `.env.example`, `README.md`, `docs/GETTING_STARTED.md` (sections that say the
  data comes from the live bulk CSV/API), `docs/PROJECT_BRIEF.md` §2 (data source note).
- Dependencies: `pandas`, `psycopg[binary]`, `pydantic`, `pydantic-settings`, `httpx`;
  dev: `pytest`, `ruff`. All free and open source.
- Disk: about 430 MB of downloads for the default window (2025), stored in `data/raw/`.
- Requires Docker Desktop running for the database.
