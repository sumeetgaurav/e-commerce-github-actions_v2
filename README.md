# 🛒 Shoply — E-Commerce DevSecOps Pipeline (GitHub Actions)

[![DevSecOps](https://github.com/sumeetgaurav/e-commerce-github-actions_v2/actions/workflows/devsecops_pipleline.yml/badge.svg)](https://github.com/sumeetgaurav/e-commerce-github-actions_v2/actions/workflows/devsecops_pipleline.yml)

A hands-on learning repo for **GitHub Actions, CI/CD, and DevSecOps**. The app itself — a tiny Flask-based storefront called **Shoply** — is intentionally minimal. The real content is the collection of **17 GitHub Actions workflows** in [`.github/workflows/`](.github/workflows/) that progressively build up from "hello world" basics to a full **shift-left → shift-right DevSecOps pipeline**: lint & test → secrets scan → SAST → dependency scan → container image scan → build & push → deploy → DAST.

---

## 📋 Table of Contents

- [What's inside](#-whats-inside)
- [The application](#-the-application)
- [Project structure](#-project-structure)
- [The DevSecOps pipeline](#-the-devsecops-pipeline)
- [All workflows at a glance](#-all-workflows-at-a-glance)
- [Required secrets & variables](#-required-secrets--variables)
- [Running locally](#-running-locally)
- [Running with Docker](#-running-with-docker)
- [Known gaps / things to be aware of](#-known-gaps--things-to-be-aware-of)

---

## 📦 What's inside

This repository is designed as a step-by-step teaching path for GitHub Actions:

1. **Fundamentals** — workflows, jobs, steps, `needs:`, `workflow_dispatch` vs `push`, secrets & variables, GitHub Pages.
2. **CI/CD** — reusable workflows (`workflow_call`) chained into orchestrators that lint, test, build, push, and deploy.
3. **DevSecOps** — security tooling bolted onto every stage of the pipeline: secret scanning, SAST, dependency (SCA) scanning, container image scanning, and DAST against the live deployed app.

## 🛍️ The application

- **[`app.py`](app.py)** — A single-route Flask app. `GET /` renders `templates/index.html`.
- **[`templates/index.html`](templates/index.html)** — A static, self-contained storefront page ("Shoply") with inline CSS/JS: hero banner, categories, a handful of hardcoded products, and a client-side-only cart (`alert()`-based, no backend logic, no persistence).
- **[`test_app.py`](test_app.py)** — Pytest smoke test asserting `GET /` returns HTTP 200.
- **[`requirements.txt`](requirements.txt)** — `flask`, `gunicorn` (prod server), `pytest` (tests), `ruff` (linter).
- **[`Dockerfile`](Dockerfile)** — `python:3.13-slim` base image; installs requirements and serves the app with `gunicorn` on port 80.
- **[`docker-compose.yml`](docker-compose.yml)** — Runs the published image `sumeetgaurav/e-commerce-python:latest`, mapping port `80:80`. Used on the deploy target (self-hosted runner / EC2 instance), not for local dev builds.

## 🗂️ Project structure

```
e-commerce-github-actions_v2/
├── .github/
│   ├── .gitleaks.toml               # Gitleaks rules/allowlist for secret scanning
│   └── workflows/                   # All 17 GitHub Actions workflows (see below)
├── templates/
│   └── index.html                   # Shoply storefront (static HTML/CSS/JS)
├── app.py                           # Flask app entrypoint
├── test_app.py                      # Pytest tests
├── requirements.txt                 # Python dependencies
├── Dockerfile                       # Container image definition
├── docker-compose.yml               # Deploy-target compose file (pulls published image)
├── sonar-project.properties         # SonarQube/SonarCloud project config
├── .trivyignore                     # CVEs explicitly accepted/ignored by Trivy
└── README.md
```

## 🔐 The DevSecOps pipeline

The centerpiece of this repo is **[`devsecops_pipleline.yml`](.github/workflows/devsecops_pipleline.yml)** — a single orchestrator workflow, triggered on every `push`, that chains together every security gate in a **shift-left → shift-right** flow:

```
push
  │
  ▼
1. secret-scan        (secrets-scanning.yml)   → Gitleaks: block leaked keys/tokens/passwords
  │
  ▼
2. code                (lint_and_test.yml)      → ruff lint + pytest, across Python 3.12 / 3.13 / 3.14
  │
  ▼
3. sast                (sonar_scan.yml)         → SonarQube/SonarCloud static analysis
  │
  ▼
4. dependency-scan      (dependency_scan.yml)    → pip-audit + OWASP Dependency-Check (fails on CVSS ≥ 7)
  │
  ▼
5. build-and-scan       (trivy_scans.yml)        → Build the Docker image, Trivy-scan it for CRITICAL CVEs
  │
  ▼
6. docker-push          (docker_push.yml)        → Push the scanned image to Docker Hub
  │
  ▼
7. deploy               (cd.yml)                 → docker compose up on the self-hosted runner (EC2)
  │
  ▼
8. dast                 (dast.yml)                → OWASP ZAP baseline scan against the live running app
```

Each stage is a **reusable workflow** (`on: workflow_call`) that can also be run independently or composed into a different pipeline — see [`cicd.yml`](.github/workflows/cicd.yml) and [`ci.yml`](.github/workflows/ci.yml) for smaller, non-security-focused variants used earlier in the learning path.

## 📑 All workflows at a glance

| Workflow | Trigger | What it demonstrates / does |
|---|---|---|
| [`hello.yml`](.github/workflows/hello.yml) | `workflow_dispatch` | "Hello world" intro — basic jobs/steps and `needs:` job dependencies. |
| [`env_demo.yml`](.github/workflows/env_demo.yml) | `workflow_dispatch` (environment choice: dev/stg/prd) | `env:` vars, repo **variables** (`vars.*`), **secrets** (`secrets.*`), and `workflow_dispatch` inputs. |
| [`lint_and_test.yml`](.github/workflows/lint_and_test.yml) | `workflow_call` | Matrix-tests Python 3.12 / 3.13 / 3.14 → `ruff check .` → `pytest`. |
| [`secrets-scanning.yml`](.github/workflows/secrets-scanning.yml) | `workflow_call` | Scans the full git history for leaked tokens/keys/passwords using **Gitleaks** (config in `.github/.gitleaks.toml`). |
| [`sonar_scan.yml`](.github/workflows/sonar_scan.yml) | `workflow_call` | **SAST** via SonarQube/SonarCloud (`SonarSource/sonarqube-scan-action`). |
| [`dependency_scan.yml`](.github/workflows/dependency_scan.yml) | `workflow_call` | **SCA**: `pip-audit` + OWASP `dependency-check` (fails on CVSS ≥ 7), across the same Python matrix. |
| [`trivy_scans.yml`](.github/workflows/trivy_scans.yml) | `workflow_call` | Builds the Docker image and scans it with **Trivy**, failing on CRITICAL, fixable OS/library CVEs. |
| [`docker_push.yml`](.github/workflows/docker_push.yml) | `workflow_call` | Builds and pushes the final image to Docker Hub as `<DOCKER_USERNAME>/e-commerce-python:latest`. |
| [`docker.yml`](.github/workflows/docker.yml) | `workflow_call` | Simpler build-and-push workflow used by the non-DevSecOps `ci.yml` / `cicd.yml` pipelines. |
| [`cd.yml`](.github/workflows/cd.yml) | `workflow_call` | **Deploy**: runs `docker compose up -d --build --force-recreate` on a **self-hosted** runner (the target EC2 instance). |
| [`dast.yml`](.github/workflows/dast.yml) | `workflow_call` | **DAST**: OWASP ZAP baseline scan against the live deployed app (`secrets.EC2_HOST`). |
| [`devsecops_pipleline.yml`](.github/workflows/devsecops_pipleline.yml) | `push` | 🔐 The full end-to-end DevSecOps pipeline described above. |
| [`cicd.yml`](.github/workflows/cicd.yml) | `push` | Simpler CI/CD (no security gates): lint/test → build & push → deploy. |
| [`ci.yml`](.github/workflows/ci.yml) | `workflow_dispatch` | CI-only: lint/test → build & push (no deploy). |
| [`cicd_dummy.yml`](.github/workflows/cicd_dummy.yml) | `workflow_dispatch` | Fully mocked pipeline shape (code → build → test → deploy) using only `echo` statements, for teaching pipeline *structure* with no real side effects. |
| [`github_pages.yml`](.github/workflows/github_pages.yml) | `workflow_dispatch` | Publishes the repo's static content to GitHub Pages via `actions/configure-pages` + `upload-pages-artifact` + `deploy-pages`. |

## 🔑 Required secrets & variables

Configure these under **Settings → Secrets and variables → Actions** for the pipelines that need them:

| Name | Type | Used by | Purpose |
|---|---|---|---|
| `DOCKER_USERNAME` | Variable | `docker.yml`, `docker_push.yml`, `trivy_scans.yml` | Docker Hub username / image namespace. |
| `DOCKERHUB_TOKEN` | Secret | `docker.yml`, `docker_push.yml`, `trivy_scans.yml` | Docker Hub access token for `docker login`. |
| `SONAR_TOKEN` | Secret | `sonar_scan.yml` | Auth token for SonarQube/SonarCloud. |
| `SONAR_HOST_URL` | Secret | `sonar_scan.yml` | SonarQube/SonarCloud server URL (e.g. `sonarcloud.io`). |
| `EC2_HOST` | Secret | `dast.yml` | Hostname/IP of the deployed app, used as the ZAP scan target. |
| `TEXT_SECRET` | Secret | `env_demo.yml` | Demo-only secret for the fundamentals workflow. |
| `GITHUB_TOKEN` | Built-in | `secrets-scanning.yml`, `dast.yml` | Provided automatically by GitHub Actions. |

The **self-hosted runner** used by `cd.yml` / `docker.yml` / `docker_push.yml` / `trivy_scans.yml` (labeled `self-hosted`) must be registered against this repo — in practice, an EC2 instance running the GitHub Actions runner agent.

## 💻 Running locally

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Lint
ruff check .

# 3. Run tests
pytest

# 4. Run the dev server
python app.py
# or, closer to production:
gunicorn --bind 0.0.0.0:80 app:app
```

## 🐳 Running with Docker

```bash
# Build and run the local image
docker build -t e-commerce-python .
docker run -p 80:80 e-commerce-python

# ...or pull and run the published image (what docker-compose.yml does)
docker compose up -d
```

Then open **http://localhost** in your browser.

## ⚠️ Known gaps / things to be aware of

- `ci.yml` and `cicd.yml` overlap in purpose; only `devsecops_pipleline.yml` is the "real" security-gated pipeline meant to run on every push. If more than one `push`-triggered workflow is enabled at once, expect duplicate runs.
- `cd.yml`, `docker.yml`, `docker_push.yml`, and `trivy_scans.yml` all require a **self-hosted runner** to be online and registered — they will queue indefinitely without one.
- `.trivyignore` currently suppresses a few CVEs tied to the pinned `python:3.13-slim` base image; revisit when upgrading the base image.
- The storefront cart (`templates/index.html`) is purely client-side/cosmetic — there is no backend cart, checkout, or persistence logic to exercise.
