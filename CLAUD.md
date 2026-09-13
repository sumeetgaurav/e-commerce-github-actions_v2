# CLAUD.md

## What this folder is

A learning/demo project for practicing **GitHub Actions CI/CD fundamentals**, using a tiny Flask "e-commerce" storefront (Shoply) as the sample app to build, test, containerize, and deploy.

The app itself is intentionally minimal — the real content of this repo is the collection of GitHub Actions workflows in `.github/workflows/`, each demonstrating a specific CI/CD concept (workflows/jobs/steps, reusable workflows, secrets/variables, Docker builds, self-hosted runner deploys, GitHub Pages, workflow_dispatch inputs, etc.).

## The application

- **`app.py`** — A single-route Flask app. `GET /` renders `templates/index.html`.
- **`templates/index.html`** — A static, self-contained storefront page ("Shoply") with inline CSS/JS: hero banner, categories, a few hardcoded products, and a client-side-only cart (JS `alert()`s, no backend logic, no persistence).
- **`test_app.py`** — One pytest test asserting `GET /` returns HTTP 200.
- **`requirements.txt`** — `flask`, `gunicorn` (prod server), `pytest` (tests), `ruff` (linter).
- **`Dockerfile`** — `python:3.13-slim` base, installs requirements, runs the app with `gunicorn` on port 80.
- **`docker-compose.yml`** — Runs the prebuilt image `sumeetgaurav/e-commerce-python:latest`, mapping port 80:80. Used on the deploy target (self-hosted runner / EC2), not for local dev builds.

## GitHub Actions workflows (`.github/workflows/`)

| File | Trigger | Purpose |
|---|---|---|
| `hello.yml` | `workflow_dispatch` | "Hello world" intro workflow — echoes a greeting and the repo name; demonstrates basic jobs/steps and `needs:` job dependencies. |
| `env_demo.yml` | `workflow_dispatch` (with `environment` choice input: dev/stg/prd) | Demonstrates `env:` vars, repo **variables** (`vars.*`), **secrets** (`secrets.*`), and `workflow_dispatch` inputs. |
| `lint_and_test.yml` | `workflow_call` (reusable only) | Matrix-tests across Python 3.12/3.13/3.14: checkout → setup Python → install deps → `ruff check .` → `pytest`. |
| `docker.yml` | `workflow_call` (reusable only) | Builds the Docker image and pushes it to Docker Hub as `<DOCKER_USERNAME>/e-commerce-python:latest`, using `docker/build-push-action` and Docker Buildx. Runs on a **self-hosted** runner. |
| `cd.yml` | `workflow_call` (reusable only) | Deploy step: checks out code and runs `docker compose up -d --build --force-recreate` on a **self-hosted** runner (i.e. the target EC2 instance registered as a runner). |
| `ci.yml` | `push` | Orchestrator: calls `lint_and_test.yml`, then (on success) `docker.yml`. **Does not** call `cd.yml` — no deploy step wired in here. |
| `cicd.yml` | `push` | Fuller orchestrator: calls `lint_and_test.yml` → `docker.yml` → `cd.yml` in sequence, i.e. lint/test → build & push image → deploy. This is the "real" end-to-end pipeline. |
| `cicd_dummy.yml` | `workflow_dispatch` | Simplified/fake pipeline (code → build → test → deploy) using only `echo` statements and `docker/setup-buildx-action`, for teaching pipeline *shape* without real side effects. |
| `github_pages.yml` | `workflow_dispatch` | Publishes the repo's static content to GitHub Pages via `actions/configure-pages` + `upload-pages-artifact` + `deploy-pages`. |

### Notes / things to be aware of
- `ci.yml` and `cicd.yml` overlap significantly (both trigger on `push` and both run lint/test + docker build). `cicd.yml` additionally wires in the deploy (`cd.yml`) job that `ci.yml` omits — worth clarifying which one is meant to be the active pipeline if both are enabled, since both trigger on every push.
- `docker.yml` and `cd.yml` both require a **self-hosted runner** (labeled `self-hosted`), implying an EC2 instance (per `cd.yml`'s comment and recent commit history — "deploying to EC2") is registered as a GitHub Actions runner.
- `docker.yml` expects repo-level `vars.DOCKER_USERNAME` and `secrets.DOCKERHUB_TOKEN` to be configured for pushing to Docker Hub.
- `env_demo.yml` expects a `secrets.TEXT_SECRET` for its demo output.

## Typical local commands

```bash
pip install -r requirements.txt
ruff check .          # lint
pytest                # test
python app.py         # run dev server (note: no debug/run block currently defined beyond Flask default)
docker build -t e-commerce-python .
docker compose up -d  # runs the published image from Docker Hub, not the local build
```
