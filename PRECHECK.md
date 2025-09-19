# ✅ Environment Precheck – DevOps with Kubernetes Workshop

This file guides you through **one-time prechecks** to ensure your laptop is ready.
Follow the steps in order. Run commands in **PowerShell (Windows)** or **Terminal (macOS/Linux)**.

> Tip: Lines beginning with `#` are comments – you don’t type them.

---

## 0) Identify your OS & CPU Architecture (for downloads)
**macOS/Linux:**
```sh
uname -m
```
- `x86_64` → Intel/AMD (amd64)
- `arm64` or `aarch64` → Apple Silicon / ARM (arm64)

**Windows (PowerShell):**
```powershell
echo $env:PROCESSOR_ARCHITECTURE
```
- `AMD64` → Intel/AMD (amd64)
- `ARM64` → ARM (arm64)

> This matters only when you download binaries manually (e.g., kubectl, kind).

---

## 1) Tool Versions (minimums & examples)

Run the following and check versions meet or exceed the minimums.

```sh
docker --version
kubectl version --client
kind version
helm version
node -v
npm -v
git --version
k6 version
```

**Pass if:**  
- Docker ≥ **27.x** (e.g., 28.4.0 OK)  
- kubectl ≥ **1.31.x** (e.g., 1.32.2 OK)  
- kind ≥ **0.23.x** (e.g., 0.30.0 OK)  
- Helm ≥ **3.15.x** (e.g., 3.18.6 OK)  
- Node LTS **20.x**, npm **10.x**  
- Git ≥ **2.45**  
- k6 ≥ **0.51** (e.g., 1.2.2 OK)

---

## 2) Docker Engine Running

**Start Docker Desktop** (look for the 🐳 whale icon in your system tray/menu bar).  
Then run:

```sh
docker run hello-world
```

**Expected:** output includes “**Hello from Docker!**”.  
If you see an error like *“cannot find //./pipe/dockerDesktopLinuxEngine”*, start Docker Desktop and try again.  
On Windows, ensure **WSL 2** is installed and enabled in **Docker Desktop → Settings → General → Use the WSL 2 based engine**.

---

## 3) Create a local Kubernetes cluster with kind

```sh
# Create
kind create cluster --name precheck

# Verify node is Ready
kubectl get nodes

# Optional: cluster endpoints
kubectl cluster-info --context kind-precheck
```

**Expected:**  
- `kind create cluster` shows steps like *Starting control-plane*, *Installing CNI*, *Installing StorageClass*.  
- `kubectl get nodes` shows **1 node** with **STATUS = Ready** and a version like `v1.34.x`.  
- `kubectl cluster-info` prints URLs for Kubernetes control plane and CoreDNS.

---

## 4) Deploy a tiny test app and reach it

Create a simple NGINX Deployment and expose it, then access it locally.

```sh
# Create a Deployment
kubectl create deployment web --image=nginx:alpine

# Expose it inside the cluster
kubectl expose deployment web --port=80

# Port-forward to your laptop (keep this running)
kubectl port-forward deploy/web 8080:80
```

Open **http://localhost:8080** in your browser (or run `curl http://localhost:8080`).

**Expected:** A web page that includes “**Welcome to nginx!**”.  
(Press **Ctrl+C** in the terminal to stop the port-forward.)

---

## 5) (Optional) Helm smoke test

Verify Helm can reach public chart repos.

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Now search for nginx charts:  

- **Linux/macOS (bash/zsh):**
```sh
helm search repo nginx | head -n 5
```
- **Windows (PowerShell):**
```powershell
helm search repo nginx | Select-Object -First 5
```

**Expected:** A short table of nginx charts is listed (name/version).

---

## 6) Clean up

You can remove the test app and the precheck cluster any time:

```sh
# Delete test resources
kubectl delete deployment/web service/web || true

# Delete the cluster
kind delete cluster --name precheck
```

**Expected:** Resources/cluster are deleted with confirmation messages.

---

## Troubleshooting Quick Reference

- **Docker connect error (`dockerDesktopLinuxEngine` pipe not found):**  
  Start Docker Desktop and wait until it says “Running”. On Windows, install/enable **WSL 2**.

- **`kubectl get nodes` shows NotReady:**  
  Wait 1–2 minutes after cluster creation. If it persists, run `kubectl get pods -A` and look for CrashLoopBackOff or Pending pods.

- **Port-forward works but browser can’t connect:**  
  Ensure the port-forward command is still running. Some corporate VPNs/firewalls can interfere — try pausing VPN.

- **Windows without Chocolatey:**  
  Use **Scoop** or **manual downloads** for kubectl/kind/helm; ensure their folders are in your **PATH**.

---

## Success Criteria (All Green)
1. `docker run hello-world` prints *Hello from Docker!*  
2. `kind create cluster --name precheck` completes without errors  
3. `kubectl get nodes` shows one **Ready** node  
4. NGINX page loads at **http://localhost:8080** via port-forward  
5. (Optional) `helm search repo nginx` returns results

You’re ready for the workshop
