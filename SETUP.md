# Pre-Workshop Setup Guide  
**DevOps with Kubernetes (4 Hours, Beginner Friendly)**  

This guide helps you prepare your laptop for the workshop. Follow all steps before the session.  

---

## ✅ System Requirements
- Windows 10/11, macOS, or Linux  
- 8 GB RAM (16 GB recommended)  
- 15–20 GB free disk space  
- Internet connection  

---

## CPU Architecture (ARM vs AMD/Intel)
Some laptops (like newer MacBooks with Apple Silicon M1/M2/M3) use **ARM64 architecture**, while most Windows and older macOS/Linux laptops use **AMD64/Intel (x86_64)**.  
This matters when downloading binaries (like `kubectl`, `kind`) or running Docker images.  

### How to Check Your Architecture
- **Linux / macOS (Terminal):**
  ```sh
  uname -m
  ```
  - `x86_64` → AMD/Intel (use amd64 binaries)  
  - `arm64` or `aarch64` → Apple Silicon / ARM (use arm64 binaries)  

- **Windows (PowerShell):**
  ```powershell
  echo $env:PROCESSOR_ARCHITECTURE
  ```
  - `AMD64` → Intel/AMD  
  - `ARM64` → ARM  

### Where You Need to Take Care
- **kubectl downloads:** Replace `amd64` with `arm64` if on ARM. Example for macOS ARM:
  ```sh
  curl -LO "https://dl.k8s.io/release/v1.31.0/bin/darwin/arm64/kubectl"
  ```
- **kind binaries:** kind also provides separate builds for arm64. Replace `amd64` with `arm64` in the download URL. Example:
  ```sh
  curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-darwin-arm64
  ```
- **Docker images:** Most official images (Node, Postgres, NGINX, etc.) now support multi-arch (they auto-detect ARM vs AMD). No special changes required.  
- **Helm, Git, k6:** Installers auto-detect architecture — no manual change needed.  

✅ If you follow ARM-specific binaries for `kubectl` and `kind`, everything else should “just work.”  

---

## Installations

### 1. Docker Desktop
Download and install: https://www.docker.com/products/docker-desktop/  
After installation, open **PowerShell (Windows)** or **Terminal (macOS/Linux)** and run:
```sh
docker --version
```
✅ Expected: `Docker version 27.x` or newer (e.g., 28.4.0 is fine).  

> ⚠️ Always ensure **Docker Desktop is opened and running** (look for the 🐳 whale icon in system tray/menu bar) **before running Kubernetes checks**.

---

### Note for Windows Users (Package Managers)
Some instructions below use **Chocolatey (`choco`)**.  
If you see `choco is not recognized`, you can:  

- **Option 1: Install Chocolatey (recommended)**  
  Open **PowerShell as Administrator** and run:  
  ```powershell
  Set-ExecutionPolicy Bypass -Scope Process -Force; `
    [System.Net.ServicePointManager]::SecurityProtocol = `
    [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; `
    iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
  ```  
  Then restart PowerShell and test:  
  ```powershell
  choco --version
  ```  

- **Option 2: Use Scoop**  
  Install Scoop in PowerShell:  
  ```powershell
  Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
  irm get.scoop.sh | iex
  ```  
  Then install tools:  
  ```powershell
  scoop install kubernetes-cli
  scoop install kind
  scoop install helm
  ```  

- **Option 3: Manual Download**  
  - kubectl: https://kubernetes.io/releases/download/  
  - kind: https://kind.sigs.k8s.io/  
  - helm: https://helm.sh/  
  After download, move the `.exe` into `C:\Windows\System32` or add its folder to PATH.  

---

### 2. kubectl (Kubernetes CLI)
Linux/macOS (Terminal) — replace `amd64` with `arm64` if on ARM:
```sh
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/$(uname | tr '[:upper:]' '[:lower:]')/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```
Windows (PowerShell as Admin, if using Chocolatey):  
```powershell
choco install kubernetes-cli
```
Verify:
```sh
kubectl version --client
```
✅ Expected: `v1.31.x` or newer (e.g., v1.32.2 is fine).  

---

### 3. kind (Kubernetes in Docker)
Linux/macOS (Terminal) — replace `amd64` with `arm64` if on ARM:
```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-$(uname)-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/
```
Windows (PowerShell as Admin, if using Chocolatey):  
```powershell
choco install kind
```
Verify:
```sh
kind version
```
✅ Expected: `v0.23.x` or newer (e.g., v0.30.0 is fine).  

---

### 4. Helm (Kubernetes Package Manager)
Linux/macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
Windows (PowerShell as Admin, if using Chocolatey):  
```powershell
choco install kubernetes-helm
```
Verify:
```sh
helm version
```
✅ Expected: `v3.15.x` or newer (e.g., v3.18.6 is fine).  

---

### 5. Node.js + npm
Download LTS version (20.x): https://nodejs.org/en/download  
Verify:
```sh
node -v
npm -v
```
✅ Expected: `node v20.x`, `npm v10.x` (higher versions are fine if LTS).  

---

### 6. Git
Download: https://git-scm.com/downloads  
Verify:
```sh
git --version
```
✅ Expected: `git version 2.45+` (higher is fine).  

---

### 7. k6 (Load Testing Tool)
Linux (Debian/Ubuntu, Terminal):
```sh
curl -s https://dl.k6.io/key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/k6-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt update && sudo apt install k6
```
macOS (Terminal, Homebrew):
```sh
brew install k6
```
Windows (PowerShell as Admin, if using Chocolatey):  
```powershell
choco install k6
```
Verify:
```sh
k6 version
```
✅ Expected: `v0.51.x` or newer (e.g., v1.2.2 is fine).  

---

## Final Precheck

> ⚠️ **Make sure Docker Desktop is open and running** before running these checks.

```sh
docker run hello-world
kind create cluster --name precheck
kubectl get nodes
kind delete cluster --name precheck
```

**Expected:**  
- Docker prints `Hello from Docker!`  
- Kubernetes shows node `kind-control-plane` with STATUS = Ready  

---

## 🧭 Quick Architecture Cheat‑Sheet (ARM vs AMD/Intel)
| OS | Command | Sample Output | Arch | Use These Downloads | Notes |
|---|---|---|---|---|---|
| macOS Intel | `uname -m` | `x86_64` | amd64 | kubectl: darwin/amd64 • kind: kind-darwin-amd64 | Homebrew auto-detects |
| macOS ARM (M1/M2/M3) | `uname -m` | `arm64` | arm64 | kubectl: darwin/arm64 • kind: kind-darwin-arm64 | Re-download if wrong arch |
| Linux Intel | `uname -m` | `x86_64` | amd64 | kubectl: linux/amd64 • kind: kind-linux-amd64 | Most laptops/desktops |
| Linux ARM | `uname -m` | `aarch64` | arm64 | kubectl: linux/arm64 • kind: kind-linux-arm64 | Common on Raspberry Pi |
| Windows Intel | PowerShell: `echo $env:PROCESSOR_ARCHITECTURE` | `AMD64` | amd64 | choco/scoop auto-detect | |
| Windows ARM | PowerShell: `echo $env:PROCESSOR_ARCHITECTURE` | `ARM64` | arm64 | Use ARM builds or WSL | If tool lacks ARM build, use WSL |

---

## You’re Ready!
Bring your laptop with all tools installed, admin rights (for Docker/Kubernetes), and internet access.  
We’ll build, deploy, scale, and monitor apps on Kubernetes together

---
