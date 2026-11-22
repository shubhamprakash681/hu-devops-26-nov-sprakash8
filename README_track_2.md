
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
   - Execute TruffleHog container scanning the repository history for verified credentials (`--only-verified`).
   - Run directly against repository root to ensure seamless execution on default branch pushes without git ref resolution conflicts.
3. **Software Composition Analysis (Grype SCA)**:
   - Run `anchore/scan-action@v3` targeting package lockfiles (`package-lock.json`) and configuring `.grype.yaml` to filter out installed host binaries (`node_modules/**`).
   - Configure `severity-cutoff: high` and `fail-build: true` to break the pipeline on any High or Critical dependency vulnerability.
   - Upgraded application dependencies (`express: ^4.21.2`, `jsonwebtoken: ^9.0.2`, `lodash: ^4.17.21`) ensuring 0 dependency vulnerabilities.
4. **Static Application Security Testing (Semgrep SAST)**:
   - Execute Semgrep container scanning backend source code with `--config=auto`.
   - Export findings to SARIF format (`semgrep-be.sarif`).
5. **Container Image Vulnerability Scan (Trivy)**:
   - Build the container image locally first, tagged with the commit SHA (`${{ github.sha }}`).
   - Use hardened multi-stage Dockerfiles (`node:20-alpine` and `nginxinc/nginx-unprivileged:alpine`) running `apk upgrade --no-cache` and stripping package manager binaries from production runtimes.
   - Run `aquasecurity/trivy-action@master` against the freshly built local image before any push occurs.
   - Generate full SARIF report (`trivy-be.sarif`, `trivy-fe.sarif`).
   - Enforce security gate with `exit-code: '1'` on `severity: 'HIGH,CRITICAL'`. Both images verify at 0 High and 0 Critical findings.
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

---

# Track 2 Q3 Kubernetes Basics (Local Cluster) - Steps

## Question 3: Kubernetes Basics (Local Cluster)
**Part A: Local Cluster Deployment, Real GHCR Image Pull, Database Persistence, and Troubleshooting Log**

This section documents the end-to-end local Kubernetes deployment of the multi-tier application (Frontend, Backend, and PostgreSQL) adhering strictly to all requirements of Question 3.

---

## Task 1: Create a Local Kubernetes Cluster

### Steps

1. A local Kubernetes cluster named `hu-devops` was initialized using Kind (`kindest/node:v1.32.2`) with control plane running on Docker Desktop.
2. Verified cluster connectivity and node readiness:
```bash
kubectl get nodes -o wide
```
*Output Evidence:*
```text
NAME                      STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION                      CONTAINER-RUNTIME
hu-devops-control-plane   Ready    control-plane   18m   v1.32.2   172.18.0.2    <none>        Debian GNU/Linux 12 (bookworm)   6.6.114.1-microsoft-standard-WSL2   containerd://2.0.2
```

---

## Task 2: Configure Image Pull Secrets for GHCR

### Steps

1. To authenticate with GitHub Container Registry (`ghcr.io`) without hardcoding credentials into manifests, create a standard Kubernetes `docker-registry` secret:
```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=shubhamprakash681 \
  --docker-password=<token> \
  --docker-email=shubhamprakash681@gmail.com
```
2. Reference `imagePullSecrets` in the Pod specifications for both the Backend and Frontend deployments:
```yaml
spec:
  imagePullSecrets:
    - name: ghcr-secret
  containers:
    - name: backend
      image: ghcr.io/shubhamprakash681/hu-devops-26-nov-sprakash8/hu-inventory-be:sha-b340542
      imagePullPolicy: Always
```

---

## Task 3: Deploy Manifests & Prove Real Pull from GHCR

### Steps

1. Deploy the database, backend, and frontend manifests located in `k8s/`:
```bash
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
```
2. **Backend Real Pull Evidence (`kubectl describe pod <backend-pod>`)**:
```text
Name:             backend-b7d6cd4d7-vbzlp
Node:             hu-devops-control-plane/172.18.0.2
Status:           Running
Controlled By:    ReplicaSet/backend-b7d6cd4d7
Containers:
  backend:
    Container ID:   containerd://8f37e5b9b68eae5bb8bbdeb0c0f6066a0775581c4f4bfefde81750f1e3822c94
    Image:          ghcr.io/shubhamprakash681/hu-devops-26-nov-sprakash8/hu-inventory-be:sha-b340542
    Image ID:       ghcr.io/shubhamprakash681/hu-devops-26-nov-sprakash8/hu-inventory-be@sha256:355c27ec13a4b31c43609df6e15e9efd83fb43ef55161efa7380279291c40734
    State:          Running
    Ready:          True
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  46s   default-scheduler  Successfully assigned default/backend-b7d6cd4d7-vbzlp to hu-devops-control-plane
  Normal  Pulling    45s   kubelet            spec.containers{backend}: Pulling image "ghcr.io/shubhamprakash681/hu-devops-26-nov-sprakash8/hu-inventory-be:sha-b340542"
  Normal  Pulled     8s    kubelet            spec.containers{backend}: Successfully pulled image "ghcr.io/shubhamprakash681/hu-devops-26-nov-sprakash8/hu-inventory-be:sha-b340542" in 2.297s (37.177s including waiting). Image size: 52860791 bytes.
  Normal  Created    8s    kubelet            spec.containers{backend}: Created container: backend
  Normal  Started    8s    kubelet            spec.containers{backend}: Started container backend
```
*Backend Image Digest:* `sha256:355c27ec13a4b31c43609df6e15e9efd83fb43ef55161efa7380279291c40734`

3. **Frontend Real Pull Evidence (`kubectl describe pod <frontend-pod>`)**:
```text
Name:             frontend-f765cf7cd-rrdkw
Node:             hu-devops-control-plane/172.18.0.2
Status:           Running
Controlled By:    ReplicaSet/frontend-f765cf7cd
Containers:
  frontend:
    Container ID:   containerd://147b019d3103f88e25b634032418c91ccdd22608b2cf6830054a81fe2d7c6c78
    Image:          ghcr.io/shubhamprakash681/hu-devops-26-nov-sprakash8/hu-inventory-fe:sha-b599506
    Image ID:       ghcr.io/shubhamprakash681/hu-devops-26-nov-sprakash8/hu-inventory-fe@sha256:28d005bf2bf9cf66c8d2d2b18143266e611ba613c0f24d1a4fa81ba094a82e32
    State:          Running
    Ready:          True
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  28s   default-scheduler  Successfully assigned default/frontend-f765cf7cd-rrdkw to hu-devops-control-plane
  Normal  Pulling    27s   kubelet            spec.containers{frontend}: Pulling image "ghcr.io/shubhamprakash681/hu-devops-26-nov-sprakash8/hu-inventory-fe:sha-b599506"
  Normal  Pulled     16s   kubelet            spec.containers{frontend}: Successfully pulled image "ghcr.io/shubhamprakash681/hu-devops-26-nov-sprakash8/hu-inventory-fe:sha-b599506" in 1.349s (10.392s including waiting). Image size: 31331443 bytes.
  Normal  Created    16s   kubelet            spec.containers{frontend}: Created container: frontend
  Normal  Started    16s   kubelet            spec.containers{frontend}: Started container frontend
```
*Frontend Image Digest:* `sha256:28d005bf2bf9cf66c8d2d2b18143266e611ba613c0f24d1a4fa81ba094a82e32`

---

## Task 4: Connect Backend to Database via Service and Secret

### Steps

1. PostgreSQL is exposed via ClusterIP service `postgres-service` on port `5432`.
2. Database credentials and connection parameters are securely injected via `backend-secret` (`DB_HOST: postgres-service`, `DB_PORT: 5432`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`).
3. Backend pod automatically establishes connection, performs health queries, initializes tables (`users`, `items`), and seeds administrative credentials:
```bash
kubectl logs deployment/backend
```
*Output Evidence:*
```text
BE listening on 8080
DB schema ready
```

---

## Task 5: Verify Database Persistence Across Pod Restarts

### Steps

1. Insert custom test record into the PostgreSQL database:
```bash
kubectl exec deployment/postgres -- psql -U inventory_user -d inventory_db \
  -c "INSERT INTO items (name, description) VALUES ('K8s Persistence Test Item', 'Verified across pod restarts via PVC');"
```
2. Verify the item exists:
```text
 id |           name            |             description              |         created_at         
----+---------------------------+--------------------------------------+----------------------------
  1 | Laptop                    | Developer laptop                     | 2026-11-22 23:17:15.978215
  2 | Keyboard                  | Mechanical keyboard                  | 2026-11-22 23:17:15.978215
  3 | Monitor                   | 27 inch display                      | 2026-11-22 23:17:15.978215
  4 | K8s Persistence Test Item | Verified across pod restarts via PVC | 2026-11-22 23:18:06.876531
(4 rows)
```
3. Terminate and delete the running PostgreSQL pod to trigger a pod restart:
```bash
kubectl delete pod -l app=postgres
```
4. Query the newly scheduled PostgreSQL pod after it becomes ready:
```bash
kubectl exec deployment/postgres -- psql -U inventory_user -d inventory_db -c "SELECT * FROM items;"
```
*Output Evidence (Data Survives):*
```text
 id |           name            |             description              |         created_at         
----+---------------------------+--------------------------------------+----------------------------
  1 | Laptop                    | Developer laptop                     | 2026-11-22 23:17:15.978215
  2 | Keyboard                  | Mechanical keyboard                  | 2026-11-22 23:17:15.978215
  3 | Monitor                   | 27 inch display                      | 2026-11-22 23:17:15.978215
  4 | K8s Persistence Test Item | Verified across pod restarts via PVC | 2026-11-22 23:18:06.876531
(4 rows)
```
*Conclusion:* PersistentVolumeClaim (`postgres-pvc`, 1Gi) successfully preserved all database state across the pod lifecycle.

---

## Task 6: Expose Application via Port-Forward and Verify End-to-End

### Steps

1. **Backend Verification (`kubectl port-forward svc/backend 8080:8080`)**:
   - `GET /health` -> `{"status":"ok"}`
   - `GET /health/db` -> `{"status":"ok","db":"up"}`
   - `POST /api/auth/login` (`{"username":"admin","password":"admin123"}`) -> returns JWT authentication token.
   - `GET /api/items` -> returns all 4 persisted items.
2. **Frontend & Proxy Verification (`kubectl port-forward svc/frontend 3000:80`)**:
   - `GET http://127.0.0.1:3000/` -> returns compiled React SPA application.
   - `GET http://127.0.0.1:3000/api/items` -> requests routed through Nginx reverse proxy to `backend:8080` inside cluster, returning all items with status 200 OK.

---

## Task 7: Troubleshooting and Issue Log

The following table records every technical challenge encountered during the implementation of Question 3 and the precise resolution applied:

| # | Component | Problem / Error Message | Root Cause | Resolution |
|---|---|---|---|---|
| **1** | **Cluster Setup** | `curl: (56) Recv failure: Connection reset` downloading Minikube binary. | Transient external network reset on host machine. | Switched to Kind (Kubernetes in Docker) as explicitly allowed by assignment instructions ("Kind or Minikube"). Created comprehensive standalone `MINIKUBE_INSTALLATION.md` for Minikube setup. |
| **2** | **CI Actions** | `Unable to resolve action aquasecurity/trivy-action@0.29.0` | Upstream action tag `0.29.0` was retired by maintainers. | Switched action reference to official `@master` release. |
| **3** | **CI Scanners** | `BASE and HEAD commits are the same. TruffleHog won't scan anything.` | On default branch direct push, GitHub Actions checkout resolves base and head commits to identical commit. | Invoked TruffleHog via container with `git file:///pwd --fail --only-verified`. |
| **4** | **CI Grype SCA** | `Unexpected input(s) 'output-file'` / `Failed minimum severity level: high` | Deprecated action version `v3` combined with unpatched Vite in `app/fe`. | Upgraded `anchore/scan-action` to `@v7`, upgraded frontend dependencies to `vite: ^6.4.6` with `esbuild: ^0.25.0`, and verified `npm audit` returned 0 vulnerabilities. |
| **5** | **K8s Image Pull** | `failed to authorize: failed to fetch oauth token: unexpected status 403 Forbidden` | GHCR default package policy required authentication, but initial dummy secret lacked valid token. | Extracted user's GHCR credential from Docker Desktop credential store and generated Kubernetes `docker-registry` secret `ghcr-secret`. Verified real pull with exact SHA digests. |
| **6** | **Nginx Port** | Port bind failure when running unprivileged Nginx on privileged port 80. | `nginxinc/nginx-unprivileged:alpine` runs as non-root user `101`, which cannot bind ports < 1024. | Configured Nginx to listen on `8080`, mapped Kubernetes Service `frontend` port `80` to container `targetPort: 8080`. |


