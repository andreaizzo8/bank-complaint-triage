# Project brief — Bank Complaint Triage

Use this document as the source of context for OpenSpec (`openspec/project.md` or the project context section, depending on the OpenSpec version) and for every change proposal.

## 1. Purpose

Build a small but production-minded system that:

- triages bank/financial-product complaints with an LLM,
- **measures** how good that triage is against real labels,
- lets a user ask questions about the complaints (RAG with citations),
- is packaged, tested and documented like a real service.

It is a personal learning project to explore, end to end, the topics involved in building applied LLM systems: LLMs and prompting, structured + unstructured data pipelines, relational + vector databases, APIs, Docker, Git, CI/CD, monitoring, reliability, security, documentation.

## 2. Data

**Source:** CFPB Consumer Complaint Database (U.S. Consumer Financial Protection Bureau), public domain. Available as bulk CSV/JSON download and through a public search API on consumerfinance.gov; mirrored on Kaggle.

**Relevant fields (check exact names in the downloaded file):**

- `Complaint ID`, `Date received`
- `Product`, `Sub-product` → ground truth for product classification
- `Issue`, `Sub-issue` → ground truth for issue classification
- `Consumer complaint narrative` → free text (only present when the consumer consented; scrubbed of PII by the CFPB)
- `Company`, `State`, `Submitted via`, `Company response to consumer`, `Timely response?`

**Scope rules:**

- Keep only rows with a non-empty narrative.
- Work on a configurable, reproducible sample (fixed random seed). Default: ~5k complaints for evaluation, ~20k for RAG.
- The label set changes over the years (CFPB renamed products in 2017). Restrict to a recent time window (e.g. last 2–3 full years) so labels are consistent.
- Raw files stay in `data/raw/` (gitignored); the database is the single source for the app.

## 3. Functional scope

| Capability | Description |
|---|---|
| Ingestion | Download → clean → sample → load into PostgreSQL. Idempotent (re-running does not duplicate rows). |
| Classification | `POST /classify` with complaint text → `{product, issue, urgency, sentiment, rationale}`. Product and issue must be chosen from the label set found in the data (closed list), validated with Pydantic. Invalid LLM output is retried once, then flagged. |
| Batch classification | CLI script to classify the evaluation sample and store predictions in the DB, with model name, prompt version, latency. |
| Evaluation | Accuracy, macro-F1, per-class report, confusion matrix, average latency. Compare: (a) classical baseline TF-IDF + logistic regression, (b) LLM prompt v1, (c) LLM prompt v2 or a second local model. Results saved to `docs/results/` and summarised in the README. |
| RAG | Embed narratives into pgvector. `POST /ask` with a question → answer + list of cited complaint IDs and snippets. Answer must say "not enough information" when retrieval finds nothing relevant. |
| Observability | Structured logs (JSON), request IDs, timing for LLM calls, `/health` endpoint. |
| UI (optional, last) | Minimal Streamlit or HTML page to try `/classify` and `/ask`. |

Urgency and sentiment have no ground truth: define them explicitly in the spec (e.g. urgency = mentions of financial harm, deadlines, fraud, foreclosure) and evaluate them qualitatively on a small hand-checked sample.

## 4. Architecture

```
CFPB CSV/API ──► ingestion script ──► PostgreSQL (complaints, predictions)
                                          │            ▲
                                          │            │ batch classify
                                          ▼            │
                                   pgvector (embeddings)
                                          │
FastAPI ── /classify ──► LLMClient (Ollama) ◄── prompts/*.md
        ── /ask ───────► retriever (pgvector) ──► LLMClient
        ── /health
```

- `docker compose up` starts **PostgreSQL+pgvector** and the **API**. Ollama runs on the host (it is already installed on the dev machine) and the API reaches it via `OLLAMA_BASE_URL` (on Docker Desktop for Windows: `http://host.docker.internal:11434`).
- Database schema managed with SQL migration files (or Alembic) — keep it simple.

## 5. Zero-cost choices

- **LLM:** local models through Ollama. Suggested: a ~3B model for fast development (e.g. `llama3.2:3b` or `qwen2.5:3b`) and a ~7–8B model for the final evaluation (e.g. `qwen2.5:7b` or `llama3.1:8b`), depending on what the laptop can run.
- **Embeddings:** local embedding model through Ollama (e.g. `nomic-embed-text`).
- **Baseline:** scikit-learn, CPU only.
- **CI:** GitHub Actions (free for public repositories). CI runs lint + unit tests with mocked LLM; it does not call real models.
- **Deploy:** not required. Optionally later: a free tier hosting only the API with a mocked or tiny model, clearly labelled as a demo.

## 6. Non-functional requirements

- Reproducible: fixed seeds, pinned dependencies, documented commands.
- Testable: unit tests for parsing/validation/retrieval logic; LLM calls mocked.
- Safe: no secrets in git; input length limits on API endpoints; the LLM output is never executed or used to build SQL.
- Documented: README with architecture diagram, how to run, results table, limitations.

## 7. Out of scope

Authentication, multi-user features, fine-tuning, paid APIs, cloud infrastructure.
