# AI Code-Review Copilot (Local-First, Reproducible)

A privacy-preserving AI bot that reviews Python pull requests, flags bugs/smells, suggests minimal patches, and enforces style/security policies. Built for reproducibility and benchmarking with an evaluation harness and BI-friendly metrics.

## Features
- Diff-aware AI reviews using local LLMs (via Ollama) with inline PR comments and patch suggestions.
- Static analysis stack: Ruff/Black, Bandit, MyPy; optional pip-audit/Semgrep.
- Policy engine (YAML) to gate merges on severity, coverage deltas, and custom rules.
- Patch validation in an isolated container: git apply + pytest + coverage.
- Reproducible evaluation harness with seeded bugs, pinned model versions, and metric logging to DuckDB for dashboards (Power BI/Tableau/Superset).
- Local-first by default; optional cloud toggle for model comparison.

## Architecture
- FastAPI service receives GitHub/GitLab webhooks → clones repo → collects diffs.
- Context builder parses AST (tree-sitter) to include nearest symbols/imports with each diff hunk.
- LLM layer generates review findings and minimal unified-diff patches; validator applies patches and runs tests.
- Results posted back to the PR and persisted as SARIF/JSON/Markdown; metrics stored in DuckDB.

## Quickstart

Prerequisites
- Docker and Docker Compose
- Python 3.10+
- GitHub/GitLab repo with Python project and tests
- Ollama installed with at least one model pulled (e.g., Llama 3 or Mistral)

1) Clone and set up
- git clone <your-repo>
- cd ai-code-review-copilot
- cp app/config/rules.example.yaml app/config/rules.yaml
- python -m venv .venv && source .venv/bin/activate
- pip install -r requirements.txt

2) Start services (Ollama + API)
- ollama serve  # in a separate terminal
- ollama pull llama3  # or mistral
- docker compose up -d  # optional, to run sandbox/test infra
- uvicorn app.main:app --reload

3) Configure webhook
- Expose locally: uvicorn on port 8000; use ngrok if needed.
- Point your GitHub/GitLab webhook to POST /webhook with push/PR events.

4) Dry run on a repo/PR
- python scripts/bootstrap_repo.sh https://github.com/<org>/<repo>.git
- python scripts/run_local.sh --repo .tmp/repo --base <base_sha> --head <head_sha>

## Configuration (app/config/rules.yaml)
- severity_threshold: high
- fail_on:
  - bandit: HIGH
  - coverage_drop_pct: 2
- enable_llm_patch_suggestions: true
- languages: [python]
- exclude_paths: ["docs/", "migrations/"]
- model:
  - provider: ollama
  - name: llama3
  - temperature: 0.2
  - max_tokens: 1024

## Repo Layout
- app/
  - main.py — FastAPI server, webhook handler
  - reviewers/
    - linters.py — Ruff/Black/Bandit/MyPy runners → normalized findings
    - llm.py — Ollama client, prompt templates, patch generator
    - context.py — AST/diff packing (tree-sitter)
    - policy.py — YAML rule engine
    - comments.py — PR comment/SARIF/Markdown emitters
  - vcs/
    - git.py — clone/checkout, diff, changed files
    - ci.py — GitHub/GitLab API helpers
  - exec/
    - sandbox.py — Dockerized test runner, git apply, coverage gate
  - eval/
    - seed_bugs.py — mutation seeding for ground truth
    - harness.py — batch evaluation, logging
    - metrics_store.py — DuckDB sink
  - config/
    - rules.example.yaml
- dashboards/ — SQL for BI, dataset notes
- scripts/ — run_local.sh, run_eval.sh, bootstrap_repo.sh
- Dockerfile, docker-compose.yml
- pyproject.toml / requirements.txt
- README.md

## Prompts (overview)
- System: “You are a senior Python reviewer. Be concise. Output JSON with issues and an optional unified diff patch.”
- User context: repo policy summary + file header + diff hunk + nearest symbols.
- Output schema: {issues: [{file, line, severity, rule, message}], patch: "unified diff"}.

## Evaluation 
- Datasets: pinned OSS SHAs + synthetic bug seeds (off-by-one, unsafe eval, missing closes).
- Metrics:
  - Detection: precision/recall/F1 vs. seeded ground truth.
  - Patching: patch apply rate, unit-test pass rate.
  - Performance: median latency/PR, peak memory, token usage (if applicable).
- Reproducibility: fixed seeds, pinned model tags, 5-run medians with IQR.
- Commands:
  - python app/eval/seed_bugs.py --repo <path> --mutations 100
  - python app/eval/harness.py --suite basic --out metrics.duckdb

## CI/CD
- GitHub Actions workflow:
  - On pull_request: run linters → LLM review job (optional) → patch validation stage.
  - Upload SARIF (code scanning) and Markdown summary as artifacts.
- Docker images:
  - api: FastAPI service
  - sandbox: test runner with Python, pytest, coverage

## Dashboards
- Export metrics.duckdb or CSV to Power BI/Tableau/Superset.
- Included SQL: severity mix over time, time-to-first-comment, patch-acceptance rate, latency distributions.

## Security and Privacy
- Local-first inference with Ollama; no code leaves the machine unless cloud mode enabled.
- Redact secrets in diffs; configurable path allow/deny lists.

## Roadmap
- Multi-language support (JS/TS) via tree-sitter grammars and language-specific linters.
- Triage classifier to rank findings by acceptance likelihood.
- Test generation suggestions and coverage-aware gates.
- Cost/latency profiles: lint-only vs. deep review.

## License
Specify your license (e.g., MIT).

## Acknowledgments
Open-source tools used: Ollama, Ruff, Bandit, MyPy, Black, tree-sitter, DuckDB, FastAPI, Docker.


- Local-first AI code-review bot with Ollama, FastAPI, and linters; inline comments and validated patches.
- Reproducible evaluation harness with seeded bugs; metrics to DuckDB; BI dashboards for PR KPIs.
- Policy-driven gates, Dockerized sandbox, and CI integration for automated PR review.
