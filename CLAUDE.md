# CLAUDE.md: Endurance build guide

This file tells Claude (Claude Code) how to build this project. Read **README.md** first: it is the design spec. This file is the build plan.

## Goal

Deploy the Ironbank API to AWS with Terraform (VPC, ECS Fargate behind an ALB, RDS PostgreSQL, ECR, Secrets Manager) and to a local k3d Kubernetes cluster via Helm + Argo CD, with GitHub Actions CI/CD using OIDC, Trivy and Checkov gates, and Prometheus + Grafana monitoring.

## Tech stack

Terraform 1.x, AWS (VPC, ALB, ECS Fargate, RDS, ECR, S3 remote state, Secrets Manager, IAM OIDC), Docker, k3d, Helm, Argo CD, kube-prometheus-stack (Prometheus + Grafana), Trivy, Checkov, GitHub Actions. LLM layer: anthropic and openai Python SDKs, python-dotenv, pydantic-settings.

## Build order (phases)

1. Scaffold: repo layout, Terraform remote state (S3 + lock), pre-commit with `terraform fmt`/`validate`/`tflint`, CI.
2. Provider layer `llm/`: Claude (anthropic SDK), OpenAI (openai SDK), custom OpenAI-compatible endpoint; `.env` + Settings page (or env vars / GitHub secrets where there is no UI) with masked keys and Test connection; per-call token/cost log; monthly budget cap; fake provider for tests.
3. Terraform modules: network (VPC, subnets, NAT or VPC endpoints), RDS, ECR, ECS Fargate service + ALB; deploy Ironbank.
4. GitHub Actions: OIDC role, build + push image, Trivy scan, Checkov scan, `terraform plan` on PR and `apply` on main with approval.
5. Local k3d cluster: Helm chart for Ironbank + Postgres; Argo CD syncing from `gitops/`.
6. kube-prometheus-stack: Grafana dashboard (rate, errors, latency) and one alert rule.
7. AI features: GitHub Action PR reviewer (terraform plan + Checkov + Trivy + Infracost -> structured risk report as one PR comment) and a Grafana alert explainer webhook posting to Slack/Discord; keys from GitHub secrets, provider/model from repository variables.
8. Cost notes and `make destroy`; screenshots (AWS console, Argo CD, Grafana, CI); README results; demo GIF.

## Working rules for Claude

- Keep the scope at the level in README.md: this is a portfolio project for a master's graduate with about a year of experience. Do not add components the README does not list. The owner must be able to explain every line in an interview.
- Build **one phase at a time**, in order. Each phase must run end to end before the next starts.
- After each phase, update the **Roadmap checklist in README.md** and commit with a clear message.
- **Never invent results.** Every number in README.md (accuracy, latency, throughput, cost) must come from a run in this repo. Save the raw output in `docs/results/`.
- **Screenshots:** when a phase produces something visible (UI, dashboard, Grafana, MLflow, test or load-test report), capture real screenshots into `docs/screenshots/` and embed them in README.md. Never mock them up.
- Keep secrets out of git: `.env` (gitignored) plus a committed `.env.example`.
- Never commit datasets or large model files. Write download scripts and gitignore the data folders.
- Python 3.11+. Dependencies live in a pinned `requirements.txt` (runtime) and `requirements-dev.txt` (pytest, ruff). Setup is `pip install -r requirements.txt`. Use `ruff` for lint/format, type hints, and `pytest` for tests.
- **LLM rules:**
  - Every LLM call goes through the `llm/` provider layer: Claude through the official `anthropic` SDK; OpenAI through the official `openai` SDK; a "custom" provider through the `openai` SDK with a user-supplied `base_url`, API key and model name (for Gemini, Groq, DeepSeek, OpenRouter, Ollama and others).
  - Keys come from `.env` (with a committed `.env.example`) or from the app's Settings page. Settings are saved to a gitignored `settings.local.json`, keys are masked in the UI and never logged, and the page has a "Test connection" button.
  - Defaults are `claude-opus-5-5` for main reasoning steps and `claude-haiku-5-5` for bulk steps. Both are configurable per feature.
  - Log tokens and cost for every call, and enforce a monthly budget cap (`LLM_MONTHLY_BUDGET_USD`).
  - Check `stop_reason` (including `refusal`) before reading a response. Use structured outputs for JSON. Put stable prompts first and use prompt caching. Run offline evals with the Message Batches API.
  - Tests use a fake provider, so CI never spends money or needs keys.
- `docker compose up` must bring up the demo locally.
- Add a GitHub Actions workflow (lint + tests) in Phase 1 and keep it green.
- Prefer small, readable modules. Add a short "How it works" note to `docs/` after each phase.

## Definition of done

- One-command local demo (`docker compose up` or the documented command)
- README has the architecture diagram, setup steps, real screenshots, a demo GIF and measured results
- CI is green and tests cover the core logic

## Status

Only `README.md` and this file exist so far. Start at **Phase 1**.
