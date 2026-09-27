# DevOps Career Switch Prep — Day 1 Notes
**Background:** 6 years in automotive (software/embedded engineer), switching to DevOps Engineer, targeting 3-years-experience level roles. Interview timeline: 1-3 months. Focus: project-based/scenario interview questions, not textbook definitions.

---

## Overall Plan
Building one end-to-end project: **"Deploy a web app with a full CI/CD pipeline to Kubernetes on AWS"**

Architecture pieces (in build order):
1. App + Dockerfile ✅ DONE
2. Local Kubernetes (minikube) — Deployment/Service/Ingress ⏳ IN PROGRESS (cluster installed, not yet deployed)
3. CI pipeline (GitHub Actions) — auto build + push image
4. Terraform — provision real EKS cluster on AWS
5. CD pipeline — connect CI to deploying on EKS
6. Monitoring (Prometheus + Grafana)
7. Deliberately break something, document the incident (interview story material)

---

## Project Folder Structure (on Windows, in `C:\devops practice\day1\projects\devops-demo-app`)
```
devops-demo-app/
├── venv/              (local Python env — NOT committed to Git)
├── .gitignore
├── app.py
├── requirements.txt
└── Dockerfile
```

---

## Files Created

### app.py
```python
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route("/")
def home():
    return jsonify({"message": "Hello from DevOps pipeline project"})

@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port)
```

### requirements.txt
```
flask==3.0.0
```
Generated via: `python -m venv venv` → activate → `pip install flask` → `pip freeze > requirements.txt`

### .gitignore
```
venv/
__pycache__/
*.pyc
```

### Dockerfile (FINAL, WORKING VERSION — after bug fix)
```dockerfile
# ---- Stage 1: Build ----
FROM python:3.11-slim AS builder

WORKDIR /app

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ---- Stage 2: Runtime ----
FROM python:3.11-slim

WORKDIR /app

COPY --from=builder /opt/venv /opt/venv
COPY app.py .

ENV PATH="/opt/venv/bin:$PATH"
ENV PORT=5000

EXPOSE 5000

RUN useradd -m appuser
USER appuser

CMD ["python", "app.py"]
```

---

## THE BUG (real debugging story #1 — use in interviews)

**Original (broken) Dockerfile used:**
```dockerfile
RUN pip install --no-cache-dir --user -r requirements.txt
...
COPY --from=builder /root/.local /root/.local
...
ENV PATH=/root/.local/bin:$PATH
```

**Error:** `ModuleNotFoundError: No module named 'flask'`

**Root cause:** `pip install --user` installs packages tied to the *current user's home directory*. Build stage ran as `root` → installed to `/root/.local`. But the runtime stage switches to `USER appuser` (non-root, for security) → `appuser`'s home is `/home/appuser`, not `/root` → Python couldn't find Flask even though the files existed in the image, just under the wrong path.

**Fix:** Use a Python virtual environment at a fixed path (`/opt/venv`) instead of `--user` installs. A venv is path-based, not user-based, so it works regardless of which user runs the container.

**Interview-ready answer:** (full version saved — ask me to regenerate if needed) Structure: Context → what multi-stage build does → the bug → root cause (user context changes between build/runtime stages) → fix (venv over `--user`) → takeaway (container debugging is about understanding layers and user context, not just app code).

**Likely follow-ups:**
- Why run as non-root? → limits blast radius if container compromised
- Why not just switch back to root? → reintroduces security risk, not a real fix
- `--user` vs venv? → `--user` ties packages to a user's home dir; venv is a portable, user-independent folder

---

## Key Concepts Learned (with the "why")

**Image vs Container**
- Image = static built blueprint (on disk)
- Container = running instance of an image
- `docker images` = list blueprints; `docker ps` = running containers only; `docker ps -a` = all containers incl. stopped
- You push/pull **images**, never containers

**Multi-stage Docker builds**
- Stage 1 (builder) installs dependencies; Stage 2 (runtime) copies only what's needed
- Reduces final image size (no build tools/caches shipped)
- Layer order matters for caching: copy `requirements.txt` and install deps BEFORE copying app code, so code changes don't invalidate the dependency-install cache layer

**Docker networking (real confusion point — now resolved)**
- `host="0.0.0.0"` in Flask = listen on all interfaces (vs `127.0.0.1` = only localhost, unreachable from outside)
- Flask prints `127.0.0.1` and `172.17.0.2` (container's internal Docker bridge network IP) — **neither is reachable from your laptop**
- What actually works: `http://localhost:5000` — this works because of `-p 5000:5000` in `docker run`, a Docker-level port mapping that Flask has no awareness of
- `172.17.0.2` ≠ your system's real IP (real IP is usually `192.168.x.x`, check via `ipconfig`)

**Routes/endpoints**
- `/` and `/health` are separate URL paths defined in your own Flask code
- `/health` matters specifically because Kubernetes will later use it for liveness/readiness probes (auto health checks)

**Git/GitHub workflow used**
```bash
git init
git add .
git commit -m "initial commit - flask app with dockerfile"
git remote add origin https://github.com/yourusername/devops-demo-app.git
git branch -M main
git push -u origin main
```
Auth: GitHub no longer accepts plain passwords — used Personal Access Token or VS Code's built-in GitHub sign-in (Accounts icon, bottom-left).

**Docker Hub push workflow**
```bash
docker login
docker tag myapp:v1 hkdevops108/devops-demo-app:v1
docker push hkdevops108/devops-demo-app:v1
```
Naming convention: `username/repo:tag` — required by registries so they know which account/repo/version an image belongs to. Tag by version (e.g., `v1`), not `latest`, for traceability.

**Real debugging story #2 — Docker Hub push timeout**
Large layer upload hit `net/http: timeout awaiting response headers` mid-push (unstable connection). Fix: just retried — Docker skips layers already uploaded ("Already exists"), so retries are incremental, not from scratch. Eventually succeeded after 2-3 retries. Also discussed: Docker Desktop → Settings → Docker Engine → add `"max-concurrent-uploads": 1` to reduce parallel upload strain on flaky connections.

**Detached mode (for easier container management)**
```bash
docker run -d -p 5000:5000 --name myapp-container myapp:v1
docker ps                      # see it running
docker stop myapp-container    # stop it later
```

---

## Kubernetes Setup — COMPLETED TODAY

Installed on Windows via PowerShell (also usable in VS Code's integrated terminal — same thing):

**kubectl install:**
```powershell
curl.exe -LO "https://dl.k8s.io/release/v1.31.0/bin/windows/amd64/kubectl.exe"
mkdir C:\kubectl
move kubectl.exe C:\kubectl\
# then added C:\kubectl to system PATH via Environment Variables UI
kubectl version --client
```

**minikube install:**
```powershell
winget install minikube
minikube version
```

**Started cluster (using Docker as driver — no extra VM software needed):**
```powershell
minikube start --driver=docker
```

**Verified — cluster is healthy:**
```
kubectl get nodes
→ NAME       STATUS   ROLES           AGE   VERSION
  minikube   Ready    control-plane   85s   v1.37.0

minikube status
→ host: Running, kubelet: Running, apiserver: Running, kubeconfig: Configured
```

All free/local — no cloud cost. Cloud costs only start later with AWS/EKS (Terraform stage) — will be flagged clearly when we reach that point, with reminders to `terraform destroy` after practice.

---

## NEXT STEPS (pick up here in new chat)

1. **Deploy the pushed image (`hkdevops108/devops-demo-app:v1`) into minikube:**
   - Write first Pod YAML, understand what a Pod is
   - Deploy it, verify running (`kubectl get pods`, `kubectl describe pod`)
2. **Move from Pod → Deployment** (replicas, self-healing)
   - Write Deployment YAML
   - Test self-healing by killing a pod and watching it get recreated
3. **Service** — expose the deployment (ClusterIP → NodePort), understand pod networking (this is expected to be another "real confusion" moment like Docker networking was — budget time for it)
4. **ConfigMaps/Secrets**
5. **Readiness/liveness probes** using the existing `/health` endpoint — ties directly back into Day 1 work
6. **Debugging practice** — break something on purpose, use `kubectl logs` / `kubectl describe` to fix it (interview story #3)
7. **GitHub Actions CI pipeline** — automate the build+push you did manually today
8. **Terraform + AWS EKS** (cloud costs begin here — be careful, tear down after practice)
9. **Monitoring** — Prometheus + Grafana
10. Full mock interview practice using all 3+ real debugging stories

---

## Docker Hub / GitHub account reference
- Docker Hub image: `hkdevops108/devops-demo-app:v1`
- GitHub repo: `devops-demo-app` (under your GitHub username)

---

*Paste this file into a new chat to continue exactly where you left off.*
