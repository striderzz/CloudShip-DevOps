# CLAUDE.md: CloudShip DevOps build guide

This file tells Claude (Claude Code) how to build this project. Read **README.md** first: it is the design spec. This file is the build plan.

## Goal

Deploy the IronPay payments API (`striderzz/IronPay-LedgerAPI`, pinned by commit SHA in `ironpay.ref`) to AWS with infrastructure as code and a secure pipeline:

- Terraform modules with remote state (S3 + DynamoDB lock): VPC with public/private subnets, ALB, ECS Fargate service, RDS PostgreSQL, ECR, Secrets Manager, CloudWatch logs and alarms, least-privilege IAM
- Environments dev and prod from one root module via `dev.tfvars` / `prod.tfvars` and separate state keys
- GitHub Actions: lint/test, build image, Trivy image scan and Checkov IaC scan as blocking gates, push to ECR, deploy to ECS, authenticated with OIDC (no long-lived AWS keys); prod behind a manual approval
- Monitoring: CloudWatch logs and alarms in AWS; Prometheus + Grafana dashboards locally via Docker Compose
- Rollback by redeploying the previous ECS task definition revision (`make rollback`), plus the ECS deployment circuit breaker
- Cost control: everything tears down with `make destroy`; `docs/cost.md` lists free-tier vs paid parts without invented dollar figures

## Tech stack

Terraform 1.x (+ tflint), AWS (VPC, ALB, ECS Fargate, RDS PostgreSQL, ECR, Secrets Manager, CloudWatch, SNS, IAM, S3 + DynamoDB for state), GitHub Actions with OIDC, Docker and Docker Compose, Trivy, Checkov, Prometheus, Grafana, Make and Bash.

## Build order (phases)

1. Scaffold: repo layout, Makefile (`up`, `down`, `test`, `scan`, `plan`, `apply`, `deploy`, `rollback`, `destroy`), `.env.example`, pre-commit with `terraform fmt`/`validate`/`tflint`, CI lint workflow, local `docker-compose.yml` running IronPay + Postgres.
2. Bootstrap and network: `terraform/bootstrap/` (versioned encrypted S3 state bucket, DynamoDB lock table, GitHub OIDC provider, CI plan and deploy roles); `network` module (VPC across two AZs, public/private subnets, NAT gateway, route tables, security groups).
3. AWS runtime: `ecr`, `database` (RDS with managed master password in Secrets Manager), `iam` (execution role, task role), `service` (ECS cluster, task definition, service with circuit breaker, ALB, target group; `ignore_changes = [task_definition]`). First manual deploy to dev and smoke test.
4. Pipeline: `ci.yml` (lint, IronPay tests, build, Trivy, Checkov, `terraform plan` with the read-only role) and `deploy.yml` (gates, push SHA-tagged image to ECR, apply and deploy dev, smoke test, prod after approval). Concurrency group per environment.
5. Monitoring: CloudWatch log group with retention, alarms (ALB 5xx, target response time, unhealthy hosts, ECS CPU/memory, RDS CPU/free storage) to an SNS email topic; Prometheus + Grafana in Docker Compose with a provisioned RED dashboard and one error-rate alert rule.
6. Operations and results: `scripts/rollback.sh` + runbook, `make destroy` verified to leave nothing behind, `docs/cost.md`, measured results (lead time, pipeline duration, rollback time, findings blocked, cost estimate) in `docs/results/`, real screenshots (AWS console, CI run, Grafana).

## Stretch goals

Only after Phase 6 is done and measured:

- Local k3d cluster, Helm chart for IronPay, Argo CD syncing from a `gitops/` folder
- Blue/green deploys on ECS with CodeDeploy
- AI pull-request reviewer: GitHub Action sends `terraform plan` + Trivy/Checkov output to Claude (`claude-haiku-5-5`) and posts one structured risk comment (follow the LLM rules below)
- Infracost cost diff on pull requests that change Terraform

## Working rules for Claude

- Keep the scope at the core level in README.md (stretch goals only after Phase 6): this is a portfolio project for a master's graduate with about a year of experience. Do not add components the README does not list. The owner must be able to explain every line in an interview.
- Build **one phase at a time**, in order. Each phase must run end to end before the next starts.
- After each phase, update the **Roadmap checklist in README.md** and commit with a clear message.
- **Never invent results.** Every number in README.md (accuracy, latency, throughput, cost) must come from a run in this repo. Save the raw output in `docs/results/`.
- **Screenshots:** when a phase produces something visible (UI, dashboard, Grafana, MLflow, test or load-test report), capture real screenshots into `docs/screenshots/` and embed them in README.md. Never mock them up.
- Keep secrets out of git: `.env` (gitignored) plus a committed `.env.example`.
- Never commit datasets or large model files. Write download scripts and gitignore the data folders.
- Most code here is Terraform, YAML and Bash. Pin Terraform and provider versions, run `terraform fmt`, never hard-code account IDs or ARNs, and keep `make destroy` working after every phase. Any Python helper uses Python 3.11+. Dependencies live in a pinned `requirements.txt` (runtime) and `requirements-dev.txt` (pytest, ruff). Setup is `pip install -r requirements.txt`. Use `ruff` for lint/format, type hints, and `pytest` for tests.
- **LLM rules (stretch AI reviewer only; the core uses no LLM):**
  - Every LLM call goes through the `llm/` provider layer: Claude through the official `anthropic` SDK; OpenAI through the official `openai` SDK; a "custom" provider through the `openai` SDK with a user-supplied `base_url`, API key and model name (for Gemini, Groq, DeepSeek, OpenRouter, Ollama and others).
  - Keys come from `.env` (with a committed `.env.example`) locally and from GitHub repository secrets in Actions; provider and model come from repository variables. Keys are never logged.
  - The PR reviewer defaults to `claude-haiku-5-5`; `claude-opus-5-5` is available for hard reasoning steps. Both are configurable.
  - Log tokens and cost for every call, and enforce a monthly budget cap (`LLM_MONTHLY_BUDGET_USD`).
  - Check `stop_reason` (including `refusal`) before reading a response. Use structured outputs for JSON. Put stable prompts first and use prompt caching. Run offline evals with the Message Batches API.
  - Tests use a fake provider, so CI never spends money or needs keys.
- `docker compose up` (via `make up`) must bring up the local demo: IronPay, Postgres, Prometheus, Grafana.
- Never run `terraform apply` against prod outside the approved pipeline, and run `make destroy ENV=dev` at the end of every AWS session.
- Add a GitHub Actions workflow (lint + tests) in Phase 1 and keep it green.
- Prefer small, readable modules. Add a short "How it works" note to `docs/` after each phase.

## Definition of done

- One-command local demo (`make up`) and one-command teardown of AWS (`make destroy`)
- README has the architecture diagram, setup steps, real screenshots, a demo GIF and measured results
- CI is green, Trivy and Checkov gates are enforced, and a rollback has been tested end to end

## Status

Only `README.md`, `INFO.md` and this file exist so far. Start at **Phase 1**.
