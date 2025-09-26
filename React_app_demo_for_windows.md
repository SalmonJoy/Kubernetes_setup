# One-Box Kubernetes DevOps Demo (Windows 11 + PowerShell)

**Stack:** React (frontend) · Node.js/Express (API) · PostgreSQL · Docker · kind · HPA (CPU) · metrics-server · k6 · kube-prometheus-stack (Prometheus + Grafana)

**Default ports:** Frontend `30080`, API `30081`, Grafana `30090`  
**Cluster name:** `demo`  
**Shell:** PowerShell

---

## 0) Pre-flight

```powershell
docker version
kind version
kubectl version --client
helm version
node -v
npm -v
k6 version
git --version
```

---

## 1) Project layout

> Set your project root once and reuse it everywhere.

```powershell
# Set a sample, share-safe project path (change if you like)
$PROJECT_ROOT = "C:\Projects\k8s-devops-demo"

# Create folders
New-Item -ItemType Directory -Path $PROJECT_ROOT -Force | Out-Null
New-Item -ItemType Directory -Path "$PROJECT_ROOT\api" -Force | Out-Null
New-Item -ItemType Directory -Path "$PROJECT_ROOT\frontend" -Force | Out-Null
New-Item -ItemType Directory -Path "$PROJECT_ROOT\k8s" -Force | Out-Null
New-Item -ItemType Directory -Path "$PROJECT_ROOT\k6" -Force | Out-Null
```

---

## 2) API (Node.js + Express + CORS + PG)

**`$PROJECT_ROOT\api\package.json`**
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

**`$PROJECT_ROOT\api\server.js`**
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

**`$PROJECT_ROOT\api\Dockerfile`**
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

```powershell
Set-Location $PROJECT_ROOT
npx create-react-app frontend --use-npm
```

**`$PROJECT_ROOT\frontend\src\App.js`**
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

**`$PROJECT_ROOT\frontend\Dockerfile`**
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

```powershell
Set-Location $PROJECT_ROOT
docker build -t demo-api:local .\api
docker build -t demo-frontend:local .\frontend
```

---

## 5) Create kind cluster `demo` (or detect)

```powershell
$clusters = kind get clusters
if ($clusters -notcontains "demo") {
  kind create cluster --name demo
} else {
  Write-Host "Cluster 'demo' already exists. Skipping creation."
}

kubectl cluster-info
kubectl get nodes
```

---

## 6) Load images into kind

```powershell
kind load docker-image demo-api:local --name demo
kind load docker-image demo-frontend:local --name demo
```

---

## 7) Kubernetes manifests

**`$PROJECT_ROOT\k8s\postgres-deployment.yaml`**
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

**`$PROJECT_ROOT\k8s\api-deployment.yaml`**
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

**`$PROJECT_ROOT\k8s\frontend-deployment.yaml`**
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
```powershell
Set-Location $PROJECT_ROOT

kubectl apply -f .\k8s\postgres-deployment.yaml
kubectl rollout status deploy/postgres --timeout=180s

kubectl apply -f .\k8s\api-deployment.yaml
kubectl rollout status deploy/api --timeout=180s

kubectl apply -f .\k8s\frontend-deployment.yaml
kubectl rollout status deploy/frontend --timeout=180s

kubectl get pods -o wide
kubectl get svc
```

---

## 8) Access services on Windows: port-forward

> NodePorts aren’t host-accessible by default on kind (Windows/macOS). Use port-forward (keep these windows open):

```powershell
# API → http://localhost:30081
Start-Process powershell -ArgumentList '-NoExit','-Command','kubectl port-forward svc/api 30081:80'

# Frontend → http://localhost:30080
Start-Process powershell -ArgumentList '-NoExit','-Command','kubectl port-forward svc/frontend 30080:80'
```

**Verify:**
```powershell
Invoke-RestMethod http://localhost:30081/api/health
Invoke-RestMethod http://localhost:30081/api/time
Start-Process http://localhost:30080
```

---

## 9) metrics-server (Helm)

```powershell
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update

helm upgrade --install metrics-server metrics-server/metrics-server `
  --namespace kube-system `
  --set args="{--kubelet-insecure-tls,--kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP}"

kubectl rollout status deploy/metrics-server -n kube-system --timeout=180s
kubectl wait --for=condition=Available apiservice/v1beta1.metrics.k8s.io --timeout=180s

kubectl top nodes
kubectl top pods -A
```

---

## 10) HPA (CPU) for API

**`$PROJECT_ROOT\k8s\hpa-api.yaml`**
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
```powershell
kubectl apply -f .\k8s\hpa-api.yaml
kubectl get hpa
```

---

## 11) k6 load test (to trigger scaling)

**`$PROJECT_ROOT\k6\k6-burn.js`**
```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = { vus: 30, duration: '3m' };

export default function () {
  http.get('http://localhost:30081/api/burn?ms=400', { timeout: '10s' });
  sleep(0.05);
}
```

**Watch scaling + run:**
```powershell
# watchers (each opens a window)
Start-Process powershell -ArgumentList '-NoExit','-Command','kubectl get hpa -w'
Start-Process powershell -ArgumentList '-NoExit','-Command','kubectl get deploy api -w'
Start-Process powershell -ArgumentList '-NoExit','-Command','while ($true) { kubectl top pods -l app=api; Start-Sleep -Seconds 2 }'

# run load (keep API port-forward open)
Set-Location $PROJECT_ROOT
k6 run .\k6\k6-burn.js
```

Expected: API replicas scale up (≤5), then down within ~1–2 minutes after load (HPA `behavior` speeds downscale).

---

## 12) kube-prometheus-stack (Prometheus + Grafana) + NodePort 30090

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install kube-prometheus prometheus-community/kube-prometheus-stack `
  --namespace monitoring --create-namespace

# wait (can take a few minutes)
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/instance=kube-prometheus -n monitoring --timeout=600s

# patch Grafana to NodePort 30090
kubectl patch svc kube-prometheus-grafana -n monitoring `
  -p '{"spec":{"type":"NodePort","ports":[{"port":80,"targetPort":3000,"nodePort":30090}]}}'

# port-forward Grafana (keep open)
Start-Process powershell -ArgumentList '-NoExit','-Command','kubectl port-forward svc/kube-prometheus-grafana -n monitoring 30090:80'

# creds
$u = kubectl get secret kube-prometheus-grafana -n monitoring -o jsonpath="{.data.admin-user}"
$p = kubectl get secret kube-prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}"
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($u))
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p))

Start-Process http://localhost:30090
```

**In Grafana:**
- **Dashboards → Import**: IDs **315**, **1860**, **6417** (select Prometheus).
- **Explore → Prometheus**:
  - Sanity: `up`
  - API CPU per pod:
    ```promql
    sum by (pod) (
      rate(container_cpu_usage_seconds_total{
        namespace="default",
        pod=~"api-.*",
        container!="POD"
      }[2m])
    )
    ```
- Re-run `k6` to see CPU & replicas change in Grafana and `kubectl` watchers.

---

## 13) Cleanup

**Delete resources (keep cluster):**
```powershell
kubectl delete -f .\k8s\hpa-api.yaml
kubectl delete -f .\k8s\frontend-deployment.yaml
kubectl delete -f .\k8s\api-deployment.yaml
kubectl delete -f .\k8s\postgres-deployment.yaml
helm uninstall kube-prometheus -n monitoring
helm uninstall metrics-server -n kube-system
```

**Delete cluster:**
```powershell
kind delete cluster --name demo
```

---

### Optional: Make NodePorts reachable without port-forward

**`$PROJECT_ROOT\kind-config.yaml`**
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

```powershell
kind delete cluster --name demo
kind create cluster --name demo --config "$PROJECT_ROOT\kind-config.yaml"
kind load docker-image demo-api:local --name demo
kind load docker-image demo-frontend:local --name demo
kubectl apply -f .\k8s\postgres-deployment.yaml
kubectl apply -f .\k8s\api-deployment.yaml
kubectl apply -f .\k8s\frontend-deployment.yaml
kubectl apply -f .\k8s\hpa-api.yaml
# Grafana install/patch as above; now http://localhost:30080/30081/30090 work directly.
```

---

### Quick Troubleshooting

- **Frontend says “API unavailable”** → API port-forward running? CORS headers present in API?  
- **API → DB fails** → Postgres env must be `POSTGRES_*` (mapped from Secret).  
- **HPA shows `<unknown>`** → wait 30–60s; ensure `metrics-server` is Ready and `kubectl top pods` works.  
- **k6 timeouts** → increase `timeout` in k6; keep port-forward windows open.  
- **Downscale slow** → HPA `behavior.scaleDown` provided; allow ~1–2 minutes after load ends.
