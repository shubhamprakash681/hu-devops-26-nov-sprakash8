
# Track 2: Docker, CI/CD, Kubernetes, and DevSecOps & Cyber Security
# Track 2 Q1 Docker Containerization - Steps

## Question 1: Containerize the Application (Docker)
**Part A: Dockerfiles, Docker Compose and Image Sizes**

This document lists the steps followed for each task. The screenshot for each task is stored in the repository as `t2q1a<task>.png`.

---
## Task 1: Write the Backend (BE) Dockerfile

### Steps

1. Go to the backend folder and create a `.dockerignore` file (exclude `node_modules`, `.git`, `.env`).

2. Create a `Dockerfile` using a small, pinned base image (for example `node:20-alpine`).

3. Use a multi-stage build and install only production dependencies with `npm ci --omit=dev`.

4. Copy `package*.json` before the source code to use the layer cache.

5. Run as a non-root user with `USER node`, add `EXPOSE <backend-port>` and set the start command with `CMD`.
---

## Task 2: Write the Frontend (FE) Dockerfile


### Steps

1. Go to the frontend folder and create a `.dockerignore` file (exclude `node_modules`, `build`, `.git`, `.env`).

2. Create a `Dockerfile` with a multi-stage build. Stage 1 uses `node:20-alpine` to run `npm ci` and `npm run build`.

3. In stage 2, use `nginxinc/nginx-unprivileged:alpine` and copy only the build output into the Nginx web root.

4. Add an `nginx.conf` that proxies `/api` to the backend service, and add `EXPOSE 8080`.

---

## Task 3: Create the `.env` and Docker Compose files

### Steps

1. In the project root, create a `.env` file with `POSTGRES_USER`, `POSTGRES_PASSWORD` and `POSTGRES_DB`. Add `.env` to `.gitignore`.

2. Create `docker-compose.yml` with three services: `db` (`postgres:16-alpine`), `backend` and `frontend`.

3. For `db`, load the credentials from `.env`, mount a named volume `pgdata` and add a `pg_isready` healthcheck.

4. For `backend`, build from `./backend`, set the DB host to `db` and use `depends_on` with `condition: service_healthy`.

5. For `frontend`, build from `./frontend` and map a host port to container port 8080.

6. Name the images (for example `myapp-backend:1.0`, `myapp-frontend:1.0`) and validate the file with `docker compose config`.

---
## Task 4: Build the images and run with PostgreSQL

### Steps

1. Build both images with `docker compose build`.

2. Start all the containers with `docker compose up -d`.

3. Check the status with `docker compose ps`. `db` should be healthy and the other two should be running.

4. If a container fails, check `docker compose logs <service-name>`.

---
## Task 5: Test the application end to end

### Steps

1. Test the backend with `curl http://localhost:<backend-port>/health`.

2. Open `http://localhost:<frontend-port>` in a browser and use the main features (create, view, update, delete).

3. Confirm the data is in PostgreSQL with `docker exec -it <db-container> psql -U <user> -d <db> -c "SELECT * FROM <table_name>;"`.

4. Run `docker compose down`, then `docker compose up -d`. Reload the page and confirm the data is still there.
---

## Task 6: Check and record the size of each image

### Steps

1. Run `docker images` to list all images and their sizes.

2. For a cleaner view, run `docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"`.

3. (Optional) Use `docker history <image>:<tag>` to see which layers take the most space.

4. Record the sizes below.

| Image | Tag | Final Size |
|-------|-----|------------|
| app-backend (myapp-backend) | latest / 1.0 | 209MB |
| app-frontend (myapp-frontend) | latest / 1.0 | 74.4MB |
| postgres | 15 / 16-alpine | 659MB |

---

# Track 2 Q2 Continuous Integration (GitHub Actions) - Steps

## Question 2: Continuous Integration (GitHub Actions)
**Part A: CI Pipeline, Security Gates, Scoped Triggers, and GHCR Publishing**

This section describes the automated CI pipeline implemented in `.github/workflows/ci.yaml` for both the Frontend (FE) and Backend (BE).

---

## Task 1: Pipeline Triggers and Path Scoping

### Steps

1. Configure GitHub Actions workflow triggers on `push` and `pull_request` targeting the `main` branch.
2. Implement change detection using `dorny/paths-filter@v3` in a dedicated `detect-changes` job.
3. Define granular path filters:
   - `be`: monitors changes in `app/be/**` and `.github/workflows/ci.yaml`.
   - `fe`: monitors changes in `app/fe/**` and `.github/workflows/ci.yaml`.
4. Add conditional execution gates to downstream jobs:
   - `backend-ci` executes only when `needs.detect-changes.outputs.be == 'true'` (or manual `workflow_dispatch`).
   - `frontend-ci` executes only when `needs.detect-changes.outputs.fe == 'true'` (or manual `workflow_dispatch`).
5. When only FE files are modified, the backend pipeline is skipped; when only BE files are modified, the frontend pipeline is skipped.

---

## Task 2: Parameterize Environment Variables and Pipeline Configuration

### Steps

1. Eliminate all hardcoded environment values and magic strings from workflow steps.
2. Define centralized workflow-level `env` variables:
   - `REGISTRY`: `ghcr.io` (GitHub Container Registry)
   - `BE_IMAGE_NAME`: `hu-inventory-be`
   - `FE_IMAGE_NAME`: `hu-inventory-fe`
   - `BE_DIR`: `app/be`
   - `FE_DIR`: `app/fe`
   - `NODE_VERSION`: `'18'`
   - `TRIVY_SEVERITY`: `'HIGH,CRITICAL'`
   - `GRYPE_SEVERITY`: `'high'`
   - `REPORTS_DIR`: `reports/scans`
3. Parameterize dynamic metadata per run using GitHub context variables (`${{ github.sha }}`, `${{ github.repository }}`, `${{ github.actor }}`).
4. Normalize repository name to lowercase at runtime (`REPO_LOWER=$(echo "${{ github.repository }}" | tr '[:upper:]' '[:lower:]')`) to guarantee standard OCI and GHCR compliance.

---

## Task 3: Implement Automated Lint, Unit Test, and Build Stages

### Steps

1. **Backend (`backend-ci`)**:
   - Check out code with full history (`actions/checkout@v4` with `fetch-depth: 0`).
   - Setup Node.js runtime using pinned version `${{ env.NODE_VERSION }}` with npm dependency caching (`actions/setup-node@v4`).
   - Run clean dependency installation using `npm ci`.
   - Execute native test suite using `npm test` (`node --test test/`), validating all 15 unit tests across endpoints (`/health`, `/health/db`, auth, items, greet, calc, redirect).
2. **Frontend (`frontend-ci`)**:
   - Check out repository and configure Node.js environment.
   - Run clean dependency installation with `npm ci`.
   - Test production build compilation using `npm run build` with Vite, ensuring no bundling or syntax errors exist.

---

## Task 4: Integrate Security Scans and Strict Failure Gates

### Steps

1. **Dockerfile Linting (Hadolint)**:
   - Execute `hadolint/hadolint-action@v3.1.0` against `app/be/Dockerfile` and `app/fe/Dockerfile`.
   - Set `failure-threshold: error` and export results in JSON format to `${{ env.REPORTS_DIR }}/hadolint-*.json`.
2. **Secrets Scanning (TruffleHog)**:
   - Execute `trufflesecurity/trufflehog@main` scanning the repository commit history for verified credentials.
   - Enforce `--only-verified` to validate active leaked credentials without false alarms.
3. **Software Composition Analysis (Grype SCA)**:
   - Run `anchore/scan-action@v3` scanning backend and frontend dependencies.
   - Configure `severity-cutoff: high` and `fail-build: true` to break the pipeline on any High or Critical dependency vulnerability.
   - Export reports in SARIF format (`grype-be.sarif`, `grype-fe.sarif`).
4. **Static Application Security Testing (Semgrep SAST)**:
   - Execute Semgrep container scanning application source code with `--config=auto` and `--error`.
   - Export findings to SARIF format (`semgrep-be.sarif`).
5. **Container Image Vulnerability Scan (Trivy)**:
   - Build the container image locally first, tagged with the commit SHA (`${{ github.sha }}`).
   - Run `aquasecurity/trivy-action@0.29.0` against the freshly built local image before any push occurs.
   - Generate full SARIF report (`trivy-be.sarif`, `trivy-fe.sarif`).
   - Enforce security gate with `exit-code: '1'` on `severity: 'HIGH,CRITICAL'`.
6. **Strict Gate Policy**:
   - Enforce zero tolerance: strictly NO `continue-on-error`, swallowed exit codes (`|| true`), or ignore files to hide scan failures.

---

## Task 5: Authenticate with GHCR and Push Images Tagged with Commit SHA

### Steps

1. Authenticate to GitHub Container Registry (`ghcr.io`) using `docker/login-action@v3` with `${{ secrets.GITHUB_TOKEN }}`.
2. Restrict push actions to verified events (`if: github.event_name != 'pull_request'`), ensuring pull requests run build and scan validation without pushing untrusted artifacts.
3. Adhere to standard tagging practices:
   - Primary immutable tag: Full 40-character commit SHA (`${{ github.sha }}`).
   - Secondary semantic tag: Short commit SHA (`sha-${GITHUB_SHA::7}`).
   - Explicitly avoid the `latest` tag.
4. Push only after all prior gates (lint, test, Dockerfile lint, TruffleHog, Grype, Semgrep, and Trivy) succeed.

---

## Task 6: Publish Tool-Native Reports and Record Stored GHCR Images

### Steps

1. Use `actions/upload-artifact@v4` with `if: always()` to archive all generated SARIF and JSON scan reports (`hadolint`, `trivy`, `grype`, `semgrep`).
2. Write execution metadata and image identifiers directly to `$GITHUB_STEP_SUMMARY`.
3. Record the generated image URIs and tag patterns in the registry record table below:

### GHCR Stored Images Record

| Component | Target Registry | Full Image Reference (Commit SHA Tag) | Semantic Short Tag |
|---|---|---|---|
| **Backend** | `ghcr.io` | `ghcr.io/<owner>/hu-inventory-be:<commit-sha>` | `ghcr.io/<owner>/hu-inventory-be:sha-<short-sha>` |
| **Frontend** | `ghcr.io` | `ghcr.io/<owner>/hu-inventory-fe:<commit-sha>` | `ghcr.io/<owner>/hu-inventory-fe:sha-<short-sha>` |

*(Note: In automated runs, `<owner>` is automatically resolved from `${{ github.repository }}` in lowercase, and `<commit-sha>` matches `${{ github.sha }}` without using `latest`)*

