# Getting started (Windows + VS Code)

Step-by-step setup for this project. Everything here is free.

## 1. Tools to install (once)

| Tool | Why | Check |
|---|---|---|
| Git | version control | `git --version` |
| Python 3.11+ | the app | `python --version` |
| uv | Python env + dependencies | `pip install uv` → `uv --version` |
| Node.js 20+ | needed by OpenSpec | `node --version` |
| OpenSpec | spec-driven workflow | `npm install -g @fission-ai/openspec@latest` → `openspec --version` |
| Docker Desktop | PostgreSQL + pgvector, later the API | `docker --version` (free for personal use) |
| Ollama | local LLMs, no API costs (already installed) | `ollama --version` |
| Claude Code | AI-assisted development | `claude --version` |

## 2. Download the local models

```powershell
ollama pull llama3.2:3b        # fast model for development
ollama pull nomic-embed-text   # embeddings for RAG
# later, for the final evaluation, if the laptop handles it:
ollama pull qwen2.5:7b
```

Quick test: `ollama run llama3.2:3b "Classify: my bank charged me twice for an overdraft fee"`.

## 3. Initialise the repo

From this folder in a VS Code terminal:

```powershell
git init
git add .
git commit -m "Project setup: brief, roadmap, Claude Code guidance"
openspec init        # choose Claude Code when asked which AI tools to configure
```

`openspec init` creates the `openspec/` folder and the Claude Code slash commands for the version you installed, which is why it is not pre-created here.

## 4. Give OpenSpec the project context

Open Claude Code in the folder (`claude`) and send:

> Read CLAUDE.md, docs/PROJECT_BRIEF.md and docs/ROADMAP.md. Then fill in the OpenSpec project context (the file `openspec init` created for project conventions/context) using the brief: purpose, tech stack, conventions, constraints (zero cost, local Ollama, no raw data in git), domain context about the CFPB dataset. Keep it concise.

Review the result, then commit it.

## 5. First change

Use the OpenSpec proposal command in Claude Code (e.g. `/opsx:propose` or `/openspec:proposal`, depending on the version — type `/` to see them) with:

> Create the change `add-data-ingestion` as described in row 1 of docs/ROADMAP.md and sections 2–5 of docs/PROJECT_BRIEF.md. Include: project skeleton with uv, ruff and pytest; docker-compose.yml with the pgvector/pgvector:pg16 image; a script that downloads the CFPB complaints data, keeps rows with a narrative in a recent time window, takes a reproducible sample (configurable size and seed) and loads it into a `complaints` table idempotently. First check the current CFPB download options and field names, and state them in the proposal.

Then:

1. **Review** `proposal.md`, `tasks.md` and the spec deltas. Ask Claude Code to change anything unclear *before* coding.
2. **Apply** the change (apply command), task by task.
3. **Verify**: run the tests and the ingestion command yourself, check the rows in the DB.
4. **Archive** the change (archive command) and commit.

Repeat for each row of the roadmap.

## 6. Publish on GitHub

Create an empty public repo `bank-complaint-triage` on GitHub, then:

```powershell
git remote add origin https://github.com/andreaizzo8/bank-complaint-triage.git
git branch -M main
git push -u origin main
```

Keep the `openspec/` folder in the repo: it shows how the project was specified, not just the final code.
