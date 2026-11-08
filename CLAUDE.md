# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Project

Bank Complaint Triage: LLM classification + RAG over the public CFPB Consumer Complaint Database.
Full context, goals and architecture: **read `docs/PROJECT_BRIEF.md` before proposing any change.**
Planned changes and their order: `docs/ROADMAP.md`.

## Workflow: spec-driven with OpenSpec

- Every feature starts as an OpenSpec change proposal (`openspec/changes/<change-id>/`). Do not write implementation code for a feature that has no approved proposal.
- Implement one change at a time, following its `tasks.md`; tick tasks as they are completed.
- When a change is done and verified, archive it with OpenSpec so `openspec/specs/` reflects the current behaviour.
- If something in the spec turns out wrong or ambiguous while implementing, stop and update the proposal/spec first, then continue.

## Hard constraints

- **Zero cost.** No paid APIs or services. LLMs and embeddings run locally through Ollama. Keep the LLM client behind a small interface (`LLMClient`) so a different provider could be plugged in later, but do not add paid providers as dependencies or defaults.
- **Runs on a laptop.** Default dataset sample must be small enough to process locally (configurable sample size, default ~5,000 complaints for evaluation). Prefer small models (3B–8B parameters).
- **No raw data in git.** Downloaded files go in `data/` (gitignored). Scripts must be able to re-download them.
- **Secrets / config** via environment variables (`.env`, never committed; keep `.env.example` updated).

## Code conventions

- Python 3.11+, type hints everywhere, Pydantic models for all API and LLM I/O.
- Layout: `src/complaint_triage/` (package), `tests/`, `scripts/` (one-off CLI tasks), `docs/`.
- Dependency management with `uv` (fallback: `pip` + `requirements.txt`).
- Formatting/linting: `ruff`. Tests: `pytest`; every change adds or updates tests. LLM calls are mocked in unit tests.
- Prompts live in versioned files (`prompts/<task>_v<N>.md`), never inline in code, so evaluations can compare versions.
- Small, focused commits with clear messages.

## The developer

Andrea is learning this stack (Docker, FastAPI, RAG, CI). When making non-obvious decisions, explain the *why* briefly, with the alternatives considered, so the reasoning is clear and easy to revisit. Prefer simple, well-known solutions over clever ones.
