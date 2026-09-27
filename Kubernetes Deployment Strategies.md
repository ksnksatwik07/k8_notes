# Kubernetes Deployment Strategies — Complete Mastery Guide
### From Beginner Rolling Updates to Production Progressive Delivery

---

# PART 1 — THE TWO NATIVE STRATEGIES

## 1–2. Recreate vs RollingUpdate

Every Deployment has exactly one of two native `.spec.strategy.type` values:

```yaml
spec:
  strategy:
    type: Recreate           # OR: RollingUpdate (the default)
```

| | Recreate | RollingUpdate |
|---|---|---|
| Old Pods | ALL terminated first | Terminated gradually, interleaved with new ones |
| New Pods | Created only after old ones are fully gone | Created alongside remaining old ones |
| Downtime | Yes — a gap with **zero** running Pods | No — some version is always serving, if configured correctly |
| Use case | Workloads that CANNOT have two versions running simultaneously (e.g., a schema-incompatible singleton, some legacy apps holding an exclusive lock/port) | The overwhelming majority of stateless workloads |

## 3. Rolling updates — mechanics recap

*(Full internal sequence diagram already covered in the Architecture masterclass, Part 5, Diagram F — this guide builds forward from that mechanical foundation into the fields and failure handling around it.)*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels: { app: my-app }
  template:
    metadata: { labels: { app: my-app } }
    spec:
      containers:
        - name: app
          image: myapp:v1
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 5
```

---

# PART 2 — THE EXACT v1 → v2 WALKTHROUGH, 5 REPLICAS

## Setup
```
maxSurge: 1          → at most 1 EXTRA Pod above desired count (6 total max, momentarily)
maxUnavailable: 0     → NEVER fewer than 5 Pods available at once
```

```
Step 0 (before update):
  ReplicaSet-v1: [v1][v1][v1][v1][v1]   (5/5 Ready)

Step 1: user runs `kubectl set image deployment/my-app app=myapp:v2`
  Deployment controller creates ReplicaSet-v2 (0 replicas so far)

Step 2: maxSurge allows +1 → scale ReplicaSet-v2 to 1
  ReplicaSet-v1: [v1][v1][v1][v1][v1]   (5 Ready)
  ReplicaSet-v2: [v2:starting]           (0 Ready yet — total: 6 Pods, 5 Ready)

Step 3: new v2 Pod passes its readinessProbe → becomes Ready
  ReplicaSet-v1: [v1][v1][v1][v1][v1]   (5 Ready)
  ReplicaSet-v2: [v2:Ready]              (1 Ready — total: 6 Pods, 6 Ready)

Step 4: NOW it's safe to remove an old Pod (maxUnavailable: 0 requires
        staying at/above 5 Ready at all times — we're at 6, so removing
        1 keeps us at 5)
  ReplicaSet-v1: [v1][v1][v1][v1]        (4 Ready)
  ReplicaSet-v2: [v2:Ready]               (1 Ready — total: 5 Pods, 5 Ready)

Step 5: maxSurge allows +1 again → scale ReplicaSet-v2 up again
  ReplicaSet-v1: [v1][v1][v1][v1]        (4 Ready)
  ReplicaSet-v2: [v2:Ready][v2:starting]  (1 Ready, 1 starting — total: 6 Pods, 5 Ready)

... this cycle repeats: wait for new Pod Ready → remove one old Pod →
    surge one more new Pod → wait for Ready → remove another old Pod ...

Step N (final):
  ReplicaSet-v1: []                        (0 — scaled to zero, KEPT for rollback)
  ReplicaSet-v2: [v2][v2][v2][v2][v2]       (5/5 Ready)
```

## The single gating rule, stated once, precisely
**A step that removes an old Pod never happens until the replacement new Pod has passed its `readinessProbe`** — not merely started, not merely `Running`. This is the entire mechanism behind "zero-downtime" rolling updates: total *available* capacity (Ready Pods, old + new combined) never drops below `desired - maxUnavailable`, at any single instant during the whole rollout.

```bash
kubectl rollout status deployment/my-app
# Waiting for deployment "my-app" rollout to finish: 2 out of 5 new replicas have been updated...
kubectl get replicasets -l app=my-app
# NAME             DESIRED   CURRENT   READY   AGE
# my-app-7d4f9c     0         0         0       10m    (old, v1, kept for rollback)
# my-app-8b2e1a     5         5         5       2m     (new, v2, active)
```

---

# PART 3 — maxSurge AND maxUnavailable IN DEPTH

## What each controls, precisely

```yaml
rollingUpdate:
  maxSurge: 25%          # can also be a percentage — rounds UP
  maxUnavailable: 25%     # rounds DOWN
```
With 5 replicas: `maxSurge: 25%` → `ceil(5 × 0.25) = 2` extra Pods allowed; `maxUnavailable: 25%` → `floor(5 × 0.25) = 1` Pod allowed to be missing at once. **The rounding direction is deliberate and asymmetric** — surge rounds up (favor having more capacity available during rollout), unavailable rounds down (favor not sacrificing more capacity than strictly necessary) — both choices lean toward preserving availability during the transition.

## Four meaningfully different configurations

| maxSurge | maxUnavailable | Behavior |
|---|---|---|
| `1` | `0` | Slowest, safest — always at full capacity or above, one extra Pod at a time (the v1→v2 walkthrough above) |
| `0` | `1` | Never exceeds desired count, but capacity dips below full during rollout — useful when the cluster has zero spare resource headroom to surge into |
| `25%` | `25%` | Faster rollout, some transient over-capacity AND some transient under-capacity simultaneously |
| `100%` | `0` | Full blue-green-like behavior WITHIN a single Deployment — doubles capacity momentarily, then cuts over entirely once all new Pods are Ready |

```
maxSurge: 0, maxUnavailable: 1  — the OPPOSITE shape from Part 2's walkthrough:

Step 0: [v1][v1][v1][v1][v1]           (5 Ready)
Step 1: REMOVE one old Pod FIRST (no room to surge — maxSurge: 0)
        [v1][v1][v1][v1]                (4 Ready — capacity DIPPED below 5)
Step 2: create a v2 Pod to replace it, wait for Ready
        [v1][v1][v1][v1][v2:Ready]       (5 Ready again)
Step 3: repeat — remove another old Pod BEFORE creating its replacement
```
This variant is appropriate specifically when the cluster genuinely cannot spare the resources for even one surged Pod — the trade-off is accepting brief reduced capacity instead.

---

# PART 4 — REVISIONS, ROLLOUT HISTORY, AND ROLLBACK

## 7–8. Deployment revisions and rollout history

Every time `.spec.template` changes, the Deployment controller creates (or reuses, if it matches an existing one exactly) a ReplicaSet and increments a revision counter, tracked via an annotation.

```bash
kubectl rollout history deployment/my-app
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         kubectl set image deployment/my-app app=myapp:v2
# 3         kubectl set image deployment/my-app app=myapp:v3
```
```bash
# to get meaningful CHANGE-CAUSE entries, annotate deliberately:
kubectl apply -f deployment.yaml --record          # deprecated but still common
# OR set it explicitly:
kubectl annotate deployment/my-app kubernetes.io/change-cause="Bump to v3 for security patch"
```

## 9. rollout status
```bash
kubectl rollout status deployment/my-app
# blocks and streams progress until the rollout completes or fails —
# exactly what a CI/CD pipeline step should wait on before proceeding
kubectl rollout status deployment/my-app --timeout=120s
```

## 10. rollout undo
```bash
kubectl rollout undo deployment/my-app                     # back to the PREVIOUS revision
kubectl rollout undo deployment/my-app --to-revision=1       # back to a SPECIFIC revision
```
**Mechanically, this is nothing new** — per the ReplicaSets guide, the old ReplicaSet was never deleted, only scaled to 0. `rollout undo` simply runs the exact same rolling-update machinery from Part 2, **in reverse**: scale the old (target) ReplicaSet back up, scale the current one down, gated by the same readiness checks — which is precisely why rollbacks are typically much faster than the original rollout: no new image pull is needed if nodes still have it cached, and the ReplicaSet's Pod template is already fully formed.

```
Before undo:  RS-v2 (5 Ready) ── RS-v1 (0, kept)
During undo:  RS-v2 scaling DOWN ── RS-v1 scaling UP (same maxSurge/maxUnavailable rules apply)
After undo:   RS-v2 (0, now kept for a POSSIBLE redo) ── RS-v1 (5 Ready)
```

## 11. Pause and resume — controlled, inspectable rollouts
```bash
kubectl rollout pause deployment/my-app
kubectl set image deployment/my-app app=myapp:v2      # change is recorded but NOT rolled out yet
# ... inspect, run manual checks, wait for a maintenance window ...
kubectl rollout resume deployment/my-app                # NOW the rolling update actually proceeds
```
Pausing is also a legitimate way to batch multiple changes (image + env vars + resource limits) into a **single** rollout instead of triggering a separate rolling update for each individual `kubectl set`/`kubectl patch` call.

---

# PART 5 — FAILURE HANDLING DURING ROLLOUTS

## 13–15. Failed deployments, bad images, and failed probes

**The critical safety property:** a rollout that produces Pods failing their readiness probe **simply stalls** — it does not proceed to remove more old Pods, and it does not roll back automatically either. It just... stops making progress, holding whatever mix of old/new Pods it had reached.

```bash
kubectl rollout status deployment/my-app
# Waiting for deployment "my-app" rollout to finish: 2 out of 5 new replicas
# have been updated...
# (this can hang here INDEFINITELY if the new Pods never become Ready)
```
```yaml
spec:
  progressDeadlineSeconds: 600    # after 10 minutes with no progress, mark the
                                    # Deployment condition as "ProgressDeadlineExceeded"
```
**`progressDeadlineSeconds` does not trigger an automatic rollback** — it only flips a `.status.conditions` entry to signal failure, for monitoring/alerting to catch. Kubernetes deliberately does not auto-rollback on its own; a human (or a CI/CD pipeline's own explicit logic) must decide to run `kubectl rollout undo`.

```bash
kubectl get deployment my-app -o jsonpath='{.status.conditions}'
# type: Progressing, status: "False", reason: ProgressDeadlineExceeded
```

## Diagram: a stuck rollout, frozen mid-transition
```
ReplicaSet-v1: [v1][v1][v1][v1]         (4 Ready — old version, still serving real traffic)
ReplicaSet-v2: [v2:CrashLoopBackOff]     (0 Ready — broken new version, stuck forever)

Total available: 4/5 desired — BELOW full capacity, but the Service
still routes ONLY to the 4 Ready v1 Pods (readiness-gated, per the
Services guide) — users experience reduced capacity, NOT v2's bugs,
because v2 never became Ready enough to receive any traffic at all
```
**This is precisely why readiness probes are the single most important safety mechanism in this entire guide** — a genuinely broken v2 image can sit there failing forever without ever serving a single real user request, because `Ready: False` (Pods masterclass §6–9) keeps it out of the Service's EndpointSlice entirely.

## 16. Backward compatibility
Because old and new versions run **simultaneously** during any rolling update (by design, per Part 2), **the new version must be able to coexist with the old one** for the rollout's duration — this has real implications beyond just the application code itself:
- **Database schema changes** must be backward-compatible with the *previous* application version for the rollout's duration (additive migrations — add a column, don't rename/drop one in the same deploy that also changes code to use the new name)
- **API contracts** between services must tolerate both old and new callers/responders simultaneously
- This is precisely why "expand-contract" migration patterns exist: expand the schema (additive, safe with both versions) in one deploy, migrate code to use it, THEN contract (remove old columns/fields) in a later, separate deploy once the old version is completely gone

---

# PART 6 — ADVANCED STRATEGIES (CONCEPTUAL — BEYOND NATIVE DEPLOYMENT OBJECTS)

**None of the following are native Kubernetes Deployment features** — they require either manual orchestration, a service mesh, or a dedicated progressive-delivery controller (Argo Rollouts, Flagger). This section explains the *concept* and *which components participate*, since a Deployment object alone cannot express any of them directly.

## Blue-Green

```
┌─────────────────────┐         ┌─────────────────────┐
│   Deployment "blue"    │         │   Deployment "green"    │
│   (v1, currently LIVE)  │         │   (v2, fully deployed,   │
│   5/5 Ready               │         │    fully tested, but      │
│                             │         │    receiving ZERO          │
│                             │         │    real traffic yet)        │
└───────────┬───────────┘         └───────────┬───────────┘
             │                                    │
             └───────────────┬────────────────────┘
                              ▼
                   ┌────────────────────┐
                   │   Service selector    │  ← currently points at "blue"
                   │   (app: blue)           │
                   └────────────────────┘

CUTOVER: instantly flip the Service's selector to "app: green"
         → ALL traffic switches to v2 in one atomic step, no gradual
           transition, no mixed-version window at all
```
**Which components participate:** two **complete, independent** Deployments (both at full replica count simultaneously — this costs 2x the steady-state resources during the transition window) and a single **Service** whose `selector` is edited to cut over. No Kubernetes-native object models "blue-green" directly — it's just two Deployments plus one deliberate Service edit, usually scripted/automated by a CI/CD pipeline or a tool like Argo Rollouts, which can manage this pattern declaratively.

**Rollback:** instant — just flip the Service selector back to "blue." This is blue-green's headline advantage over rolling updates: rollback has zero propagation delay, since the old version's full-capacity Deployment never stopped running.

## Canary

```
┌─────────────────────┐         ┌─────────────────────┐
│   Deployment "stable"  │         │   Deployment "canary"   │
│   (v1) — 9 replicas      │         │   (v2) — 1 replica         │
└───────────┬───────────┘         └───────────┬───────────┘
             │                                    │
             └───────────────┬────────────────────┘
                              ▼
                   ┌────────────────────┐
                   │      Service           │  ← selector matches BOTH
                   │  (matches a label        │     Deployments' shared label
                   │   common to BOTH)          │
                   └────────────────────┘
                              │
                    ~10% of traffic lands on
                    the single canary Pod
                    (pure statistical accident
                     of Service load-balancing
                     across 10 total matching
                     Pods, per the Services guide)
```
**Which components participate:** two Deployments sharing enough common labels for **one Service** to select Pods from both — the traffic **percentage** is controlled crudely by the **replica-count ratio** in this native-only approach (1 canary Pod out of 10 total ≈ roughly 10% of traffic, since the Service load-balances essentially evenly across all matching, Ready Pods). A **service mesh** (Istio, Linkerd) or an **Ingress controller with weighted routing** can instead control the traffic percentage directly and precisely, decoupled from replica counts entirely — this is the more precise, production-grade version of the same idea.

**Progression:** gradually shift the replica ratio (or mesh traffic weight) from canary toward stable being fully replaced — 10% → 25% → 50% → 100%, watching error rates/latency at each step, pausing or reverting immediately if the canary's metrics look worse than the stable baseline.

## A/B testing

```
Request arrives at Ingress/mesh
        │
        ▼
Route based on a REQUEST ATTRIBUTE (a header, a cookie, a user ID
hash) — NOT randomly, unlike canary's statistical traffic split
        │
    ┌────┴────┐
    ▼           ▼
 Version A   Version B
 (users in    (users in
  cohort A)    cohort B)
```
**Which components participate:** this requires **Layer 7, content-aware routing** — a plain Kubernetes Service (L4, per the Services guide) has no concept of headers or cookies at all; A/B testing needs either an Ingress controller with routing-rule annotations, or a service mesh's request-routing rules, to make the version choice based on *who's asking*, not a random percentage split. **This is a fundamentally different goal from canary** — canary is about *safety* (catch bugs on a small slice before full rollout); A/B is about *measurement* (compare two genuinely different versions' business/UX outcomes on deliberately chosen user cohorts), even after both are considered "fully validated."

## Progressive delivery (the general, tooling-assisted pattern)

```
┌─────────────────────────────────────────────────────────┐
│                Argo Rollouts / Flagger                       │
│  (a CUSTOM RESOURCE + controller replacing the native          │
│   Deployment object, per the Architecture masterclass's         │
│   CRD-extensibility pattern)                                      │
│                                                                    │
│  Watches: real metrics (Prometheus error rate, latency,             │
│           success rate) DURING the rollout                          │
│  Acts: automatically advances traffic percentage on good              │
│        metrics, automatically PAUSES or ROLLS BACK on bad ones          │
└─────────────────────────────────────────────────────────────┘
```
This is canary/blue-green **automated and metric-gated**, rather than manually watched and manually advanced — the controller itself decides, based on real observed metrics, whether to proceed, pause, or abort, closing the loop that native Kubernetes Deployments (and even hand-rolled canary/blue-green) leave entirely to a human watching dashboards.

---

# PART 7 — STRATEGY COMPARISON TABLE

| Strategy | Native to Kubernetes? | Downtime | Resource cost during transition | Rollback speed | Traffic control precision |
|---|---|---|---|---|---|
| Recreate | Yes | Yes | None extra | Fast (same mechanism, reversed) | N/A (all-or-nothing) |
| RollingUpdate | Yes | No | Small (maxSurge) | Fast, reuses old ReplicaSet | None — proportional to Pod mix only |
| Blue-Green | No (manual/tooling) | No | 2x (both full versions running) | Instant (Service selector flip) | None — all-or-nothing cutover |
| Canary | No (manual/mesh/tooling) | No | Small (extra canary replicas) | Fast (scale canary to 0) | Coarse (native) to precise (mesh) |
| A/B | No (mesh/Ingress required) | No | Depends on cohort split | Depends on routing config | Precise, attribute-based, not random |
| Progressive delivery (Argo/Flagger) | No (CRD + controller) | No | Similar to canary | Fast, automated | Precise, and **automated** on real metrics |

---

# PART 8 — PRODUCTION CHECKLIST

1. **Always set a `readinessProbe`** — without one, a rolling update has no reliable signal for "is the new Pod actually safe to receive traffic," and the entire safety mechanism in Part 5 doesn't function.
2. **Set `progressDeadlineSeconds`** deliberately, and alert on `ProgressDeadlineExceeded` — a stuck rollout with no alerting can sit half-migrated for hours before anyone notices.
3. **Never assume automatic rollback** — Kubernetes never rolls back on its own; wire `kubectl rollout undo` into your pipeline's failure path explicitly, triggered by `kubectl rollout status`'s exit code or the progress-deadline condition.
4. **Use expand-contract for schema changes** — never ship a breaking schema change in the same deploy that also requires it, given old and new versions coexist during any rolling update.
5. **Tune `maxSurge`/`maxUnavailable`** deliberately based on actual cluster headroom — don't leave the defaults unexamined for workloads where either resource pressure or availability margins are tight.
6. **Meaningful `change-cause` annotations** on every Deployment update — `rollout history` with no context is nearly useless during an actual incident at 2 AM.
7. **For genuinely high-stakes rollouts**, consider Argo Rollouts/Flagger over hand-rolled canary — automated metric-gated promotion catches regressions faster and more reliably than a human watching a dashboard.

---

# PART 9 — FAILURE SCENARIOS AND ROLLBACK EXERCISES

## Exercise: induce a stuck rollout, then roll back
```bash
kubectl create deployment demo --image=nginx:1.25 --replicas=5
kubectl rollout status deployment/demo
kubectl set image deployment/demo nginx=nginx:this-tag-does-not-exist
kubectl rollout status deployment/demo --timeout=30s
# → times out; ImagePullBackOff on the new ReplicaSet's Pods
kubectl get replicasets -l app=demo
# → old RS still has Ready Pods; new RS stuck at 0 Ready
kubectl rollout undo deployment/demo
kubectl rollout status deployment/demo
# → recovers immediately, since the old RS never actually lost its Pods
```

## Exercise: readiness-probe-gated rollout, deliberately broken
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata: { name: demo2 }
spec:
  replicas: 5
  strategy:
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  selector: { matchLabels: { app: demo2 } }
  template:
    metadata: { labels: { app: demo2 } }
    spec:
      containers:
        - name: web
          image: nginx
          readinessProbe:
            httpGet: { path: /this-path-does-not-exist, port: 80 }
            periodSeconds: 5
EOF
kubectl rollout status deployment/demo2 --timeout=30s
# → never completes; new Pods run but NEVER pass readiness (404 on
#   every probe) — demonstrates a rollout stalling safely rather than
#   forcing broken Pods into service
kubectl get pods -l app=demo2
# → new Pods show READY 0/1, Running (not Crashing — the CONTAINER is
#   fine, only the PROBE PATH is wrong) — an important distinction from
#   CrashLoopBackOff
```

---

# INTERVIEW QUESTIONS

**Fundamentals**
1. What's the fundamental difference between Recreate and RollingUpdate?
2. Walk through, step by step, what happens updating 5 replicas from v1 to v2 with `maxSurge: 1, maxUnavailable: 0`.
3. Why does surge rounding go up while unavailable rounding goes down for percentage values?

**Rollback mechanics**
4. What object does `kubectl rollout undo` actually operate on, mechanically?
5. Why are rollbacks typically faster than the original rollout?
6. Does Kubernetes ever roll back a Deployment automatically? What DOES happen on a failed rollout?

**Safety**
7. What's the single mechanism that prevents a broken new version from receiving real traffic during a rolling update?
8. What does `progressDeadlineSeconds` actually do, and what does it NOT do?
9. Why must application changes deployed via rolling update be backward-compatible, even temporarily?

**Advanced strategies**
10. Why is blue-green not a native Kubernetes Deployment feature — what actually implements it?
11. How is canary traffic percentage controlled using only native Kubernetes objects, and why is that method imprecise?
12. What's the fundamental difference in GOAL between canary and A/B testing, even though both involve running two versions simultaneously?
13. What does a tool like Argo Rollouts or Flagger add on top of hand-rolled canary/blue-green?

**Production**
14. Why is "expand-contract" the standard pattern for database schema changes alongside rolling updates?
15. A rollout is stuck with new Pods `Running` but never `Ready` — is this the same failure class as CrashLoopBackOff? Explain the difference.

---

*This guide covers Kubernetes deployment strategies from native Recreate/RollingUpdate mechanics through blue-green, canary, A/B, and metric-gated progressive delivery — the complete arc from beginner rolling updates to production-grade release engineering.*
