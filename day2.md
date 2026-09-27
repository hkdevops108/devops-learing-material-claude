# DevOps Career Switch Prep — Day 2 Notes
**Continues from Day 1.** Background: 6 years automotive (software/embedded engineer) → DevOps Engineer, targeting 3-years-experience level roles. Focus: project-based/scenario interview questions.

---

## Progress vs roadmap

1. ✅ App + Dockerfile
2. ✅ Local Kubernetes (minikube) — Pod → Deployment → Service, all working
3. ✅ ConfigMaps/Secrets — wired into Deployment via `envFrom`
4. ⏳ **NEXT: Readiness/liveness probes** (using existing `/health` endpoint)
5. Debugging stories collected so far: 4 (see below) — ahead of schedule
6. GitHub Actions CI pipeline — not started
7. Terraform + AWS EKS — not started (cloud costs begin here)
8. Monitoring (Prometheus + Grafana) — not started
9. Mock interview practice — not started

---

## Kubernetes objects built today

### `k8s/pod.yaml` (used briefly, then deleted once Deployment took over)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: devops-demo-app-pod
  labels:
    app: devops-demo-app
spec:
  containers:
    - name: devops-demo-app
      image: hkdevops108/devops-demo-app:v1
      ports:
        - containerPort: 5000
```

### `k8s/deployment.yaml` (current, using v2 image + Config/Secret)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-demo-app-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devops-demo-app
  template:
    metadata:
      labels:
        app: devops-demo-app
    spec:
      containers:
        - name: devops-demo-app
          image: hkdevops108/devops-demo-app:v2
          ports:
            - containerPort: 5000
          envFrom:
            - configMapRef:
                name: devops-demo-app-config
            - secretRef:
                name: devops-demo-app-secret
```

### `k8s/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: devops-demo-app-service
spec:
  type: NodePort
  selector:
    app: devops-demo-app
  ports:
    - port: 80
      targetPort: 5000
```

### `k8s/configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: devops-demo-app-config
data:
  APP_ENV: "minikube-local"
```

### `k8s/secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: devops-demo-app-secret
type: Opaque
stringData:
  API_KEY: "demo-placeholder-key-123"
```

### `app.py` (updated to v2 — reads APP_ENV)
```python
from flask import Flask, jsonify
import os

app = Flask(__name__)

APP_ENV = os.environ.get("APP_ENV", "not-set")

@app.route("/")
def home():
    return jsonify({"message": "Hello from DevOps pipeline project", "environment": APP_ENV})

@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port)
```

---

## Key Concepts Learned Today

### Pod
- Smallest deployable unit in Kubernetes — a wrapper around one or more containers sharing network/storage
- Almost always one container per Pod in simple setups
- `apiVersion: v1` (core API group)
- Not self-healing on its own — if it dies, it's gone

### Deployment
- `apiVersion: apps/v1` (apps API group, not core)
- Manages desired replica count; creates a **ReplicaSet**, which creates the Pods (Deployment doesn't create Pods directly)
- `selector.matchLabels` must exactly match `template.metadata.labels`, or validation fails
- Self-healing: continuously reconciles actual vs desired replica count
- **Live demo done today:** deleted a running Pod → Deployment detected 1 < 2 replicas → ReplicaSet spun up a replacement automatically within seconds

### Service
- Solves the problem that Pod IPs are ephemeral (change every time a Pod is recreated)
- Uses a `selector` (label match) to route traffic to whichever Pods currently match, regardless of Pod churn
- `type: NodePort` exposes it outside the cluster (vs default `ClusterIP`, which is cluster-internal only)
- On minikube + Docker driver on Windows, use `minikube service <name> --url` to get a working local URL — must keep that terminal window open, as it runs a live tunnel process (closing/reusing the window kills the tunnel and changes the port on next run)

### ConfigMaps & Secrets — the "why"
- **Core problem solved:** decouple configuration from the Docker image, so the same built image can run unchanged across dev/staging/prod — only the config paired with it changes
- Without this, you'd have to rebuild the image every time a config value changes, defeating "build once, deploy anywhere"
- **ConfigMap** = non-sensitive key-value config, lives as its own cluster object, does nothing until a Pod references it
- **Secret** = structurally identical to ConfigMap, same consumption mechanisms, but intended for sensitive values
- **Important gotcha (interview question):** Secrets are base64-*encoded*, not encrypted. Anyone with `kubectl get secret -o yaml` or etcd read access can decode them trivially. Real protection needs encryption-at-rest on etcd or an external secrets manager (Vault, AWS Secrets Manager, etc.)
- Three ways to consume either one in a Pod:
  1. `envFrom` — inject every key as an env var (used today — simplest)
  2. `env` + `valueFrom.configMapKeyRef` / `secretKeyRef` — pull in one specific key, can rename it
  3. Mount as a volume — each key becomes a file inside the container (used for full config files, not single values)
- `stringData` (plain text, auto-encoded by Kubernetes) vs `data` (must pre-encode yourself) — `stringData` is easier to write by hand

---

## Debugging Stories Log (interview material)

**#1 — Docker multi-stage build, `ModuleNotFoundError: No module named 'flask'`** *(Day 1)*
`pip install --user` ties packages to the current user's home dir. Build stage ran as root → installed to `/root/.local`. Runtime stage switched to non-root `appuser` → wrong home dir → module not found despite files existing in the image. Fix: use a venv at a fixed path (`/opt/venv`) instead — path-based, not user-based.

**#2 — Docker Hub push timeout** *(Day 1)*
Large layer upload hit `net/http: timeout awaiting response headers` on a flaky connection. Fix: simply retried — Docker skips already-uploaded layers ("Already exists"), so retries are incremental. Also discussed `"max-concurrent-uploads": 1` in Docker Desktop settings to reduce strain on unstable connections.

**#3 — Pod stuck in `Pending`** *(Day 2)*
New Pod wouldn't schedule. Root cause: minikube's node wasn't in a healthy `Ready` state (likely after a sleep/resume cycle). Diagnosed via `kubectl describe pod` (Events section) → `kubectl get nodes` → `minikube status`. Fix: `minikube stop` + `minikube start --driver=docker` restored a healthy node, and the Pod scheduled immediately after.

**#4 — Self-healing demo** *(Day 2)*
Deleted a running Pod managed by a Deployment (`replicas: 2`). Watched it go `Terminating` → disappear → new Pod appear with a fresh name hash → back to 2/2 `Running` within seconds, with zero manual intervention. Demonstrates the ReplicaSet continuously reconciling desired vs actual state — the mechanism behind zero-downtime deploys and node-failure recovery in production.

**#5 — Stale deployment spec after image rebuild** *(Day 2)*
Rebuilt and pushed a new image tag (`:v2`) with updated app code. `docker images` confirmed the new tag existed locally and on Docker Hub. App still returned old (v1) behavior after `kubectl apply`. Root cause: `kubectl describe deployment` showed the Deployment was still running `:v1` — the `deployment.yaml` file's image line had never actually been edited/saved before applying, so Kubernetes correctly deployed exactly what the manifest said, not what was assumed. Fix: verify actual file content (`Get-Content` / `cat`) immediately before applying, rather than trusting an edit happened. **Takeaway:** `kubectl apply` does exactly what the YAML says — when output doesn't match expectations, check the YAML before suspecting the cluster.

---

## Useful commands learned today

```powershell
# Pods
kubectl get pods
kubectl describe pod <name>
kubectl delete pod <name>
kubectl port-forward pod/<name> 5000:5000

# Deployments
kubectl apply -f k8s/deployment.yaml
kubectl rollout status deployment/<name>
kubectl describe deployment <name> | Select-String "Image"

# Services
kubectl get services
minikube service <service-name> --url

# Nodes / cluster health
kubectl get nodes
minikube status
minikube stop
minikube start --driver=docker

# Secrets/env verification
kubectl exec -it deploy/<deployment-name> -- env | Select-String "API_KEY|APP_ENV"

# Local file sanity check before applying (habit to build)
Get-Content k8s\deployment.yaml | Select-String image
```

---

## NEXT STEPS (pick up here in new chat)

1. **Readiness/liveness probes** — wire the existing `/health` endpoint into `deployment.yaml`
   - Liveness probe: restarts the container if it fails (app is stuck/deadlocked)
   - Readiness probe: removes the Pod from Service routing if it fails (app is up but not ready for traffic yet)
2. **Debugging practice** — break something on purpose, use `kubectl logs` / `kubectl describe` to fix it
3. **GitHub Actions CI pipeline** — automate the build+push done manually so far
4. **Terraform + AWS EKS** (cloud costs begin here — be careful, tear down with `terraform destroy` after practice)
5. **Monitoring** — Prometheus + Grafana
6. Full mock interview practice using all 5 real debugging/demo stories

---

## Docker Hub / GitHub account reference
- Docker Hub image: `hkdevops108/devops-demo-app` (tags: `v1`, `v2`)
- GitHub repo: `devops-demo-app` (under your GitHub username)

---

*Paste this file into a new chat to continue exactly where you left off.*
