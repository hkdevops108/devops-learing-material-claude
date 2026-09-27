# Kubernetes Study Notes

## Part 5 — Readiness & Liveness Probes

**Core distinction**

| | Liveness Probe | Readiness Probe |
|---|---|---|
| Question | Is this container alive, or stuck? | Is this container ready for traffic *right now*? |
| On failure | Kubernetes kills & restarts the container | Pod is pulled from the Service's endpoints — no restart |
| Use case | Deadlocks, infinite loops, hung processes that won't self-recover | Temporary unreadiness — booting, warming cache, waiting on DB, overloaded |

**Mental model:** liveness = "kill it and hope a fresh start fixes it" (last resort). Readiness = "don't send traffic yet/anymore, but leave it alone — it might recover." A Pod can be alive but not ready (e.g., up, not crashed, but waiting on a downstream dependency).

**Why both matter independently**
- Liveness only, no readiness → slow-starting Pods get traffic before they can handle it → errors on every rollout.
- Readiness only, no liveness → a truly hung Pod sits forever, marked not-ready but never restarted, silently eating capacity.

**Wiring into `deployment.yaml`**
```yaml
spec:
  containers:
    - name: devops-demo-app
      image: devops-demo-app:latest
      ports:
        - containerPort: 5000
      readinessProbe:
        httpGet:
          path: /health
          port: 5000
        initialDelaySeconds: 5
        periodSeconds: 10
        failureThreshold: 3
      livenessProbe:
        httpGet:
          path: /health
          port: 5000
        initialDelaySeconds: 15
        periodSeconds: 20
        failureThreshold: 3
```

**Field-by-field**
- `httpGet.path`/`port` — plain GET; 2xx–3xx = success, anything else (or timeout/refused) = failure. Alternatives: `tcpSocket` (plain port check), `exec` (run a command in-container) — use when there's no HTTP endpoint.
- `initialDelaySeconds` — grace period before first check. Liveness usually gets a *longer* delay than readiness (don't kill a Pod that's just slow to boot).
- `periodSeconds` — how often to re-check after that.
- `failureThreshold` — consecutive failures needed before it's counted as a real failure (not one blip).

**Interview-ready reasoning**
> Point liveness at something cheap and narrow (is the process responsive), and be conservative with failure thresholds — a liveness failure causes a disruptive restart. Readiness can be stricter and check real dependencies (e.g. "can I reach the DB"), because failing readiness just pulls the Pod out of rotation temporarily — much cheaper to get wrong.

**Shallow vs. deep health checks**
- Shallow `/health` — is the process running.
- Deep `/health` — checks DB connectivity, downstream services, etc.
- **Anti-pattern:** using a deep check for *liveness*. If the DB goes down, a deep liveness check would restart every app Pod for a problem restarts can't fix — potential cascading outage. Deep checks belong on **readiness**; liveness stays shallow.

**Next step (per roadmap)**
1. Wire probes into real `deployment.yaml`, watch `kubectl get pods` during a rollout — expect `READY 0/1` briefly on new Pods before `1/1`.
2. Item 3: debugging practice — deliberately break a probe (wrong port, 404 path) and diagnose live with `kubectl describe pod` and `kubectl logs`. Gives a concrete "bug I hit and fixed" interview story.

---
