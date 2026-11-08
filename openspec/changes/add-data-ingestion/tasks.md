# Tasks: add-data-ingestion

## 1. Project skeleton

- [ ] 1.1 Create `pyproject.toml` (Python >=3.11, package `complaint_triage` in `src/`), add dependencies `pandas`, `psycopg[binary]`, `pydantic`, `pydantic-settings`, `httpx` and dev dependencies `pytest`, `ruff` with `uv add`; verify `uv sync` succeeds and `uv.lock` is created
- [ ] 1.2 Configure ruff (line length, import sorting) and pytest (`testpaths`, `integration` marker) in `pyproject.toml`; add `src/complaint_triage/__init__.py` and a trivial `tests/test_package.py`; verify `uv run ruff check .`, `uv run ruff format --check .` and `uv run pytest` all pass
- [ ] 1.3 Add `Settings` (pydantic-settings) in `src/complaint_triage/config.py` with `DATABASE_URL`, `DATA_DIR`, `INGEST_DATE_FROM`, `INGEST_DATE_TO`, `SAMPLE_SIZE`, `RANDOM_SEED` and validation (`from <= to`, size > 0); update `.env.example`; verify with `tests/test_config.py` (defaults, env override, invalid window rejected)

## 2. Database

- [ ] 2.1 Add `docker-compose.yml` with the `db` service (`pgvector/pgvector:pg16`, `.env` credentials, `pgdata` volume, `pg_isready` healthcheck, port `${POSTGRES_PORT:-5432}`); add `POSTGRES_PORT` to `.env.example`; verify `docker compose up -d db` reports the service healthy
- [ ] 2.2 Add `migrations/001_create_complaints.sql` (table and index from design D5) and the migration runner in `src/complaint_triage/db.py` (connect, apply pending files in order, record them in `schema_migrations`, clear error when DB unreachable); verify with an integration test that applying twice is a no-op and the table exists

## 3. Archive download

- [ ] 3.1 Add the archive manifest (21 exports with date ranges) and `exports_for_window(date_from, date_to)` in `src/complaint_triage/ingestion/archive.py`; verify with unit tests: default 2025 window → exactly exports 9–14, partial overlap cases, window outside 2011-12-01…2026-08-14 raises a clear error
- [ ] 3.2 Implement streamed download to `<name>.zip.part` + rename, skipping files already cached (design D2); verify with `httpx.MockTransport` tests: cached file → no request; failed/interrupted response → non-zero error and no file at the final path

## 4. Reading, filtering, sampling

- [ ] 4.1 Implement header validation and chunked reading from the ZIP (`dtype=str`, `usecols`) in `src/complaint_triage/ingestion/reader.py`; verify with fixture-ZIP tests: missing column → error naming column and file; ZIP code leading zeros preserved
- [ ] 4.2 Implement filtering (trimmed narrative non-empty, inclusive date window, dedupe by Complaint ID) and conversion to a Pydantic `ComplaintRecord` (empty strings → `None`, `Timely response?` → bool); verify with unit tests covering each scenario of the "Keep only labelled complaints" requirement
- [ ] 4.3 Implement hash-rank sampling in `src/complaint_triage/ingestion/sampling.py` (design D4); verify with unit tests: same seed → same IDs, 5k ⊂ 20k, different seed → different IDs, size > candidates → all rows + warning, result independent of input order

## 5. Load and CLI

- [ ] 5.1 Implement the idempotent load (COPY into temp table → `INSERT … ON CONFLICT DO NOTHING`, returns inserted count) in `src/complaint_triage/ingestion/loader.py`; verify with integration tests: running twice keeps the row count, 5k then 20k gives 20k rows with the first 5k unchanged, missing sub-fields stored as NULL
- [ ] 5.2 Wire `run(settings)` (migrate → download → read/filter → sample → load → summary with counts and top products) and `scripts/ingest.py` with argparse overrides (`--sample-size`, `--seed`, `--date-from`, `--date-to`); verify with a unit test that a CLI flag overrides the env value and that DB-unreachable exits non-zero with the `docker compose up -d db` hint
- [ ] 5.3 Update `README.md` (Quick start: setup, `docker compose up -d db`, ingest command, data source note), `docs/GETTING_STARTED.md` and `docs/PROJECT_BRIEF.md` §2 (narratives now come from the FOIA archive; live database no longer has them); verify the README commands match the real CLI flags

## 6. End-to-end verification

- [ ] 6.1 Run the full check on a clean database and record the summary output in the change notes:
  ```powershell
  uv sync
  uv run ruff check . ; uv run ruff format --check .
  uv run pytest
  docker compose up -d db
  uv run pytest -m integration
  uv run python scripts/ingest.py --sample-size 5000
  uv run python scripts/ingest.py --sample-size 5000   # second run: inserted = 0
  uv run python scripts/ingest.py --sample-size 20000  # inserted = 15000
  docker compose exec db psql -U triage -d complaints -c "SELECT count(*), count(DISTINCT complaint_id) FROM complaints;"
  ```
  Verified when both counts are 20000, the second run reports 0 inserted, and every test passes

## Workflow follow-up

- Review the result, then archive the change with `/opsx:archive` so `openspec/specs/complaint-ingestion/` reflects the behaviour.
- Commit (small commits per task group are fine) and push to GitHub.
