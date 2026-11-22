# Complete From-Scratch Setup Guide for Windows Laptop
## Track 2: Docker, CI/CD, Kubernetes, and DevSecOps

This guide provides a step-by-step, zero-to-hero procedure to replicate and run the entire assignment (Track 2 Questions 1, 2, and 3) on a **brand new or fresh Windows 10/11 laptop**.

---

## Table of Contents
1. [System Prerequisites & BIOS Verification](#1-system-prerequisites--bios-verification)
2. [Tooling Installation via Winget & PowerShell](#2-tooling-installation-via-winget--powershell)
3. [Docker Desktop & WSL 2 Configuration](#3-docker-desktop--wsl-2-configuration)
4. [GitHub Credentials & Registry Authentication](#4-github-credentials--registry-authentication)
5. [Cloning and Initializing the Repository](#5-cloning-and-initializing-the-repository)
6. [Executing Question 1: Docker Containerization & Compose](#6-executing-question-1-docker-containerization--compose)
7. [Executing Question 2: CI Pipeline & Local Security Scanners](#7-executing-question-2-ci-pipeline--local-security-scanners)
8. [Executing Question 3: Local Kubernetes Cluster Deployment](#8-executing-question-3-local-kubernetes-cluster-deployment)
9. [Verification of Database Persistence and End-to-End Testing](#9-verification-of-database-persistence-and-end-to-end-testing)
10. [Windows-Specific Gotchas & Troubleshooting](#10-windows-specific-gotchas--troubleshooting)

---

## 1. System Prerequisites & BIOS Verification

### Hardware Requirements
- **OS**: Windows 10 (Build 19041+) or Windows 11 (64-bit).
- **RAM**: Minimum 8 GB (16 GB strongly recommended).
- **CPU**: 4+ physical/logical cores with Hardware Virtualization enabled.
- **Disk Space**: At least 30 GB free storage for Docker images, Kind/Minikube node images, and node modules.

### Verify Hardware Virtualization
Open **PowerShell as Administrator** and verify that CPU virtualization is enabled:
```powershell
Get-CimInstance Win32_Processor | Select-Object Name, VirtualizationFirmwareEnabled
```
> [!IMPORTANT]
> If `VirtualizationFirmwareEnabled` is `False`, reboot your machine into BIOS/UEFI (usually by tapping `F2`, `F12`, `Del`, or `Esc` on startup), locate **Intel VT-x** or **AMD SVM (Secure Virtual Machine)**, set it to **Enabled**, save and exit.

---

## 2. Tooling Installation via Winget & PowerShell

Run **PowerShell as Administrator** and install all required command-line utilities using `winget` (Windows Package Manager):

```powershell
# 1. Install Git for Windows
winget install --id Git.Git -e --source winget

# 2. Install Node.js LTS (Node 20+)
winget install --id OpenJS.NodeJS.LTS -e --source winget

# 3. Install Kubernetes CLI (kubectl)
winget install --id Kubernetes.kubectl -e --source winget

# 4. Install Minikube
winget install --id Kubernetes.minikube -e --source winget

# 5. Install Kind (Kubernetes in Docker - alternative local cluster runtime)
# You can install via Chocolatey, Scoop, or download directly:
New-Item -ItemType Directory -Force -Path "C:\tools"
curl.exe -Lo "C:\tools\kind.exe" "https://github.com/kubernetes-sigs/kind/releases/download/v0.27.0/kind-windows-amd64"
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\tools", [EnvironmentVariableTarget]::User)
```

Close and reopen PowerShell to refresh the system `$env:PATH`. Verify installations:
```powershell
git --version
node --version
npm --version
kubectl version --client
minikube version
kind --version
```

---

## 3. Docker Desktop & WSL 2 Configuration

### Step A: Install and Enable WSL 2
Open PowerShell as Administrator and run:
```powershell
wsl --install --no-distribution
wsl --update
wsl --set-default-version 2
```
Restart your computer if prompted by Windows.

### Step B: Install Docker Desktop
Install Docker Desktop via winget:
```powershell
winget install --id Docker.DockerDesktop -e --source winget
```
Alternatively, download the installer directly from [Docker Official Website](https://www.docker.com/products/docker-desktop/).

### Step C: Configure Docker Desktop Settings
1. Launch **Docker Desktop**.
2. Go to **Settings (Gear Icon)** -> **General**:
   - Check **Use the WSL 2 based engine**.
3. Go to **Settings** -> **Resources** -> **WSL Integration**:
   - Check **Enable integration with additional distributions** if using an installed distro like Ubuntu.
4. Verify Docker daemon is running from PowerShell:
```powershell
docker version
docker compose version
```

---

## 4. GitHub Credentials & Registry Authentication

To pull and push images to GitHub Container Registry (`ghcr.io`), you need a **GitHub Personal Access Token (PAT)**:

### Step A: Generate GitHub PAT (Classic)
1. Go to GitHub -> **Settings** -> **Developer settings** -> **Personal access tokens** -> **Tokens (classic)**.
2. Click **Generate new token (classic)**.
3. Name: `hu-devops-laptop-token`.
4. Select Scopes:
   - `repo` (Full control of private repositories)
   - `write:packages` (Upload packages to GitHub Package Registry)
   - `read:packages` (Download packages from GitHub Package Registry)
   - `delete:packages` (Optional)
5. Generate the token and copy the string (e.g. `ghp_xxxxxxxxxxxxxxxxxxxx`).

### Step B: Authenticate Local Docker to GHCR
Open PowerShell and log in to `ghcr.io`:
```powershell
echo "YOUR_GITHUB_PAT" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```
*Expected Output:* `Login Succeeded`

---

## 5. Cloning and Initializing the Repository

Clone the project repository to a development path (e.g., `C:\dev\hu_devops`):

```powershell
# Create parent directory
New-Item -ItemType Directory -Force -Path "C:\dev\hu_devops"
cd C:\dev\hu_devops

# Clone repository
git clone https://github.com/shubhamprakash681/hu-devops-26-nov-sprakash8.git
cd hu-devops-26-nov-sprakash8
```

### Directory Structure Overview
```text
hu-devops-26-nov-sprakash8/
├── .github/
│   └── workflows/
│       └── ci.yaml              # Parameterized CI pipeline with security gates
├── app/
│   ├── be/                      # Node.js Express Backend
│   │   ├── Dockerfile           # Multi-stage hardened Alpine build
│   │   ├── package.json
│   │   ├── src/                 # Application server & database logic
│   │   └── test/                # Native test suite (15 unit tests)
│   └── fe/                      # React + Vite Frontend
│       ├── Dockerfile           # Multi-stage build with unprivileged Nginx
│       ├── nginx.conf           # Reverse proxy routing /api/ to backend
│       └── package.json
├── k8s/                         # Kubernetes deployment manifests
│   ├── backend.yaml             # Backend Deployment, Service, Secret
│   ├── frontend.yaml            # Frontend Deployment, Service
│   ├── postgres.yaml            # PostgreSQL Deployment, Service, PVC (1Gi)
│   └── kustomization.yaml
├── MINIKUBE_INSTALLATION.md     # Dedicated Minikube setup guide
└── README_track_2.md            # Track 2 steps, verification, and issue log
```

---

## 6. Executing Question 1: Docker Containerization & Compose

### Step A: Build Backend & Frontend Images Locally
```powershell
# Build Backend image
docker build -t app-backend:latest ./app/be

# Build Frontend image
docker build -t app-frontend:latest ./app/fe
```

### Step B: Create Root `.env` Configuration
In `hu-devops-26-nov-sprakash8/`, create `.env`:
```ini
POSTGRES_USER=inventory_user
POSTGRES_PASSWORD=inventory_pass_123
POSTGRES_DB=inventory_db
DB_HOST=db
DB_PORT=5432
PORT=8080
JWT_SECRET=dev_jwt_secret_key_super_secure_12345
DB_CONNECT_RETRIES=10
DB_CONNECT_DELAY_MS=2000
SEED_ADMIN_USERNAME=admin
SEED_ADMIN_PASSWORD=admin123
SEED_TRAINEE_USERNAME=trainee
SEED_TRAINEE_PASSWORD=trainee123
```

### Step C: Run with Docker Compose
If `docker-compose.yml` is used:
```powershell
docker compose up -d
docker compose ps
```

### Step D: Test End-to-End Locally
```powershell
# Test backend health
curl.exe http://localhost:8080/health
# Response: {"status":"ok"}

curl.exe http://localhost:8080/health/db
# Response: {"status":"ok","db":"up"}

# Test frontend
curl.exe http://localhost:3000
```

### Step E: Record Final Image Sizes
```powershell
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```
- `app-backend`: ~209 MB
- `app-frontend`: ~74.4 MB
- `postgres:15-alpine`: ~659 MB (or ~170MB for alpine)

Tear down the local containers:
```powershell
docker compose down
```

---

## 7. Executing Question 2: CI Pipeline & Local Security Scanners

Before pushing to GitHub Actions, you can test every security gate locally using Docker containers:

### 1. Dockerfile Linting (Hadolint)
```powershell
docker run --rm -v "${PWD}:/work" hadolint/hadolint:latest hadolint /work/app/be/Dockerfile
docker run --rm -v "${PWD}:/work" hadolint/hadolint:latest hadolint /work/app/fe/Dockerfile
```

### 2. Secrets Scanning (TruffleHog)
```powershell
docker run --rm -v "${PWD}:/pwd" trufflesecurity/trufflehog:latest git file:///pwd --fail --only-verified
```

### 3. Dependency Vulnerability Scanning (Grype SCA)
```powershell
docker run --rm -v "${PWD}:/src" anchore/grype:latest dir:/src/app/be --fail-on high
docker run --rm -v "${PWD}:/src" anchore/grype:latest dir:/src/app/fe --fail-on high
```

### 4. SAST Code Scanning (Semgrep)
```powershell
docker run --rm -v "${PWD}:/src" semgrep/semgrep:latest semgrep scan --config=auto /src/app/be
```

### 5. Container Image Vulnerability Scan (Trivy)
```powershell
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --severity HIGH,CRITICAL app-backend:latest
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --severity HIGH,CRITICAL app-frontend:latest
```

### 6. Pushing to GitHub and Verifying GitHub Actions
Push your changes to trigger the automated CI workflow in `.github/workflows/ci.yaml`:
```powershell
git add .
git commit -m "Trigger CI pipeline"
git push origin main
```
The workflow will:
1. Detect changed paths via `dorny/paths-filter@v3`.
2. Run linting, unit tests, Hadolint, TruffleHog, Grype, Semgrep, and Trivy.
3. Automatically authenticate to GHCR using `${{ secrets.GITHUB_TOKEN }}`.
4. Push images tagged with the commit SHA:
   - `ghcr.io/<owner>/hu-inventory-be:sha-<short-sha>`
   - `ghcr.io/<owner>/hu-inventory-fe:sha-<short-sha>`

---

## 8. Executing Question 3: Local Kubernetes Cluster Deployment

### Step A: Initialize the Local Cluster

#### Option 1: Using Kind (Recommended on Windows with Docker Desktop)
```powershell
# Create Kind cluster named hu-devops
kind create cluster --name hu-devops

# Verify node readiness
kubectl get nodes -o wide
```

#### Option 2: Using Minikube
```powershell
# Start Minikube with Docker driver
minikube start --driver=docker --cpus=2 --memory=4096

# Verify status
minikube status
```

---

### Step B: Create the GHCR `imagePullSecret`
Create the secret so the Kubernetes kubelet can pull private or authenticated images from GitHub Container Registry:

```powershell
kubectl create secret docker-registry ghcr-secret `
  --docker-server=ghcr.io `
  --docker-username="YOUR_GITHUB_USERNAME" `
  --docker-password="YOUR_GITHUB_PAT" `
  --docker-email="YOUR_EMAIL@example.com"
```

---

### Step C: Deploy Manifests
Deploy the multi-tier application stack:

```powershell
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
```

Check the status of all pods:
```powershell
kubectl get pods -o wide
```

---

### Step D: Prove Real Image Pull from GHCR
Inspect the pod events to prove images are genuinely pulled from GHCR:

```powershell
# Describe Backend Pod
kubectl describe pod -l app=backend

# Describe Frontend Pod
kubectl describe pod -l app=frontend
```

Look for events containing:
```text
Normal  Pulling   kubelet  Pulling image "ghcr.io/.../hu-inventory-be:sha-..."
Normal  Pulled    kubelet  Successfully pulled image ... in X.XXs
Normal  Created   kubelet  Created container: backend
Normal  Started   kubelet  Started container backend
```
And verify the running image digest matches the OCI digest in GHCR.

---

## 9. Verification of Database Persistence and End-to-End Testing

### Step A: Database Persistence Test Across Pod Restarts

1. **Insert a custom test item into the running PostgreSQL database**:
```powershell
kubectl exec deployment/postgres -- psql -U inventory_user -d inventory_db -c "INSERT INTO items (name, description) VALUES ('Fresh Laptop Verification', 'Survives pod restarts via PVC');"
```

2. **Verify the record is stored**:
```powershell
kubectl exec deployment/postgres -- psql -U inventory_user -d inventory_db -c "SELECT * FROM items;"
```

3. **Delete the PostgreSQL pod to simulate a failure / restart**:
```powershell
kubectl delete pod -l app=postgres
```

4. **Wait for the replacement pod to become Ready (1/1)**:
```powershell
kubectl get pods -l app=postgres -w
```

5. **Query the database again**:
```powershell
kubectl exec deployment/postgres -- psql -U inventory_user -d inventory_db -c "SELECT * FROM items;"
```
*Confirmation:* The item `Fresh Laptop Verification` is still present, proving persistent storage backed by `postgres-pvc` (1Gi).

---

### Step B: Expose via Port-Forwarding and Verify End-to-End

#### 1. Backend Verification
Open a separate PowerShell window and port-forward the backend service:
```powershell
kubectl port-forward svc/backend 8080:8080
```
In your main terminal, test the endpoints:
```powershell
# Health checks
curl.exe http://127.0.0.1:8080/health
curl.exe http://127.0.0.1:8080/health/db

# Authentication
$login = Invoke-RestMethod -Uri "http://127.0.0.1:8080/api/auth/login" -Method Post -ContentType "application/json" -Body '{"username":"admin","password":"admin123"}'
Write-Host "JWT Token: " $login.token

# Fetch items using token
Invoke-RestMethod -Uri "http://127.0.0.1:8080/api/items" -Headers @{ Authorization = "Bearer $($login.token)" } | Format-Table
```

#### 2. Frontend & Nginx Proxy Verification
Open a separate PowerShell window and port-forward the frontend service:
```powershell
kubectl port-forward svc/frontend 3000:80
```
Open your browser or run in terminal:
```powershell
# Verify React UI loads
curl.exe http://127.0.0.1:3000/

# Verify Nginx reverse proxy routes /api/ to backend:8080
Invoke-RestMethod -Uri "http://127.0.0.1:3000/api/items" | Format-Table
```
Open **`http://localhost:3000`** in Chrome / Edge to interact with the full inventory user interface.

---

## 10. Windows-Specific Gotchas & Troubleshooting

### 1. PowerShell Script Execution Policy
If you get `File cannot be loaded because running scripts is disabled on this system`:
```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

### 2. Line Endings (CRLF vs LF)
Windows Git may convert Linux line breaks (`LF`) to Windows (`CRLF`), causing bash scripts or Docker commands inside containers to fail with `^M: bad interpreter` or `/bin/sh: command not found`.
To prevent this, configure Git globally:
```powershell
git config --global core.autocrlf input
```

### 3. Port Conflicts on Windows
If ports `8080` or `5432` are occupied on your host:
```powershell
# Identify process using the port
Get-NetTCPConnection -LocalPort 8080 -ErrorAction SilentlyContinue | Select-Object LocalPort, OwningProcess

# Stop process if needed
Stop-Process -Id <PID> -Force
```

### 4. Docker Desktop WSL 2 Memory Limits
By default, WSL 2 can consume up to 50% of total host RAM. To restrict it and keep your laptop responsive, create `C:\Users\<YourUsername>\.wslconfig`:
```ini
[wsl2]
memory=6GB
processors=4
swap=2GB
```
Apply with:
```powershell
wsl --shutdown
```

### 5. `kubectl` "Access is denied" on Config
If `kubectl` fails with `open C:\Users\<Username>\.kube\config: Access is denied`, ensure your PowerShell terminal is launched with standard user permissions or elevated permissions matching the owner of the `.kube` folder.

---

### Clean Up / Teardown
When finished testing:
```powershell
# Delete Kubernetes resources
kubectl delete -f k8s/frontend.yaml
kubectl delete -f k8s/backend.yaml
kubectl delete -f k8s/postgres.yaml
kubectl delete secret ghcr-secret

# Delete local Kind cluster
kind delete cluster --name hu-devops

# Or stop Minikube
minikube stop
```
