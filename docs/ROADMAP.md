# Roadmap — OpenSpec changes

One OpenSpec change per row, in this order. Each one should end with something that runs and is committed.

| # | Change id | Goal | Done when |
|---|---|---|---|
| 1 | `add-data-ingestion` | Project skeleton (package, uv, ruff, pytest), `docker-compose.yml` with PostgreSQL+pgvector, download + clean + sample CFPB data, load into `complaints` table | `docker compose up -d db` + one command loads the sample; re-running does not duplicate rows; tests pass |
| 2 | `add-llm-classification` | `LLMClient` interface + Ollama implementation, versioned prompt, Pydantic output schema with closed label lists, FastAPI app with `/health` and `/classify` | `/classify` returns validated JSON for a sample complaint; invalid output is retried/flagged; unit tests with mocked LLM |
| 3 | `add-evaluation` | Batch classification script storing predictions; TF-IDF + logistic regression baseline; metrics (accuracy, macro-F1, confusion matrix, latency); results written to `docs/results/` | A results table comparing baseline vs. LLM prompt v1 vs. v2 (or second model) is in the README |
| 4 | `add-rag-qa` | Embeddings into pgvector, retriever, `/ask` endpoint with citations and "not enough information" behaviour | `/ask` answers 5 test questions with correct citations; retrieval tested |
| 5 | `add-production-readiness` | API container in compose, structured logging, request IDs, input limits, GitHub Actions (ruff + pytest + docker build), README polish | Green CI badge; `docker compose up` runs the whole stack |
| 6 | `add-demo-ui` *(optional)* | Small Streamlit/HTML UI for `/classify` and `/ask`; screenshots/GIF in README | UI works locally |

## After the project

Summarise the **measured** results in the README, e.g.:

> LLM-based complaint triage on 5k+ real banking complaints, reaching X% macro-F1 vs. ground-truth labels (Y% over a TF-IDF baseline), with FastAPI, PostgreSQL/pgvector RAG, Docker and CI/CD.
