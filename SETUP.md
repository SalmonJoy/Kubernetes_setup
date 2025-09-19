# 🛠 Pre-Workshop Setup Guide  
**DevOps with Kubernetes (4 Hours, Beginner Friendly)**  

This guide helps you prepare your laptop for the workshop. Follow all steps before the session.  

---

## ✅ System Requirements
- Windows 10/11, macOS, or Linux  
- 8 GB RAM (16 GB recommended)  
- 15–20 GB free disk space  
- Internet connection  

---

## 🔽 Installations

### 1. Docker Desktop
Download and install: https://www.docker.com/products/docker-desktop/  
After installation, open **PowerShell (Windows)** or **Terminal (macOS/Linux)** and run:
```sh
docker --version
```
Expected: `Docker version 27.x`  

---

### 2. kubectl (Kubernetes CLI)
Linux/macOS (Terminal):
```sh
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/$(uname | tr '[:upper:]' '[:lower:]')/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```
Windows (PowerShell as Admin):
```powershell
choco install kubernetes-cli
```
Verify:
```sh
kubectl version --client
```
Expected: `v1.31.0` or later  

---

### 3. kind (Kubernetes in Docker)
Linux/macOS (Terminal):
```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-$(uname)-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/
```
Windows (PowerShell as Admin):
```powershell
choco install kind
```
Verify:
```sh
kind version
```
Expected: `kind v0.23.0`  

---

### 4. Helm (Kubernetes Package Manager)
Linux/macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
Windows (PowerShell as Admin):
```powershell
choco install kubernetes-helm
```
Verify:
```sh
helm version
```
Expected: `v3.15.x`  

---

### 5. Node.js + npm
Download LTS version (20.x): https://nodejs.org/en/download  
Verify:
```sh
node -v
npm -v
```
Expected: `node v20.x`, `npm v10.x`  

---

### 6. Git
Download: https://git-scm.com/downloads  
Verify:
```sh
git --version
```
Expected: `git version 2.45+`  

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
Windows (PowerShell as Admin):
```powershell
choco install k6
```
Verify:
```sh
k6 version
```
Expected: `v0.51.x` or later  

---

## 🔍 Final Precheck
Run the following to ensure everything works:
```sh
docker run hello-world
kind create cluster --name precheck
kubectl get nodes
kind delete cluster --name precheck
```
Expected:  
- Docker prints `Hello from Docker!`  
- Kubernetes shows node `kind-control-plane`  

---

## 🎉 You’re Ready!
Bring your laptop with all tools installed, admin rights (for Docker/Kubernetes), and internet access.  
We’ll build, deploy, scale, and monitor apps on Kubernetes together 🚀  

---
