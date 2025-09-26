# One-Box Kubernetes DevOps Demo (Linux & macOS)

**Stack:** React (frontend) · Node.js/Express (API) · PostgreSQL · Docker · kind · HPA (CPU) · metrics-server · k6 · kube-prometheus-stack (Prometheus + Grafana)

**Default ports:** Frontend `30080`, API `30081`, Grafana `30090`  
**Cluster name:** `demo`  
**Shell:** bash (Linux & macOS Terminal)

> ⚠️ **NodePort reachability:**  
> - **Linux (Docker Engine):** NodePorts usually work directly on `localhost`.  
> - **macOS (Docker Desktop):** NodePorts are **not** host-reachable by default; use **port-forward** or create the cluster with **extraPortMappings** (shown below).

---

## 0) Pre-flight

Check tools are installed and on PATH:
```bash
docker version
kind version
kubectl version --client
helm version
node -v
npm -v
k6 version
git --version
```

<details>
<summary>Optional installs (quick hints)</summary>

**Ubuntu/Debian (apt)**: see official docs for current repos; rough outline:
```bash
# Docker
sudo apt-get update
# install docker engine via official docs (get.docker.com script or repo)
# kind (amd64/arm64)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.30.0/kind-linux-amd64 && chmod +x kind && sudo mv kind /usr/local/bin/
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
# helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
# k6
sudo apt-get install -y gnupg ca-certificates
curl -s https://repos.k6.io/key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/k6-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://repos.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install -y k6
```

**macOS (Homebrew):**
```bash
brew install --cask docker
brew install kind kubectl helm k6 node git
# start Docker Desktop from /Applications and let it finish initializing
```
</details>

---

## 1) Project layout

> Set a share-safe project root (change if you like):
```bash
export PROJECT_ROOT="$HOME/projects/k8s-devops-demo"

mkdir -p "$PROJECT_ROOT"/{api,frontend,k8s,k6}
```

---

## 2) API (Node.js + Express + CORS + PG)

**`$PROJECT_ROOT/api/package.json`**
```json
{
  "name": "demo-api",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "express": "^4.19.2",
    "pg": "^8.12.0"
  }
}
```

**`$PROJECT_ROOT/api/server.js`**
```javascript
const express = require("express");
const { Pool } = require("pg");

const app = express();
const port = process.env.PORT || 8080;

// CORS for demo
app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Methods", "GET, OPTIONS");
  res.header("Access-Control-Allow-Headers", "Content-Type");
  if (req.method === "OPTIONS") return res.sendStatus(200);
  next();
});

const pool = new Pool({
  host: process.env.PGHOST || "postgres",
  user: process.env.PGUSER || "appuser",
  password: process.env.PGPASSWORD || "apppass",
  database: process.env.PGDATABASE || "appdb",
  port: 5432,
});

app.get("/api/health", (_, res) => res.json({ status: "ok" }));

app.get("/api/time", async (_, res) => {
  try {
    const r = await pool.query("SELECT NOW()");
    res.json({ time: r.rows[0].now });
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: "db connection failed" });
  }
});

app.get("/api/burn", (req, res) => {
  const ms = Math.min(Number(req.query.ms) || 200, 5000);
  const end = Date.now() + ms;
  while (Date.now() < end) {}
  res.json({ ok: true, burned_ms: ms });
});

app.listen(port, () => console.log(`API running on ${port}`));
```

**`$PROJECT_ROOT/api/Dockerfile`**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY . .
ENV PORT=8080
EXPOSE 8080
CMD ["npm", "start"]
```

---

## 3) Frontend (React via CRA) + Dockerfile

```bash
cd "$PROJECT_ROOT"
npx create-react-app frontend --use-npm
```

**`$PROJECT_ROOT/frontend/src/App.js`**
```javascript
import React, { useEffect, useState } from "react";

function App() {
  const [time, setTime] = useState("loading...");

  useEffect(() => {
    fetch("http://localhost:30081/api/time")
      .then(r => r.json())
      .then(d => setTime(String(d.time || "n/a")))
      .catch(() => setTime("API unavailable"));
  }, []);

  return (
    <div style={{ fontFamily: "Arial", padding: 24 }}>
      <h1>🚀 DevOps + Kubernetes Demo</h1>
      <p>Backend time: {time}</p>
    </div>
  );
}

export default App;
```

**`$PROJECT_ROOT/frontend/Dockerfile`**
```dockerfile
# build
FROM node:20-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# serve
FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx","-g","daemon off;"]
```

---

## 4) Build images

```bash
cd "$PROJECT_ROOT"
docker build -t demo-api:local ./api
docker build -t demo-frontend:local ./frontend
```

---

## 5) Create kind cluster `demo` (or detect)

```bash
if ! kind get clusters | grep -qx "demo"; then
  kind create cluster --name demo
else
  echo "Cluster 'demo' already exists. Skipping creation."
fi

kubectl cluster-info
kubectl get nodes
```

---

## 6) Load images into kind

```bash
kind load docker-image demo-api:local --name demo
kind load docker-image demo-frontend:local --name demo
```

---

## 7) Kubernetes manifests

**`$PROJECT_ROOT/k8s/postgres-deployment.yaml`**
```yaml
apiVersion: v1
kind: Secret
metadata: { name: pg-secret }
type: Opaque
stringData:
  PGUSER: appuser
  PGPASSWORD: apppass
  PGDATABASE: appdb
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: postgres }
spec:
  replicas: 1
  selector: { matchLabels: { app: postgres } }
  template:
    metadata: { labels: { app: postgres } }
    spec:
      containers:
        - name: postgres
          image: postgres:15
          env:
            - name: POSTGRES_USER
              valueFrom: { secretKeyRef: { name: pg-secret, key: PGUSER } }
            - name: POSTGRES_PASSWORD
              valueFrom: { secretKeyRef: { name: pg-secret, key: PGPASSWORD } }
            - name: POSTGRES_DB
              valueFrom: { secretKeyRef: { name: pg-secret, key: PGDATABASE } }
          ports: [{ containerPort: 5432 }]
---
apiVersion: v1
kind: Service
metadata: { name: postgres }
spec:
  selector: { app: postgres }
  ports: [{ port: 5432 }]
```

**`$PROJECT_ROOT/k8s/api-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api }
spec:
  replicas: 1
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
        - name: api
          image: demo-api:local
          env:
            - { name: PGHOST, value: postgres }
            - name: PGUSER
              valueFrom: { secretKeyRef: { name: pg-secret, key: PGUSER } }
            - name: PGPASSWORD
              valueFrom: { secretKeyRef: { name: pg-secret, key: PGPASSWORD } }
            - name: PGDATABASE
              valueFrom: { secretKeyRef: { name: pg-secret, key: PGDATABASE } }
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "500m", memory: "256Mi" }
---
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  type: NodePort
  selector: { app: api }
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30081
```

**`$PROJECT_ROOT/k8s/frontend-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: frontend }
spec:
  replicas: 1
  selector: { matchLabels: { app: frontend } }
  template:
    metadata: { labels: { app: frontend } }
    spec:
      containers:
        - name: frontend
          image: demo-frontend:local
          ports: [{ containerPort: 80 }]
---
apiVersion: v1
kind: Service
metadata: { name: frontend }
spec:
  type: NodePort
  selector: { app: frontend }
  ports:
    - port: 80
      nodePort: 30080
```

**Apply & rollout:**
```bash
cd "$PROJECT_ROOT"

kubectl apply -f ./k8s/postgres-deployment.yaml
kubectl rollout status deploy/postgres --timeout=180s

kubectl apply -f ./k8s/api-deployment.yaml
kubectl rollout status deploy/api --timeout=180s

kubectl apply -f ./k8s/frontend-deployment.yaml
kubectl rollout status deploy/frontend --timeout=180s

kubectl get pods -o wide
kubectl get svc
```

---

## 8) Access services

- **Linux:** open directly (NodePort should work):
  ```bash
  curl -s http://localhost:30081/api/health
  curl -s http://localhost:30081/api/time
  xdg-open http://localhost:30080  >/dev/null 2>&1 || echo "Open http://localhost:30080"
  ```

- **macOS:** either **port-forward** (quick) or create a **cluster config** (permanent).

  **Port-forward (keep terminals running):**
  ```bash
  # Terminal A (API)
  kubectl port-forward svc/api 30081:80

  # Terminal B (Frontend)
  kubectl port-forward svc/frontend 30080:80
  ```

  Then:
  ```bash
  curl -s http://localhost:30081/api/health
  curl -s http://localhost:30081/api/time
  open http://localhost:30080
  ```

---

## 9) metrics-server (Helm)

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update

helm upgrade --install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set args="{--kubelet-insecure-tls,--kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP}"

kubectl rollout status deploy/metrics-server -n kube-system --timeout=180s
kubectl wait --for=condition=Available apiservice/v1beta1.metrics.k8s.io --timeout=180s

kubectl top nodes
kubectl top pods -A
```

---

## 10) HPA (CPU) for API

**`$PROJECT_ROOT/k8s/hpa-api.yaml`**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api-hpa }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 1
  maxReplicas: 5
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

**Apply & check:**
```bash
kubectl apply -f ./k8s/hpa-api.yaml
kubectl get hpa
```

---

## 11) k6 load test (trigger scaling)

**`$PROJECT_ROOT/k6/k6-burn.js`**
```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = { vus: 30, duration: '3m' };

export default function () {
  http.get('http://localhost:30081/api/burn?ms=400', { timeout: '10s' });
  sleep(0.05);
}
```

**Watchers + run:**
```bash
# watchers (each in its own terminal)
kubectl get hpa -w
kubectl get deploy api -w
# CPU polling loop (top has no -w)
while true; do kubectl top pods -l app=api; sleep 2; done

# run load (ensure API is reachable per OS section above)
cd "$PROJECT_ROOT"
k6 run ./k6/k6-burn.js
```

Expected: `api` scales up (≤5), then down within ~1–2 minutes after load ends (thanks to the `behavior.scaleDown`).

---

## 12) kube-prometheus-stack (Prometheus + Grafana) + NodePort 30090

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

# This can take a few minutes
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/instance=kube-prometheus -n monitoring --timeout=600s

# Patch Grafana to NodePort 30090
kubectl patch svc kube-prometheus-grafana -n monitoring \
  -p '{"spec":{"type":"NodePort","ports":[{"port":80,"targetPort":3000,"nodePort":30090}]}}'
```

- **Linux:** open directly  
  ```bash
  xdg-open http://localhost:30090 >/dev/null 2>&1 || echo "Open http://localhost:30090"
  ```

- **macOS (no extraPortMappings):** port-forward  
  ```bash
  kubectl port-forward -n monitoring svc/kube-prometheus-grafana 30090:80
  open http://localhost:30090
  ```

**Grafana credentials:**
```bash
kubectl get secret kube-prometheus-grafana -n monitoring -o jsonpath="{.data.admin-user}" | base64 -d; echo
kubectl get secret kube-prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

**In Grafana:**
- **Dashboards → Import**: IDs **315**, **1860**, **6417** (choose Prometheus).
- **Explore → Prometheus**:
  - Sanity: `up`
  - API CPU per pod (2-min rate):
    ```promql
    sum by (pod) (
      rate(container_cpu_usage_seconds_total{
        namespace="default",
        pod=~"api-.*",
        container!="POD"
      }[2m])
    )
    ```
- Re-run `k6` to see CPU & replicas change.

---

## 13) Cleanup

**Delete resources (keep cluster):**
```bash
kubectl delete -f ./k8s/hpa-api.yaml
kubectl delete -f ./k8s/frontend-deployment.yaml
kubectl delete -f ./k8s/api-deployment.yaml
kubectl delete -f ./k8s/postgres-deployment.yaml
helm uninstall kube-prometheus -n monitoring
helm uninstall metrics-server -n kube-system
```

**Delete cluster:**
```bash
kind delete cluster --name demo
```

---

## Optional (macOS & Linux): Make NodePorts reachable without port-forward

**`$PROJECT_ROOT/kind-config.yaml`**
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 30080
    protocol: TCP
  - containerPort: 30081
    hostPort: 30081
    protocol: TCP
  - containerPort: 30090
    hostPort: 30090
    protocol: TCP
```

**Recreate & reload:**
```bash
kind delete cluster --name demo
kind create cluster --name demo --config "$PROJECT_ROOT/kind-config.yaml"

kind load docker-image demo-api:local --name demo
kind load docker-image demo-frontend:local --name demo

kubectl apply -f ./k8s/postgres-deployment.yaml
kubectl apply -f ./k8s/api-deployment.yaml
kubectl apply -f ./k8s/frontend-deployment.yaml
kubectl apply -f ./k8s/hpa-api.yaml
# Grafana install/patch as above; now http://localhost:30080/30081/30090 work directly.
```

---

### Quick Troubleshooting

- **Frontend: “API unavailable”** → Ensure API reachable (port-forward on macOS); CORS headers present in API.  
- **API → DB fails** → Postgres env must be `POSTGRES_*` (mapped from Secret); check `kubectl get pods,svc`.  
- **HPA targets `<unknown>`** → Wait 30–60s; verify metrics-server is Ready and `kubectl top pods` returns data.  
- **k6 timeouts** → Increase `timeout` in k6 script; ensure API access method (NodePort vs port-forward) is correct.  
- **Downscale “stuck” high** → Provided HPA `behavior.scaleDown`; allow ~1–2 minutes post-load.
