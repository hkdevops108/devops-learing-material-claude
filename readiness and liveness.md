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
   Here's what each line does, with the actual numbers plugged in so you can see the timeline it creates:

**readinessProbe**
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```
- Kubernetes sends `GET http://<pod-ip>:5000/health`. Any 2xx–3xx response = pass; anything else (4xx/5xx, timeout, connection refused) = fail.
- `initialDelaySeconds: 5` — wait 5 seconds after the container starts before the *first* check. This gives the app a moment to boot before judging it.
- `periodSeconds: 10` — after that first check, re-check every 10 seconds, forever, for the life of the Pod.
- `failureThreshold: 3` — it takes **3 consecutive failures** before the Pod is marked "not ready." One bad response doesn't flip it — that avoids yanking a Pod out of rotation over a single blip.
- On failure: the Pod is removed from the Service's endpoint list (stops receiving traffic), but the container keeps running untouched. It can rejoin automatically the moment `/health` starts passing again.

**livenessProbe**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3
```
- Same request, same endpoint — but the consequence is different.
- `initialDelaySeconds: 15` — longer than readiness's 5s. Deliberate: you don't want the liveness check judging the app before it's even had a fair chance to come up, since a liveness failure has a much more drastic consequence (a restart) than a readiness failure does.
- `periodSeconds: 20` — checks every 20 seconds, also slower-paced than readiness. Liveness doesn't need to react fast; it's a safety net for "is this truly stuck," not a fine-grained traffic gate.
- `failureThreshold: 3` — again, 3 consecutive misses before action is taken.
- On failure: Kubernetes **kills and restarts the container** — a much bigger hammer, so the checks leading up to it are intentionally more forgiving (longer delay, slower period).

**The pattern to notice:** both probes hit the *same* `/health` endpoint here, but liveness is tuned to be slower and more patient (15s delay, 20s period) than readiness (5s delay, 10s period). That's intentional — it means a temporary hiccup will almost always trip readiness (pull from traffic, low cost) well before it could ever trip liveness (restart, high cost). Readiness acts as an early buffer that protects liveness from overreacting.

One thing worth flagging as you go further: since both probes point at the exact same `/health` path, this only works well if that endpoint is a **shallow** check (just "is the process up"). If `/health` here also pings the database, you'd have the deep-check-on-liveness anti-pattern from Part 5 — DB blips would start restarting Pods needlessly. Worth checking what your actual `/health` handler does.

Want to move on to watching this in action with `kubectl get pods` during a rollout, or go straight to the debugging exercise?
my question: 
both probes point to same /health but how would be the ouput is it not ready or it is not alive ?

Good question — the key is that Kubernetes treats them as **two completely independent checks**, even though they happen to hit the same URL. It doesn't "share" a result between them. Each probe runs on its own clock and keeps its own failure count, so you get *both* signals, just possibly at different times.

Walk through what happens if `/health` starts failing and stays down:

**Timeline (right after Kubernetes stops getting a good response):**

- Readiness is checking every 10s, needs 3 fails in a row → it hits failure #3 at roughly **30 seconds** after the first miss.
  - Effect: Pod is pulled from the Service's endpoints. `kubectl get pods` shows `READY 0/1` while `STATUS` still says `Running`.
- Liveness is checking every 20s, needs 3 fails in a row → it hits failure #3 at roughly **60 seconds** after the first miss.
  - Effect: kubelet kills the container and restarts it. `kubectl get pods` shows `RESTARTS` increment, and briefly `STATUS: Running` → `CrashLoopBackOff`/`Error` if it keeps happening.

So in this setup, **readiness fails first, then liveness fails** — because readiness's threshold is reached sooner (faster period, shorter delay). That's not a coincidence, it's the intended design: the Pod gets pulled from traffic well before it gets killed, so users are protected before the more drastic action kicks in.

**How you'd actually see this in output:**

`kubectl get pods`:
```
NAME                    READY   STATUS    RESTARTS   AGE
devops-demo-app-xyz     0/1     Running   0          45s   ← readiness failed, still alive
devops-demo-app-xyz     0/1     Running   1          75s   ← liveness failed too, got restarted
```

`kubectl describe pod devops-demo-app-xyz` — this is where you'd see *which* probe fired, in the Events section, something like:
```
Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 500
Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 500
Normal   Killing    Container devops-demo-app failed liveness probe, will be restarted
```

That `describe pod` output is the real answer to "how do I know which one fired" — `get pods` only shows you the *consequences* (readiness → `0/1`, liveness → restart count going up), but `describe pod` names the probe explicitly in its Events log.

This is actually a good moment to point at the debugging exercise from Part 5 — if you deliberately break `/health` right now and watch both `get pods` and `describe pod`, you'll see exactly this sequence play out live rather than just reading about it. Want to do that?

my question:
here pod is pulled out from services endpoint means what will that pod deleted and creates new and also kubelet kills container and restart it means restart the same conatiner or new?
---Good distinction to nail down — these are two very different operations, and neither of them creates a new Pod.

**"Pulled from the Service's endpoints" (readiness failure)**

Nothing about the Pod itself changes. The Pod isn't touched, deleted, or restarted — it keeps running exactly as it was.

What actually happens: there's a separate object called an **Endpoints** (or **EndpointSlice**) that the Service uses as its routing table — basically a list of "which Pod IPs are currently valid targets." When readiness fails, Kubernetes just **removes that Pod's IP from the list**. The Service now skips it when load-balancing traffic. That's it — it's a routing-table edit, not an action on the Pod.

The moment `/health` starts passing again, the Pod's IP gets added back to that list and it starts receiving traffic again — automatically, no restart involved.

Think of it like a waiter being told "don't seat anyone at table 5 right now" — table 5 isn't removed from the restaurant, it's just skipped when the host is seating people. Nothing happens to the table itself.

**"Kubelet kills and restarts the container" (liveness failure)**

This is different — this one does act on the container, but not the Pod.

- The **Pod** stays the same object — same name, same IP address, same place in the cluster, same entry in `kubectl get pods`.
- The **container inside it** is what gets killed and restarted — same image, same spec, fresh process. It's like restarting a service on a VM: the VM doesn't go anywhere, just the process inside it gets killed and relaunched.
- You can see this directly: `RESTARTS` in `kubectl get pods` increments (1, 2, 3...), but the Pod's `AGE` keeps counting from when the Pod was originally created — it doesn't reset. That's your tell that it was a container restart, not a new Pod.

**So to directly answer both parts:**
- Readiness failing → Pod is **not deleted or restarted at all**. Only its entry in the traffic-routing list is toggled off/on.
- Liveness failing → **same Pod, same container definition** — kubelet just kills the running container process and starts a fresh one in its place. No new Pod is created.

A new Pod only enters the picture in totally different scenarios — like a Deployment rollout, or if the Pod got evicted from its Node entirely (e.g. Node runs out of resources or dies). Neither probe does that on its own.

Want me to add this Q&A to your notes file, and then move into the debugging exercise?
