# Kubernetes Deployments — Complete Masterclass (Beginner → Production)

---

## PART 1 — CONCEPTS

### 1. What a Deployment is

A **Deployment** is a Kubernetes controller object that manages the desired state of a set of identical Pods over time. You describe *what you want* (image, replica count, update strategy), and the Deployment controller continuously works to make reality match that description.

A Deployment doesn't run Pods directly — it manages a **ReplicaSet**, which in turn manages the **Pods**. This layering is what enables rolling updates, rollback, and history.

```
Deployment  →  manages →  ReplicaSet  →  manages →  Pods
 (strategy,                 (replica                (actual
  history,                   count for                running
  rollback)                  one version)             containers)
```

### 2. Why Deployments exist

A bare Pod has no self-healing, no scaling, no update strategy. If a Pod dies, it's gone. If you want to update your app's image, you'd have to manually delete and recreate Pods, causing downtime and giving you no rollback path.

Deployments solve:

- **Self-healing** — ReplicaSet ensures N Pods always exist
- **Declarative updates** — change the image/spec, Deployment handles the transition
- **Zero-downtime rollouts** — `RollingUpdate` strategy replaces Pods gradually
- **Rollback** — every change creates a new ReplicaSet revision; you can revert
- **Scaling** — change `replicas`, controller reconciles

### 3. Deployment vs Pod

| | Pod | Deployment |
|---|---|---|
| Self-healing | No — dies, stays dead | Yes — controller recreates |
| Scaling | Manual, one at a time | Declarative `replicas` field |
| Updates | Manual delete/recreate | Rolling update built in |
| Rollback | Not possible | Built-in revision history |
| Use directly in prod? | Rarely (debugging only) | Yes — standard workload object |

### 4. Deployment vs ReplicaSet

A **ReplicaSet**'s only job: ensure exactly N Pods matching a label selector exist. It has no concept of "versions" or "rollout strategy."

A **Deployment** sits above ReplicaSet and adds: versioned rollout history, rolling update orchestration, rollback. When you update a Deployment's Pod template (e.g., change the image), it creates a **new** ReplicaSet and gradually shifts replica counts from the old ReplicaSet to the new one — that's what a rollout actually *is*, mechanically.

```
Deployment "myapp" (desired image: v2)
 ├── ReplicaSet myapp-7d9f8 (image v1) → replicas: 0   [old, kept for rollback]
 └── ReplicaSet myapp-5c8b1 (image v2) → replicas: 3   [current]
```

You should almost never create a bare ReplicaSet — always go through Deployment.

### 5. Deployment Architecture

```
┌──────────────────────────────────────────────────────────┐
│                       Deployment                          │
│  spec: replicas=3, strategy=RollingUpdate, template=...   │
└───────────────────────────┬────────────────────────────────┘
                             │ owns (via labels + ownerReferences)
                             ▼
              ┌──────────────────────────────┐
              │  ReplicaSet (current revision) │
              │  spec: replicas=3               │
              └───────────────┬────────────────┘
                               │ owns
             ┌─────────────────┼─────────────────┐
             ▼                 ▼                 ▼
          Pod 1              Pod 2              Pod 3
```

Ownership is tracked via `ownerReferences` — this is how `kubectl delete deployment` cascades down to delete ReplicaSets and Pods, and how the Deployment controller and ReplicaSet controller each "own" and reconcile only their own children.

---

## PART 2 — YAML STRUCTURE, FIELD BY FIELD

### 6-13. Deployment YAML Structure — Basic Example

```yaml
apiVersion: apps/v1                 # Deployments live in the "apps" API group
kind: Deployment
metadata:
  name: myapp-deployment             # Name of the Deployment object itself
  labels:
    app: myapp                        # Labels on the Deployment object (not the Pods!)
spec:
  replicas: 3                         # (7) Desired number of Pod replicas
  selector:                           # (8) How the Deployment finds "its" Pods
    matchLabels:
      app: myapp                       # MUST match template.metadata.labels — immutable after creation
  template:                           # (9) The Pod template — stamped out N times
    metadata:
      labels:
        app: myapp                     # (10) Labels applied to each created Pod — must satisfy selector
    spec:
      containers:                     # (11) Container list, same as a bare Pod spec
      - name: myapp                    
        image: myapp:1.0                # (12) Image + tag
        ports:
        - containerPort: 8080            # (13) Documented container port
```

**Why `selector` and `template.metadata.labels` must match:** the Deployment controller (and the ReplicaSet it creates) uses a **label selector** to find which Pods it's responsible for. If they didn't match, the controller couldn't identify its own Pods — this is why Kubernetes rejects a Deployment where `selector` doesn't match `template.metadata.labels`, and why `selector` is **immutable** once created (changing it would orphan existing Pods).

### 14-18. Strategy

```yaml
spec:
  strategy:
    type: RollingUpdate               # (14) or "Recreate"
    rollingUpdate:
      maxSurge: 1                      # (17) how many EXTRA Pods above `replicas` allowed during update
      maxUnavailable: 0                # (18) how many Pods can be unavailable during update
```

#### 15. RollingUpdate (default)

Gradually replaces old Pods with new ones, respecting `maxSurge`/`maxUnavailable`, so the app stays available throughout.

```
Start:     [v1][v1][v1]                     (replicas=3)
Step 1:    [v1][v1][v1][v2]                 (maxSurge=1 → 4 total allowed)
Step 2:    [v1][v1]    [v2]  → terminate 1 v1, start 1 v2
Step 3:    [v1]    [v2][v2]
Step 4:        [v2][v2][v2]                 done
```

#### 16. Recreate

Kills **all** old Pods first, then creates all new ones. Causes downtime — used when the app can't tolerate two versions running simultaneously (e.g., schema-incompatible DB migrations, singleton workloads).

```
Start:     [v1][v1][v1]
Step 1:    [ ] [ ] [ ]          ← all terminated first
Step 2:    [v2][v2][v2]         ← then all created
```

#### 17. maxSurge
How many Pods **above** `replicas` are allowed temporarily during rollout. Can be a number (`1`) or percentage (`25%`). Higher = faster rollout, more resource usage during transition.

#### 18. maxUnavailable
How many Pods **below** `replicas` are tolerated during rollout. `0` means always fully available (requires `maxSurge` > 0 to make progress). Higher = faster rollout, less availability guarantee during transition.

### 19. Revision History

```yaml
spec:
  revisionHistoryLimit: 10        # how many old ReplicaSets to KEEP (scaled to 0) for rollback
```

Every time `template` changes, a new ReplicaSet revision is created; old ones are kept (scaled to 0 replicas) up to this limit, enabling `kubectl rollout undo`.

---

## PART 3 — INTERNALS: WHAT HAPPENS ON `kubectl apply -f deployment.yaml`

```
┌──────────┐   ┌────────────┐   ┌──────┐   ┌───────────────────┐   ┌────────────┐   ┌─────┐   ┌──────────┐   ┌──────────────────┐
│ kubectl  │──►│ API Server │──►│ etcd │──►│ Deployment         │──►│ ReplicaSet │──►│ Pods │──►│ Scheduler │──►│ kubelet + CRI      │
│  apply   │   │ (validate, │   │(store│   │ Controller         │   │ Controller │   │(objs)│   │ (assigns  │   │ (pulls image,      │
│          │   │ admission) │   │ etc) │   │ (watch loop)       │   │(watch loop)│   │      │   │  node)    │   │  starts container) │
└──────────┘   └────────────┘   └──────┘   └───────────────────┘   └────────────┘   └─────┘   └──────────┘   └──────────────────┘
```

Step by step:

1. **kubectl** reads the YAML, converts to JSON, sends a `PATCH`/`POST` request to the API server (with `apply`, it's a server-side apply / strategic merge).
2. **API Server** authenticates, authorizes (RBAC), runs admission controllers/webhooks, validates schema.
3. Object persisted to **etcd** — this is the single source of truth. The Deployment object now exists with `spec.replicas=3`, etc.
4. **Deployment controller** (a control loop inside `kube-controller-manager`) is watching the API server for Deployment changes. It notices the new/changed Deployment.
5. It computes: "does a ReplicaSet already exist matching this Pod template hash? If not, create one." It computes a `pod-template-hash` label from the Pod template and creates a new ReplicaSet named `<deployment>-<hash>` with the desired replica count, owned by the Deployment (`ownerReferences`).
6. If this is an update (not first creation), the Deployment controller orchestrates the rollout: incrementally patches the **old** ReplicaSet's `replicas` down and the **new** ReplicaSet's `replicas` up, respecting `maxSurge`/`maxUnavailable`, checking new Pods' readiness before continuing.
7. **ReplicaSet controller** (separate control loop) watches ReplicaSet objects. It notices `spec.replicas` doesn't match the actual Pod count for its selector, and creates (or deletes) **Pod objects** to close the gap.
8. Each new Pod object is created in `Pending` phase with no `nodeName`.
9. **Scheduler** watches for unscheduled Pods, runs filter+score algorithm, and binds each Pod to a Node (`spec.nodeName` PATCH).
10. **kubelet** on that node notices a Pod bound to it, and (as covered in the Pods chapter) talks to the **CRI/container runtime** to create the sandbox, pull images, and start containers.
11. kubelet reports Pod status back to the API server continuously; ReplicaSet controller and Deployment controller watch this status to know when to proceed (e.g., wait for new Pods to become `Ready` before terminating more old Pods).

This is **reconciliation**, not a one-shot script: every controller here runs an infinite `observe → diff → act` loop, constantly comparing desired state (spec) to observed state (status), and this is why Kubernetes self-heals even from unrelated failures (e.g., a node dying triggers the same reconciliation path to recreate lost Pods elsewhere).

### 29. Controller Reconciliation Loop (generic pattern, applies to all controllers above)

```
loop forever:
    desired = read spec (from API server / informer cache)
    actual  = read status (from API server / informer cache)
    if desired != actual:
        take minimal action to converge (create/delete/patch)
    sleep / wait for next watch event
```

---

## PART 4 — ROLLOUT MECHANICS

### 20-23. Rollout, rollout status, rollout history, rollback

Every change to `spec.template` triggers a new **rollout** (a new ReplicaSet revision). Kubernetes tracks these via the `deployment.kubernetes.io/revision` annotation on each ReplicaSet.

```bash
kubectl rollout status deployment/myapp-deployment
# Waiting for deployment "myapp-deployment" rollout to finish: 2 of 3 updated replicas are available...
# deployment "myapp-deployment" successfully rolled out

kubectl rollout history deployment/myapp-deployment
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         kubectl set image deployment/myapp-deployment myapp=myapp:2.0
# 3         kubectl set image deployment/myapp-deployment myapp=myapp:3.0

kubectl rollout history deployment/myapp-deployment --revision=2
# shows the full Pod template diff for that revision

kubectl rollout undo deployment/myapp-deployment
# rolls back to the PREVIOUS revision

kubectl rollout undo deployment/myapp-deployment --to-revision=1
# rolls back to a SPECIFIC revision
```

**Internals of rollback:** `rollout undo` doesn't "undo" anything magically — it simply copies the Pod template from the target old ReplicaSet back into the Deployment's `spec.template`, which triggers a completely normal rolling update (old ReplicaSet scales back up, current one scales down).

To make `CHANGE-CAUSE` populated, annotate your applies:
```bash
kubectl apply -f deployment.yaml --record   # deprecated but still works in most versions
# OR set it explicitly:
kubectl annotate deployment/myapp-deployment kubernetes.io/change-cause="bump to v2.0"
```

### 24. Pause / Resume

```bash
kubectl rollout pause deployment/myapp-deployment
# make multiple changes (image + resources + env) without triggering a rollout for each
kubectl set image deployment/myapp-deployment myapp=myapp:2.1
kubectl set resources deployment/myapp-deployment -c myapp --limits=cpu=500m,memory=512Mi
kubectl rollout resume deployment/myapp-deployment
# NOW a single rollout happens with all batched changes
```

### 25-27. Scaling, image updates, configuration updates

```bash
# Scaling
kubectl scale deployment/myapp-deployment --replicas=5
# Just patches spec.replicas — ReplicaSet controller creates/deletes Pods directly,
# NO new ReplicaSet revision is created (scaling isn't a "change" to the Pod template)

# Image updates
kubectl set image deployment/myapp-deployment myapp=myapp:2.0
# Patches spec.template.spec.containers[0].image → triggers a NEW ReplicaSet + rolling update

# Config updates (env/configmap reference/resources/etc.)
kubectl edit deployment/myapp-deployment
# Any change under spec.template → new ReplicaSet revision + rollout
# NOTE: changing a ConfigMap's DATA (not the Deployment's reference to it) does NOT
# trigger a rollout by itself — Pods won't pick up new values until restarted!
```

⚠️ **Important gotcha:** if you update a `ConfigMap` in place (same name, new data), Pods using it via `envFrom`/`env` **do not automatically restart** — env vars are snapshotted at container start. Volume-mounted ConfigMaps *do* eventually update the mounted file (kubelet syncs periodically), but the app process usually doesn't reload it unless it watches the file. This is why many teams embed a content hash of the ConfigMap into a Pod annotation to force a rollout on config change (see "Production Deployment" YAML below).

### 28. Deployment Lifecycle (state machine)

```
Created → Progressing → Complete
              │
              ├─→ Progressing (new rollout triggered again)
              │
              └─→ Failed (progressDeadlineSeconds exceeded)
```

```yaml
spec:
  progressDeadlineSeconds: 600   # if rollout doesn't progress in 10 min, mark condition as Failed
```

Check via:
```bash
kubectl get deployment myapp-deployment -o jsonpath='{.status.conditions}'
# Types: Available, Progressing
```

---

## PART 5 — COMPLETE YAML EXAMPLES (line by line)

### A. Basic Deployment (3 replicas)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: basic-deployment
spec:
  replicas: 3                        # desired Pod count
  selector:
    matchLabels:
      app: basic-app                  # must match template labels
  template:
    metadata:
      labels:
        app: basic-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Internally:** creates 1 ReplicaSet with `replicas: 3`; ReplicaSet controller creates 3 Pod objects; scheduler places each; kubelet on each node starts nginx.

### B. Rolling Update Strategy (explicit)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-deployment
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1              # at most 5 Pods total during rollout
      maxUnavailable: 1        # at most 3 available Pods during rollout (min)
  selector:
    matchLabels:
      app: rolling-app
  template:
    metadata:
      labels:
        app: rolling-app
    spec:
      containers:
      - name: app
        image: myapp:1.0
```

**Internally:** on an image change, Deployment controller scales new RS up by `maxSurge`, waits for those Pods to be `Ready`, then scales old RS down by an equivalent amount respecting `maxUnavailable`, repeating until fully migrated.

### C. Recreate Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recreate-deployment
spec:
  replicas: 3
  strategy:
    type: Recreate                # no maxSurge/maxUnavailable — not applicable
  selector:
    matchLabels:
      app: recreate-app
  template:
    metadata:
      labels:
        app: recreate-app
    spec:
      containers:
      - name: app
        image: legacy-app:1.0
```

**Internally:** old ReplicaSet scaled to 0 first (all Pods terminated), controller waits for them to fully terminate, **then** scales new ReplicaSet to `replicas`. Guarantees no version overlap — at the cost of downtime.

### D. Resource Limits

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resource-app
  template:
    metadata:
      labels:
        app: resource-app
    spec:
      containers:
      - name: app
        image: myapp:1.0
        resources:
          requests:
            cpu: "200m"           # scheduler uses this for bin-packing across nodes
            memory: "256Mi"
          limits:
            cpu: "500m"            # CFS throttling cap
            memory: "512Mi"         # OOM-kill cap
```

**Internally:** scheduler sums `requests` across all Pods on a candidate node to decide if a new Pod fits (`node.allocatable - sum(requests of existing Pods) >= new Pod's requests`). Rolling updates with `maxSurge` temporarily need extra headroom on nodes — undersized clusters can get stuck `Pending` mid-rollout.

### E. Probes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: probe-app
  template:
    metadata:
      labels:
        app: probe-app
    spec:
      containers:
      - name: app
        image: myapp:1.0
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          periodSeconds: 10
          failureThreshold: 3
```

**Why this matters for Deployments specifically:** during a rolling update, the Deployment controller only counts a new Pod as "available" (and proceeds terminating an old Pod) once it passes `readinessProbe` **and** stays ready for `minReadySeconds`. Without a correct readiness probe, Kubernetes might route traffic to (or count as healthy) a Pod that isn't actually ready to serve, causing errors during deploys.

```yaml
  minReadySeconds: 10   # Pod must stay Ready this long before counted as "available"
```

### F. Environment Variables

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: env-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: env-app
  template:
    metadata:
      labels:
        app: env-app
    spec:
      containers:
      - name: app
        image: myapp:1.0
        env:
        - name: ENVIRONMENT
          value: "production"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

### G. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: configmap-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cm-app
  template:
    metadata:
      labels:
        app: cm-app
    spec:
      containers:
      - name: app
        image: myapp:1.0
        envFrom:
        - configMapRef:
            name: app-config       # injects LOG_LEVEL and MAX_CONNECTIONS as env vars
```

### H. Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=       # base64-encoded value (NOT encryption — just encoding!)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secret-app
  template:
    metadata:
      labels:
        app: secret-app
    spec:
      containers:
      - name: app
        image: myapp:1.0
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
```

**Internally:** Secret data is base64-encoded (not encrypted) in etcd by default — enable **encryption at rest** at the cluster level for real protection. kubelet fetches the Secret at Pod start and injects the decoded value as a container env var.

### I. Production Deployment (all combined)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 4
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0            # zero-downtime: never drop below desired count
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v3
      annotations:
        checksum/config: "a1b2c3d4"   # forces rollout when ConfigMap content changes
    spec:
      terminationGracePeriodSeconds: 30
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: kubernetes.io/hostname   # spread replicas across nodes
      containers:
      - name: myapp
        image: myregistry.io/myapp:1.4.2             # pinned version, never :latest
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: myapp-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secret
              key: db-password
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet: { path: /readyz, port: 8080 }
          periodSeconds: 5
        livenessProbe:
          httpGet: { path: /healthz, port: 8080 }
          periodSeconds: 10
          failureThreshold: 3
        securityContext:
          runAsNonRoot: true
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
      imagePullSecrets:
      - name: registry-credentials
```

Every line here maps to a lesson already covered — this is the "production checklist," expressed as YAML.

---

## PART 6 — FAILURE SCENARIOS & TROUBLESHOOTING

### Rollout stuck ("Progressing" never completes)

```bash
kubectl rollout status deployment/myapp   # hangs
kubectl get pods -l app=myapp             # check new Pods' state
kubectl describe pod <new-pod>            # ImagePullBackOff? CrashLoop? Pending?
kubectl describe deployment myapp | grep -A10 Conditions
```

Common causes:
- New image doesn't exist / typo → `ImagePullBackOff`
- New Pods fail readiness probe → rollout can't proceed (old Pods never scaled down)
- Insufficient cluster resources for `maxSurge` Pods → new Pods `Pending`
- `progressDeadlineSeconds` exceeded → Deployment condition `Progressing=False, Reason=ProgressDeadlineExceeded`

**Fix:** `kubectl rollout undo deployment/myapp` to immediately revert while you investigate.

### Deployment shows desired replicas but fewer Pods running

```bash
kubectl get rs -l app=myapp     # check ReplicaSet(s) — is the wrong one active?
kubectl describe rs <rs-name>   # Events section shows Pod creation failures
```

Common cause: `PodDisruptionBudget` blocking eviction during a concurrent node drain, or a `ResourceQuota` in the namespace blocking new Pod creation.

### Scaling doesn't seem to work

```bash
kubectl get hpa                 # is a HorizontalPodAutoscaler fighting your manual kubectl scale?
```

If an HPA is attached to the Deployment, it will override manual `kubectl scale` changes on its next reconcile.

### Old ReplicaSets piling up / can't rollback far enough

Check `revisionHistoryLimit` — if too low, old ReplicaSets (and their rollback target) get garbage collected.

---

## PART 7 — HANDS-ON MINIKUBE LABS

```bash
minikube start --cpus=4 --memory=4096
```

### Lab 1 — Create, scale, inspect
```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deployments,rs,pods -o wide
kubectl scale deployment/web --replicas=5
kubectl get pods -w
```

### Lab 2 — Rolling update + status + history
```bash
kubectl set image deployment/web nginx=nginx:1.26
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl get rs -l app=web        # notice TWO ReplicaSets, old scaled to 0
```

### Lab 3 — Rollback
```bash
kubectl set image deployment/web nginx=nginx:doesnotexist   # broken rollout
kubectl rollout status deployment/web --timeout=20s          # will time out/hang
kubectl get pods                                              # ImagePullBackOff
kubectl rollout undo deployment/web
kubectl rollout status deployment/web                         # recovers
```

### Lab 4 — Pause/resume batching
```bash
kubectl rollout pause deployment/web
kubectl set image deployment/web nginx=nginx:1.27
kubectl set resources deployment/web -c nginx --limits=cpu=300m,memory=256Mi
kubectl rollout resume deployment/web
kubectl rollout status deployment/web
```

### Lab 5 — Recreate strategy downtime demo
```bash
kubectl patch deployment web -p '{"spec":{"strategy":{"type":"Recreate"}}}'
kubectl set image deployment/web nginx=nginx:1.25
kubectl get pods -w    # observe ALL old Pods terminate before ANY new ones appear
```

### Lab 6 — Force rollout on ConfigMap change
```bash
kubectl create configmap web-config --from-literal=LOG_LEVEL=info
# ... attach via envFrom in your deployment, then:
kubectl create configmap web-config --from-literal=LOG_LEVEL=debug --dry-run=client -o yaml | kubectl apply -f -
kubectl get pods -l app=web -o yaml | grep LOG_LEVEL   # notice: old Pods still show old value!
kubectl rollout restart deployment/web                  # forces new Pods to pick up new ConfigMap
```

---

## PART 8 — CHEAT SHEET, BEST PRACTICES, MISTAKES, INTERVIEW QUESTIONS, SCENARIOS

### Deployment Cheat Sheet

```
Deployment → ReplicaSet → Pods (three-layer hierarchy)
kubectl apply changes spec.template → new ReplicaSet revision → rolling update
kubectl scale changes spec.replicas → NO new revision, same ReplicaSet resized
RollingUpdate: gradual, zero-downtime (tune maxSurge/maxUnavailable)
Recreate: all-down-then-all-up, causes downtime
rollout status   → watch current rollout progress
rollout history  → list revisions
rollout undo     → revert to previous (or --to-revision=N)
pause/resume     → batch multiple template changes into one rollout
ConfigMap/Secret DATA changes do NOT auto-restart Pods — use rollout restart or checksum annotation
revisionHistoryLimit caps how far back you can rollback
progressDeadlineSeconds → rollout marked Failed if stuck too long
```

### kubectl Command Reference

```bash
kubectl create deployment <name> --image=<img> --replicas=N
kubectl get deployments,rs,pods -o wide
kubectl describe deployment <name>
kubectl scale deployment/<name> --replicas=N
kubectl set image deployment/<name> <container>=<image>
kubectl set resources deployment/<name> -c <container> --limits=cpu=500m,memory=512Mi
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name> [--revision=N]
kubectl rollout undo deployment/<name> [--to-revision=N]
kubectl rollout pause/resume deployment/<name>
kubectl rollout restart deployment/<name>
kubectl edit deployment/<name>
kubectl delete deployment/<name>
```

### Production Best Practices

- Always set `maxUnavailable: 0` for user-facing services requiring zero downtime (requires `maxSurge >= 1`)
- Pin exact image tags/digests; never deploy `:latest`
- Define both `readinessProbe` and `livenessProbe`; add `startupProbe` for slow-starting apps
- Set `minReadySeconds` so flaky-but-technically-ready Pods don't get counted too early
- Use `podAntiAffinity` to spread replicas across nodes/zones
- Attach a `PodDisruptionBudget` alongside every production Deployment
- Use `progressDeadlineSeconds` + alerting on `Progressing=False` to catch stuck rollouts automatically
- Force rollouts on ConfigMap/Secret changes via checksum annotations or `rollout restart`
- Keep `revisionHistoryLimit` reasonable (5-10) — balance rollback depth vs etcd bloat
- Always test rollbacks, not just rollouts, in staging

### Common Mistakes

- Forgetting `maxUnavailable: 0` and getting brief downtime during "zero-downtime" deploys
- Assuming ConfigMap edits auto-propagate to running Pods
- Using `kubectl scale` when an HPA is active (HPA will fight/override it)
- Not setting resource requests, causing `maxSurge` Pods to fail scheduling mid-rollout
- Treating `Recreate` and `RollingUpdate` as interchangeable — using `Recreate` unnecessarily and causing avoidable downtime
- Changing `selector` after creation attempts (immutable — apply will be rejected)
- Not annotating change-cause, making `rollout history` useless for auditing
- Ignoring `progressDeadlineSeconds`/stuck rollouts until a full outage happens

### 30 Interview Questions

1. What is a Deployment, and what does it manage directly?
2. Why does a Deployment create a ReplicaSet instead of managing Pods directly?
3. What's immutable about a Deployment's `selector` field, and why?
4. What triggers the creation of a new ReplicaSet revision?
5. Does scaling replicas create a new revision? Why or why not?
6. Explain RollingUpdate vs Recreate strategy trade-offs.
7. What do `maxSurge` and `maxUnavailable` control, and how do they interact?
8. How do you achieve a truly zero-downtime rolling update?
9. What is `minReadySeconds` and why does it matter during rollout?
10. What happens internally when you run `kubectl apply -f deployment.yaml` for the first time?
11. Trace the full object flow from `kubectl apply` to a running container.
12. What is a reconciliation loop, and which controllers are involved in a Deployment rollout?
13. How does `kubectl rollout undo` actually work internally?
14. What's the difference between `rollout undo` and `rollout undo --to-revision=N`?
15. Why doesn't updating a ConfigMap automatically restart Pods using it?
16. How do you force a rollout when only a ConfigMap/Secret changes?
17. What does `revisionHistoryLimit` control, and what's the trade-off of setting it too low?
18. What is `progressDeadlineSeconds`, and what condition does it set on failure?
19. What's the purpose of `kubectl rollout pause`/`resume`?
20. How does an HPA interact with manually running `kubectl scale`?
21. What Kubernetes object type actually creates and deletes Pod objects during a rollout?
22. How does the Deployment controller decide it's safe to terminate an old Pod during RollingUpdate?
23. What happens if a new Pod during rollout never passes its readiness probe?
24. How would you debug a rollout that's stuck in "Progressing"?
25. What's the effect of `podAntiAffinity` on a Deployment's replicas?
26. Why should you avoid using `:latest` as an image tag in Deployments?
27. What object connects a Deployment to its Pods for cascade deletion?
28. How would you safely batch several spec changes into a single rollout?
29. What's the difference in blast radius between a bad image update and a bad ConfigMap update?
30. Describe how you'd design a Deployment for a stateless zero-downtime production web service, end to end.

### Real-World Scenarios

**Scenario 1 — "Our deploy caused a brief 500 spike."**
Diagnosis path: check `maxUnavailable` (likely > 0), check readiness probe correctness (Pod counted ready before actually serving), check if Service `Endpoints` lag behind Pod readiness. Fix: `maxUnavailable: 0`, tighten readiness probe, add `preStop` sleep for connection draining.

**Scenario 2 — "We changed a ConfigMap and nothing happened."**
Diagnosis: ConfigMap changes don't trigger rollouts by design. Fix: add a checksum annotation of the ConfigMap content into the Pod template, or use `kubectl rollout restart` as an operational step tied to config changes.

**Scenario 3 — "Rollout hangs forever in CI/CD."**
Diagnosis: new Pods stuck `Pending` (no room for `maxSurge`) or failing readiness. Fix: add resource headroom, set `progressDeadlineSeconds` with alerting, and have CI treat rollout timeout as failure + auto `rollout undo`.

**Scenario 4 — "We need instant rollback capability during an incident."**
Ensure `revisionHistoryLimit` is high enough, change-causes are always annotated, and `kubectl rollout undo` is a rehearsed runbook step (test it in staging regularly, not just in theory).

---

This completes the Deployments chapter/batch. Ready for the next batch (Services, ReplicaSets in depth, ConfigMaps/Secrets, StatefulSets, or wherever the approved 8-batch plan goes next) whenever you want to continue.
