# Design: add-data-ingestion

## Context

Greenfield repo: only docs and config exist. For motivation and the data-source finding
see `proposal.md` (Why). The behaviour contract is in `specs/complaint-ingestion/spec.md`.

Facts checked on 2026-11-08 that shape the design:
- Source: 21 static ZIPs in the CFPB FOIA Reading Room, URL pattern
  `https://files.consumerfinance.gov/f/documents/CCDB_Export_<n>_<range>.zip`. Each holds one CSV.
- The six 2025 exports (n = 9…14) total about 434 MB compressed (64–84 MB each).
- All exports checked share the same 16-column header. Some rows have an empty narrative.
- Target machine: Windows laptop, Docker Desktop, Python 3.12.

## Goals / Non-Goals

**Goals:**
- One command goes from "nothing" to "N complaints in Postgres", and the same command is safe to re-run.
- Memory use stays bounded (no full 2025 CSV in RAM at once).
- Pure logic (window → files, filtering, sampling) is unit-testable without network or DB.

**Non-Goals:**
- Generic ETL framework, scheduling, incremental updates (the archive is frozen).
- ORM models. Later changes may add tables with new migration files.

## Decisions

### D1. Data source: hard-coded archive manifest, not HTML scraping
A small Python constant lists the 21 exports: file name, first day, last day. The window
selects the files whose range overlaps it.
*Why:* the archive is frozen, so a manifest is exact and testable. Scraping the HTML page
would break the first time the CFPB redesigns it.
*Alternatives:* Kaggle/Hugging Face mirrors (need accounts, not official, may be stale);
live bulk CSV and API (no narratives any more).

### D2. Download: stream to `<name>.part`, then rename
`httpx` streams each ZIP to `data/raw/<name>.zip.part` and renames it to `<name>.zip`
only once it is complete. Files already at the final path are skipped.
*Why:* a rename is atomic, so an interrupted download can never look like a valid cached file.
That makes "re-run is safe" true for downloads as well as for the DB.
*Alternative:* `urllib`. It works too, but `httpx` has a test transport (`MockTransport`)
that makes the download testable without network.

### D3. Read straight from the ZIP in chunks with pandas
`pandas.read_csv(zipfile_member, usecols=..., dtype=str, chunksize=50_000)`, with no extraction step.
Filtering (narrative non-empty after `strip()`, date in window) runs per chunk, and only
the kept rows stay in memory. Column validation reads the header first and fails with
the missing column names.
*Why:* it saves the disk space of extracting the CSVs (several GB uncompressed) and keeps
memory bounded. `dtype=str` avoids pandas guessing types: ZIP codes like `00949` would
otherwise lose their leading zeros.

### D4. Sampling by hash rank, not `DataFrame.sample`
Each candidate gets `key = sha256(f"{seed}:{complaint_id}")`. The sample is the `N`
candidates with the smallest keys.
*Why:* this is deterministic and independent of row/file order. It is also **nested**:
the 5k sample is exactly the first 5k of the 20k sample. So the evaluation set (≈5k) is
automatically contained in the RAG set (≈20k), and growing the sample only adds rows.
`DataFrame.sample(random_state=…)` is reproducible but not nested, and it changes if the
row order changes.
*Trade-off:* every candidate must be hashed before choosing. That takes well under a
second per 100k rows.

### D5. Plain SQL migrations + a tiny runner
`migrations/001_create_complaints.sql` is applied by a ~30-line runner that records
applied file names in a `schema_migrations` table. `psycopg` 3 is the driver.
*Why:* one table does not justify Alembic + SQLAlchemy, and plain SQL is easy to read and
explain. The `schema_migrations` table is the same idea Alembic and Flyway use, so moving
to one later is easy.

`complaints` columns: `complaint_id BIGINT PRIMARY KEY`, `date_received DATE NOT NULL`,
`product TEXT NOT NULL`, `sub_product TEXT`, `issue TEXT NOT NULL`, `sub_issue TEXT`,
`narrative TEXT NOT NULL`, `company TEXT`, `state TEXT`, `submitted_via TEXT`,
`company_response TEXT`, `timely_response BOOLEAN`, `source_file TEXT NOT NULL`,
`ingested_at TIMESTAMPTZ NOT NULL DEFAULT now()`. Index on `(product)` for later evaluation queries.

### D6. Idempotent load: COPY into a temp table, then `INSERT … ON CONFLICT DO NOTHING`
Rows are bulk-copied into a temporary staging table, then
`INSERT INTO complaints SELECT … FROM staging ON CONFLICT (complaint_id) DO NOTHING`.
The statement's row count gives "inserted". Sampled minus inserted gives "already present".
*Why:* the primary key makes duplicates impossible at the database level, not just in
Python. `COPY` loads 20k rows in about a second, where row-by-row `INSERT` would take
much longer. `DO NOTHING` instead of `DO UPDATE` is fine because the archive never
changes a published complaint.

### D7. Configuration with pydantic-settings, CLI with argparse
A `Settings` model reads `DATABASE_URL`, `DATA_DIR`, `INGEST_DATE_FROM`, `INGEST_DATE_TO`,
`SAMPLE_SIZE` and `RANDOM_SEED` from the environment / `.env`. `scripts/ingest.py` uses
`argparse` flags that override them and calls `complaint_triage.ingestion.run(settings)`.
*Why:* this follows the project rule "Pydantic for config/IO", gives one validated place
for settings, and needs no extra CLI library.

### D8. Default window: calendar year 2025
*Why:* it has enough narratives for a 20k sample, a ~430 MB download, and one stable
product/issue taxonomy. The CFPB revised product names in 2017 and again in 2023, so a
window starting in 2024 or later avoids mixed labels. It can be widened to 2024–2025 with
two settings. The first real run prints product counts so the label set can be checked.

### D9. Tests
- Unit (default `pytest`): manifest selection, column validation, filtering, sampling
  determinism/nesting, and download caching with `httpx.MockTransport`. They use small
  fixture ZIPs built in `tmp_path`, so no network is needed.
- Integration (`pytest -m integration`): load + idempotency against the compose database.
  They are skipped automatically when the DB is unreachable, so CI (change 5) can run the
  unit suite alone.

### D10. Compose service
`db` uses `pgvector/pgvector:pg16`, reads credentials from `.env`, has a named volume
`pgdata`, a `pg_isready` healthcheck, and host port `${POSTGRES_PORT:-5432}`.
*Why a configurable port:* a native Postgres installed on Windows already holds 5432.

## Risks / Trade-offs

- [The CFPB moves or removes the archive files] → The URL list lives in one place. Cached
  ZIPs in `data/raw/` keep working. The README states where the data came from and when.
- [Class imbalance: credit-reporting complaints dominate] → The sample keeps the natural
  distribution (that is the real triage workload). Evaluation uses macro-F1, and change 3
  decides whether to stratify.
- [Product/issue names drift inside the window] → The run summary prints product counts.
  If drift shows up, narrow the window instead of mapping labels in this change.
- [Large narratives / odd encoding in the CSV] → Read as UTF-8 with `on_bad_lines="error"`
  so problems surface instead of being skipped silently.
- [Docker Desktop not running] → A clear DB-unreachable error suggests `docker compose up -d db`.

## Migration Plan

Not applicable (first change, empty database). Rollback: `docker compose down -v` drops
the volume. Deleting `data/raw/` forces a fresh download.
