# Bank Complaint Triage

LLM-based triage and Q&A over **real consumer complaints about banks and financial products**, built with spec-driven development (OpenSpec + Claude Code).

> Status: 🚧 project setup — see [docs/ROADMAP.md](docs/ROADMAP.md)

## What it does

1. **Ingests** public complaints from the U.S. Consumer Financial Protection Bureau (CFPB) Consumer Complaint Database into PostgreSQL (structured fields + free-text narratives).
2. **Classifies** each complaint with an LLM (product, issue, urgency, sentiment) using structured, validated output.
3. **Evaluates** the classifier against the ground-truth labels already in the dataset (accuracy, macro-F1, confusion matrix, latency) and compares it with a classical ML baseline.
4. **Answers questions** about the complaints with RAG (pgvector), citing the complaints used.
5. **Serves** everything through a FastAPI REST API, packaged with Docker Compose and tested in CI.

## Why

Complaint handling is a regulated, high-volume process in banking. Triage is usually manual and slow. This project shows how an LLM pipeline can route complaints automatically, and — more importantly — **how well it actually works, measured against real labels**.

## Stack (100% free / local)

| Layer | Tool |
|---|---|
| Language / API | Python 3.11+, FastAPI, Pydantic |
| Database | PostgreSQL 16 + pgvector (Docker) |
| LLM | Local models via [Ollama](https://ollama.com) (no API costs) |
| Embeddings | Local embedding model via Ollama |
| Baseline | scikit-learn (TF-IDF + logistic regression) |
| Packaging | Docker Compose |
| Tests / CI | pytest, GitHub Actions |
| Dev workflow | OpenSpec (spec-driven) + Claude Code |

## Quick start

_To be completed by the first OpenSpec change (`add-data-ingestion`)._

## Results

_To be filled in after the evaluation change: accuracy / F1 per model, cost and latency._

## Data

CFPB Consumer Complaint Database — public U.S. government data. Only complaints where consumers opted to publish their narrative are used; narratives are already scrubbed of personal information by the CFPB. Raw data is **not** committed to this repository.
