# DevOps Career Switch Prep — Day 3 Notes
**Continues from Day 1 & Day 2.** Background: 6 years automotive (software/embedded engineer) → DevOps Engineer, targeting 3-years-experience level roles.

---

## Progress vs roadmap

1. ✅ App + Dockerfile
2. ✅ Local Kubernetes (minikube) — Pod → Deployment → Service
3. ✅ ConfigMaps/Secrets — wired in, plus dev-vs-devops ownership worked through
4. ✅ Deployment ↔ ReplicaSet relationship + rollback mechanics — deep dive done
5. ✅ Service types — all four covered with real scenarios
6. ⏳ **NEXT: Readiness/liveness probes** (using existing `/health` endpoint)
7. GitHub Actions CI pipeline — not started
8. Terraform + AWS EKS — not started
9. Monitoring — not started
10. Mock interview practice — not started

---

## Part 1 — ConfigMaps & Secrets: the "why" and who writes them

### Core problem they solve
Decouple configuration from the Docker image, so the **same built image** can run unchanged across dev/staging/prod — only the config paired with it changes. Without this, every config change forces an image rebuild, defeating "build once, deploy anywhere."

### ConfigMap vs Secret
| | ConfigMap | Secret |
|---|---|---|
| Purpose | Non-sensitive config | Sensitive values (passwords, keys) |
| Storage | Plain text | Base64-**encoded** (not encrypted!) |
| `kubectl describe` | Shows values | Hides values (`<set>`) |
| YAML field for values | `data:` | `stringData:` (auto-encoded) or `data:` (pre-encoded by you) |

**Interview gotcha:** Secrets are NOT encrypted by default — base64 is trivially reversible. Real protection needs encryption-at-rest on etcd, or an external secrets manager (Vault, AWS Secrets Manager) via an operator. Never assume a Secret object alone is "secure."

### Three ways to consume either one in a Pod
1. `envFrom` — inject every key as an env var (simplest, no renaming)
2. `env` + `valueFrom.configMapKeyRef` / `secretKeyRef` — one specific key, can rename it
3. Mount as a volume — each key becomes a file (used for full config files, not single values)

### Who actually writes these — dev vs DevOps

| | Decided by | Example |
|---|---|---|
| **Key names** (the contract) | **Developer** — hardcoded into app code (`os.environ.get("DB_HOST")`) | `DB_HOST`, `APP_ENV`, `EXTERNAL_API_KEY` |
| **Values** | **DevOps** — depends on actual infrastructure per environment | `"postgres-service"` (in-cluster) vs an RDS endpoint (prod) |

The developer's job ends at: "my code needs these env vars to exist." They don't know or need to know the real DB hostname, connection pool tuning, or which environment they're deploying to — that's infrastructure knowledge, which is DevOps's domain.

**Sorting rule for ConfigMap vs Secret:** ask *"if this leaked in a `kubectl describe` output or a screenshot, would it matter?"* — credentials → Secret, everything else → ConfigMap.

**Real-world note on Secret values:** actual secret values almost never get typed into a YAML file and committed to Git in a real pipeline — that's a security incident waiting to happen. In practice, values get injected at deploy time from a vault/secrets manager (Sealed Secrets, External Secrets Operator, SOPS), never hardcoded in version control. `stringData` placeholders are fine for practice only.

### Practice exercise worked through
Given a developer PR needing: `APP_ENV`, `LOG_LEVEL`, `MAX_CONNECTIONS`, `DB_HOST`, `DB_PASSWORD`, `EXTERNAL_API_KEY` —

**ConfigMap (non-sensitive):**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: devops-demo-app-config
data:
  APP_ENV: "local"
  LOG_LEVEL: "INFO"
  MAX_CONNECTIONS: "10"
  DB_HOST: "postgres-service"
```

**Secret (sensitive):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: devops-demo-app-secret
type: Opaque
stringData:
  DB_PASSWORD: "changeme123"
  EXTERNAL_API_KEY: "demo-external-key"
```

**Common YAML mistake caught while practicing:** indenting `data:` as a child of `metadata:` instead of as a sibling. Top-level fields (`apiVersion`, `kind`, `metadata`, `data`/`spec`) must all sit at column 0 — indentation is the #1 debugging suspect when a field "isn't being picked up."

**Wiring into Deployment:**
```yaml
envFrom:
  - configMapRef:
      name: devops-demo-app-config
  - secretRef:
      name: devops-demo-app-secret
```
The `name:` here must match `metadata.name` in the ConfigMap/Secret files exactly, or the Pod fails to start.

---

## Part 2 — Deployment ↔ ReplicaSet relationship

```
Deployment
    └── manages → ReplicaSet
                      └── manages → Pods
```

- **ReplicaSet**: keeps N Pods matching a label selector alive — no concept of versions/rollouts, just "how many Pods should exist right now."
- **Deployment**: layer on top adding rolling updates, rollback, and revision history. You never create ReplicaSets directly — the Deployment creates them for you.

### How an image update actually works
Changing the image tag doesn't edit existing Pods in place. The Deployment creates a **brand-new ReplicaSet** with the new template, scales it up while scaling the old one down, and keeps the old ReplicaSet around (scaled to 0, not deleted) for rollback purposes.

### Rollback mechanics
- Each ReplicaSet holds an **immutable snapshot** of the Pod template it was created with (image, env vars, everything in `spec.template`) — this never changes after creation.
- The Deployment's own `spec.template` only ever holds the **current** template — it is not a history store.
- `kubectl rollout undo` = Deployment copies an old ReplicaSet's frozen snapshot back into its own `spec.template`, then the normal reconciliation loop scales that ReplicaSet up and the current one down.
- **If a matching ReplicaSet already exists** (same template hash), it's reactivated — no new ReplicaSet is created. A new ReplicaSet is only created when the template doesn't match anything on record.
- `rollout undo` is not single-use — it always steps to the revision immediately before the current one, and each undo itself becomes a new revision number. Repeated plain `undo` toggles back and forth between the last two versions.
- To jump to a specific older revision (not just one step back): `kubectl rollout undo deployment/<name> --to-revision=N`
- Real limit on how far back you can go: `spec.revisionHistoryLimit` (default 10) — older ReplicaSets beyond that get garbage collected.

**Useful commands:**
```powershell
kubectl get replicasets
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=1
kubectl describe deployment <name> | Select-String "Image"
```

**Interview one-liner:** "ReplicaSet guarantees a Pod count. Deployment orchestrates ReplicaSets to give safe, versioned rollouts and rollback. Each ReplicaSet keeps an immutable snapshot of its Pod template — rollback works by copying an old snapshot back onto the Deployment and letting reconciliation scale it up, reusing the existing ReplicaSet rather than creating a new one."

---

## Part 3 — Service types

Choice of Service type is driven by **who needs to access the app and from where** — not by the nature of the app itself.

### 1. ClusterIP (default)
- Internal-only stable IP, reachable only from inside the cluster
- **Use for:** databases, internal microservices, anything that should never be hit directly from outside
```yaml
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080
```
**Scenario:** Frontend Pod calls `http://backend-service:8080`. Backend should never be internet-reachable — ClusterIP.

### 2. NodePort
- Everything ClusterIP has, plus opens a port (30000–32767) on every node for external access
- **Use for:** quick local/dev testing — not typically used in real production (raw node IPs, ugly high ports, no built-in load balancing across nodes)
```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```
**Scenario:** "My app is in the dev cluster, I need to access it from my laptop for testing." → NodePort. (This is what `devops-demo-app-service` uses today, on minikube.)

### 3. LoadBalancer
- Everything NodePort has, plus asks the cloud provider to provision a real external load balancer with a public IP/DNS
- **Use for:** production traffic from the real internet, on real cloud infrastructure
- Doesn't work on plain minikube (no cloud provider underneath); costs real money on AWS
```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 5000
```
**Scenario:** Online shop needs `https://shop.example.com` reachable by the public — LoadBalancer, backed by an AWS ELB/ALB.

### 4. ExternalName
- No routing to Pods at all — pure DNS aliasing (CNAME) to something outside the cluster
- **Use for:** pointing an in-cluster Service name at a real external resource (e.g. AWS RDS), so app code always calls a clean internal name regardless of whether the backing resource is in-cluster or external
```yaml
spec:
  type: ExternalName
  externalName: mydb.company.com
```
**Scenario:** Database lives outside Kubernetes as AWS RDS at `mydb.company.com`. App calls `database-service`, which DNS-aliases to the real external hostname.

### Decision table
| Type | Reachable from | Real use case |
|---|---|---|
| ClusterIP | Inside cluster only | Databases, internal services |
| NodePort | Outside cluster, node IP + high port | Local dev/testing |
| LoadBalancer | Public internet, clean IP/DNS | Production, real cloud infra |
| ExternalName | DNS alias only, no proxying | Pointing at an external (non-Pod) resource |

### Full example — online banking app, all four types in one architecture
```
INTERNET → LoadBalancer → Frontend Service → Frontend Pods
                                 │
                                 ↓
                        Backend Service (ClusterIP) → Backend Pods
                                 │
                                 ↓
                        DB Service (ClusterIP) → Database
                                 │
                        (or ExternalName if DB is external, e.g. RDS)
```

| Component | Service type | Why |
|---|---|---|
| Frontend | LoadBalancer | Users from internet need access |
| Backend | ClusterIP | Only frontend needs backend |
| Database | ClusterIP | Only backend needs database |
| External company API | ExternalName | Service exists outside cluster |

**Core principle to say in an interview:** *"You don't choose the Service type because of the application itself — you choose it based on who needs to access that application and where that access comes from."*

**Interview-style answer for "When would you use ClusterIP, NodePort and LoadBalancer?":**
> "ClusterIP is used for internal communication within the cluster. NodePort exposes the application through a port on the node, useful for simple testing or dev access. LoadBalancer is used for real external access through a cloud provider's load balancer, typically for production."

### Where this lands in the actual project
`devops-demo-app-service` stays `NodePort` for now (minikube/dev). When the roadmap reaches EKS, the same Service switches to `type: LoadBalancer` — same selector, same ports, one-line change — and AWS provisions a real DNS entry.

### Still to cover
The three-port structure in Service YAML (`port`, `targetPort`, `nodePort`) — flagged as the next thing to nail down once Service types are solid.

---

## NEXT STEPS

1. Finish understanding `port` vs `targetPort` vs `nodePort` in Service YAML
2. **Readiness/liveness probes** — wire `/health` into `deployment.yaml`
3. Debugging practice — break something on purpose, diagnose with `kubectl logs`/`describe`
4. GitHub Actions CI pipeline
5. Terraform + AWS EKS (cloud costs begin here)
6. Monitoring — Prometheus + Grafana
7. Full mock interview practice using all real debugging/demo stories collected so far

---

*Paste this file into a new chat to continue exactly where you left off.*
