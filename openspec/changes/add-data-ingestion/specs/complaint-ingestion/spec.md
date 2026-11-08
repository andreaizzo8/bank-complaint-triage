# Spec Delta: complaint-ingestion

## Purpose

Provide a reproducible, local, labelled set of real CFPB consumer complaints (with
narratives) in PostgreSQL, so classification, evaluation and RAG all work from the same data.

## ADDED Requirements

### Requirement: Download archive exports for the configured window
The system SHALL download from the CFPB Narratives Archive only the export files whose
date range overlaps the configured date window, store them under `data/raw/`, and reuse
files already present instead of downloading them again.

#### Scenario: First run downloads the needed exports
- **WHEN** ingestion runs with the default window (2025-01-01 to 2025-12-31) and `data/raw/` is empty
- **THEN** the six exports covering January to December 2025 are downloaded to `data/raw/`
- **AND** no export outside that window is downloaded

#### Scenario: Re-run uses cached files
- **WHEN** ingestion runs again and the needed exports are already in `data/raw/`
- **THEN** no network download is performed

#### Scenario: Failed download leaves no corrupt cache
- **WHEN** a download fails or is interrupted
- **THEN** the command exits with a non-zero status and an error naming the file
- **AND** no partial file remains at the export's final path in `data/raw/`

#### Scenario: Window outside the archive
- **WHEN** the configured window has no overlap with 2011-12-01 to 2026-08-14
- **THEN** the command exits with a non-zero status and explains the available range

### Requirement: Validate the source columns
The system SHALL check that each export contains the columns it relies on (Complaint ID,
Date received, Product, Sub-product, Issue, Sub-issue, Consumer complaint narrative,
Company, State, Submitted via, Company response to consumer, Timely response?) and
SHALL stop with an error naming any missing column.

#### Scenario: Missing narrative column
- **WHEN** an export lacks the `Consumer complaint narrative` column
- **THEN** the command exits with a non-zero status and the error names that column and file

### Requirement: Keep only labelled complaints with a narrative in the window
The system SHALL keep only rows whose narrative is non-empty after trimming whitespace
and whose `Date received` falls inside the configured window (both ends inclusive).
Rows with the same Complaint ID SHALL be kept once.

#### Scenario: Empty and blank narratives are dropped
- **WHEN** a row has an empty or whitespace-only narrative
- **THEN** it is not part of the candidate set

#### Scenario: Window boundaries are inclusive
- **WHEN** rows are received on the window's first day, last day, and the day after the last day
- **THEN** the first two are candidates and the third is not

#### Scenario: Duplicate complaint IDs across exports
- **WHEN** the same Complaint ID appears in two exports
- **THEN** it appears once in the candidate set

### Requirement: Reproducible, nested sampling
The system SHALL select the sample deterministically from the candidate set, the sample
size and the seed. With the same seed, a smaller sample SHALL be a subset of a larger one.
If fewer candidates exist than requested, all candidates SHALL be used and a warning shown.

#### Scenario: Same inputs give the same sample
- **WHEN** ingestion runs twice with the same window, sample size and seed
- **THEN** both runs select exactly the same Complaint IDs

#### Scenario: Smaller sample is contained in a larger one
- **WHEN** samples of 5,000 and 20,000 are drawn with the same seed and window
- **THEN** every Complaint ID in the 5,000 sample is in the 20,000 sample

#### Scenario: Different seed gives a different sample
- **WHEN** the seed changes and everything else stays the same
- **THEN** the selected Complaint IDs differ

#### Scenario: Not enough candidates
- **WHEN** the requested size exceeds the number of candidates
- **THEN** all candidates are loaded and a warning reports both numbers

### Requirement: Idempotent load into the complaints table
The system SHALL store each sampled complaint as one row in the `complaints` table keyed
by Complaint ID, with its date, product, sub-product, issue, sub-issue, narrative, company,
state, submission channel, company response and timely-response flag. Re-running SHALL
never create duplicate rows.

#### Scenario: Re-running does not duplicate rows
- **WHEN** ingestion runs twice with the same settings
- **THEN** the row count in `complaints` is the same after both runs

#### Scenario: Growing the sample adds only new rows
- **WHEN** ingestion runs with size 5,000 and then with size 20,000 and the same seed
- **THEN** the table holds 20,000 rows and the first 5,000 are unchanged

#### Scenario: Empty optional fields are stored as NULL
- **WHEN** a sampled row has no Sub-product or Sub-issue
- **THEN** the corresponding column is NULL, not an empty string

### Requirement: Configuration and run summary
The system SHALL read the database URL, window, sample size and seed from environment
variables (documented in `.env.example`), allow CLI flags to override them, and print a
summary of rows read, candidates, sampled, inserted and already present.

#### Scenario: CLI flag overrides environment
- **WHEN** `SAMPLE_SIZE=5000` is set and the command is run with `--sample-size 100`
- **THEN** 100 complaints are sampled

#### Scenario: Database unreachable
- **WHEN** the database cannot be reached
- **THEN** the command exits with a non-zero status and a message suggesting `docker compose up -d db`

#### Scenario: Summary after a run
- **WHEN** ingestion finishes
- **THEN** it prints counts for rows read, candidates, sampled, inserted and already present
