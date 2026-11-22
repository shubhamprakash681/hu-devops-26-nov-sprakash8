# Minikube Installation and Setup Guide

This document provides a comprehensive, production-grade guide for installing, configuring, and managing a local Kubernetes cluster using **Minikube** across Windows, Linux, and macOS.

---

## 1. Prerequisites and System Requirements

Before installing Minikube, ensure your host system satisfies the minimum hardware and software prerequisites:

- **CPU**: 2 CPUs or more (4 cores recommended for running multi-tier apps like Frontend, Backend, and PostgreSQL).
- **RAM**: 4 GB of free memory (8 GB recommended).
- **Disk Space**: At least 20 GB of free persistent storage.
- **Virtualization Support**: Hardware virtualization (Intel VT-x or AMD-V) must be enabled in your system BIOS/UEFI.
- **Container / VM Driver**:
  - **Docker Desktop** (Recommended with WSL2 backend on Windows).
  - Hyper-V, VirtualBox, or Podman as alternatives.
- **kubectl**: Kubernetes CLI utility for interacting with the cluster.

To verify virtualization in Windows PowerShell:
```powershell
Get-CimInstance Win32_Processor | Select-Object VirtualizationFirmwareEnabled
```

---

## 2. Installing Minikube on Windows

### Method 1: Windows Package Manager (`winget`) — Recommended
Open PowerShell or Command Prompt as Administrator and run:
```powershell
winget install Kubernetes.minikube
```

### Method 2: Chocolatey Package Manager
If you use Chocolatey:
```powershell
choco install minikube
```

### Method 3: Direct Binary Download (PowerShell)
You can download the official binary directly and add it to your system PATH:
```powershell
# Create a dedicated directory for tools
New-Item -ItemType Directory -Force -Path "C:\minikube"

# Download the latest Windows 64-bit binary
curl.exe -Lo "C:\minikube\minikube.exe" "https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe"

# Add minikube to your User Environment PATH
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\minikube", [EnvironmentVariableTarget]::User)
```

### Method 4: Official Windows Installer (.exe)
1. Download the [Minikube Windows Installer (.exe)](https://github.com/kubernetes/minikube/releases/latest/download/minikube-installer.exe).
2. Run the installer and follow the on-screen setup wizard. It automatically adds Minikube to your system PATH.

---

## 3. Installing Minikube on Linux & macOS

### Linux (x86_64)
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

### macOS (Intel & Apple Silicon)
Using Homebrew:
```bash
brew install minikube
```
Or via direct binary:
```bash
# Apple Silicon (arm64)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube

# Intel (amd64)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

---

## 4. Verifying Installation

Verify that `minikube` is correctly detected:
```powershell
minikube version
```
*Expected Output:*
```text
minikube version: v1.34.0 (or newer)
```

Also verify that `kubectl` is installed and ready:
```powershell
kubectl version --client
```

---

## 5. Starting the Minikube Cluster

### Recommended Startup Command (Docker Driver)
When Docker Desktop is installed, the `docker` driver provides the fastest and most stable local cluster:
```powershell
minikube start --driver=docker --cpus=4 --memory=8192
```

### Alternative: Hyper-V Driver (Windows Pro/Enterprise)
If using native Hyper-V:
```powershell
# Ensure PowerShell is running as Administrator
minikube start --driver=hyperv
```

### Verifying Cluster Health
Once the cluster startup finishes:
```powershell
# Check Minikube status
minikube status

# Check Kubernetes nodes
kubectl get nodes -o wide

# Check core system pods
kubectl get pods -n kube-system
```

---

## 6. Essential Minikube Addons and Features

### 1. Ingress Controller
Enable NGINX Ingress Controller for routing external traffic into cluster services:
```powershell
minikube addons enable ingress
```

### 2. Metrics Server
Enable resource monitoring for horizontal pod autoscaling and `kubectl top`:
```powershell
minikube addons enable metrics-server
```

### 3. Minikube Dashboard
Minikube includes a built-in web-based Kubernetes management dashboard:
```powershell
minikube dashboard
```
*(This automatically opens the dashboard in your default browser).*

### 4. Exposing LoadBalancer Services (`minikube tunnel`)
On Windows and macOS with the Docker driver, `LoadBalancer` services do not automatically get external IPs without a tunnel:
```powershell
minikube tunnel
```
*(Keep this command running in a dedicated terminal window while testing LoadBalancer services).*

---

## 7. Cluster Lifecycle Management

| Action | Command |
|---|---|
| **Stop cluster** (pauses VM/container, preserves state) | `minikube stop` |
| **Resume cluster** | `minikube start` |
| **Pause cluster** | `minikube pause` |
| **Unpause cluster** | `minikube unpause` |
| **Open interactive shell in node** | `minikube ssh` |
| **View logs** | `minikube logs` |
| **Delete cluster** (destroys VM/container and all state) | `minikube delete` |

---

## 8. Troubleshooting Common Issues

### Issue 1: `Exiting due to DRV_DOCKER_NOT_RUNNING`
- **Cause**: Docker daemon is stopped or Docker Desktop is not running.
- **Solution**: Launch Docker Desktop, wait until the engine icon in the taskbar is steady green, then re-run `minikube start --driver=docker`.

### Issue 2: `VT-x / AMD-V is not enabled`
- **Cause**: Hardware virtualization is turned off in motherboard firmware.
- **Solution**: Reboot your laptop, enter BIOS/UEFI settings, find **Intel Virtualization Technology (VT-x)** or **SVM Mode (AMD)**, set to **Enabled**, save and reboot.

### Issue 3: WSL2 / Hyper-V Conflicts
- **Cause**: Mixing Hyper-V and VirtualBox on Windows 10/11.
- **Solution**: Stick to `--driver=docker` which uses Windows Subsystem for Linux 2 (WSL2) seamlessly without Hyper-V switch complications.

### Issue 4: `ImagePullBackOff` for Private Registry Images
- **Cause**: Pod cannot pull container images from private GHCR repositories without credentials.
- **Solution**: Create a Kubernetes `docker-registry` secret:
  ```powershell
  kubectl create secret docker-registry ghcr-secret `
    --docker-server=ghcr.io `
    --docker-username=<github-username> `
    --docker-password=<github-token> `
    --docker-email=<github-email>
  ```
  And specify `imagePullSecrets: - name: ghcr-secret` in the Pod/Deployment specification.
