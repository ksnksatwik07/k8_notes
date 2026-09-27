# Kubernetes Architecture — Complete Masterclass
## Part 4 of 6: kubectl Internals, the Full Request Flow & Pod/Scheduling Lifecycle

*(Continuing from Parts 1–3. This part covers topics 25–29, plus sequence diagrams A–C: Creating a Deployment, Creating a Pod, and Scheduling a Pod. Diagrams D–F, plus failure scenarios, HA, security, and the full interview/cheat-sheet finale, arrive in Parts 5–6.)*

---

# 25. WHAT HAPPENS INTERNALLY: kubectl create / apply / get / delete

## `kubectl create -f pod.yaml`

**Purely imperative** — sends a single `POST` request. If an object with that name already exists, it **fails outright** with an "AlreadyExists" error; there's no merging, no reconciling with what's already there.

```bash
kubectl create -f pod.yaml
# HTTP: POST /api/v1/namespaces/default/pods
```

## `kubectl apply -f pod.yaml`

**Declarative**, and considerably more involved internally. `apply` doesn't just PUT the new object — it computes a **three-way diff/merge** every time:

```
              ┌───────────────────────────────────────────┐
              │  1. LAST APPLIED CONFIG                     │
              │     (stored as an annotation on the object,  │
              │     kubectl.kubernetes.io/last-applied-       │
              │     configuration — a snapshot of what YOU   │
              │     applied last time)                        │
              └───────────────────────────────────────────┘
              ┌───────────────────────────────────────────┐
              │  2. LIVE OBJECT STATE                        │
              │     (current state in etcd RIGHT NOW —       │
              │     may have been changed by controllers      │
              │     or other users since your last apply)     │
              └───────────────────────────────────────────┘
              ┌───────────────────────────────────────────┐
              │  3. NEW CONFIG YOU'RE APPLYING NOW           │
              │     (the file you just ran apply against)     │
              └───────────────────────────────────────────┘
                            │
                            ▼
              Three-way merge computes a PATCH:
              - Fields changed between (1) and (3) → update
              - Fields present in (1) but REMOVED in (3) → delete
              - Fields in (2) that were never in (1) or (3)
                (e.g., set by a controller/webhook) → left alone
```
This is precisely why `apply` can safely be run repeatedly (idempotent) and why it correctly **removes fields you deleted from your YAML**, while never clobbering fields a controller manages that were never part of your file (like `.status`, or fields injected by a mutating webhook).

```bash
kubectl apply -f pod.yaml
# HTTP: PATCH (strategic merge patch, or server-side apply)
#       /api/v1/namespaces/default/pods/my-pod
```

**Server-Side Apply** (the modern default in recent Kubernetes) improves on this further by having the **API server itself** track field ownership per-manager, resolving conflicts between multiple applying tools (e.g., a human via `kubectl` and a CI/CD controller both managing the same object) far more robustly than the old client-side three-way diff.

## `kubectl get pods`

A straightforward `GET` — but worth understanding exactly what's returned and from where:

```bash
kubectl get pods
# HTTP: GET /api/v1/namespaces/default/pods
```
This reads **directly from etcd** (via the API server) — not a live poll of the actual nodes. What you see is the **last status the kubelet reported**, not a real-time query of the container's live state at that exact instant. This is why, very occasionally, `kubectl get pods` can lag reality by a second or two (the kubelet's own status-reporting interval) — a subtlety worth knowing when debugging flapping Pods.

```bash
kubectl get pods -w      # adds ?watch=true — a long-lived connection, streaming live updates
```

## `kubectl delete pod my-pod`

```bash
kubectl delete pod my-pod
# HTTP: DELETE /api/v1/namespaces/default/pods/my-pod
```
Deletion is **not instantaneous** by default — it follows a **graceful termination** sequence:
```
1. API server sets .metadata.deletionTimestamp (object is NOT removed yet —
   it's marked for deletion; this is what a "finalizer" can react to before
   actual removal)
2. kubelet (watching, as always) notices the deletionTimestamp on a Pod
   assigned to its node
3. kubelet sends SIGTERM to the container's main process
4. kubelet waits up to .spec.terminationGracePeriodSeconds (default: 30s)
   for the process to exit cleanly
5. If it hasn't exited by then, kubelet sends SIGKILL — forceful, immediate
6. Once the container is confirmed stopped, the kubelet reports this back,
   and (once any finalizers are cleared) the API server finally removes the
   object from etcd entirely
```
**Crucial and commonly missed:** if the Pod is managed by a ReplicaSet/Deployment, deleting it triggers the ReplicaSet controller's reconciliation loop (per Part 3, §21) to notice "actual replicas < desired replicas" and create a brand-new replacement Pod almost immediately — deleting one Pod under a Deployment doesn't reduce your replica count; it just churns that one instance.

```bash
kubectl delete pod my-pod --grace-period=0 --force   # skips graceful termination — use with caution
```

---

# 26. THE COMPLETE FLOW: kubectl → apiserver → auth/authz → admission → etcd → controllers/scheduler → kubelet → runtime

This is the single most important diagram in the entire masterclass — every other diagram in this document is a specific instance of this general shape.

```
┌──────────┐
│  kubectl  │  1. You run: kubectl apply -f deployment.yaml
└────┬─────┘
     │ HTTPS request (client cert or token from your kubeconfig)
     ▼
┌─────────────────────────────────────────────────────────────┐
│                      kube-apiserver                            │
│                                                                  │
│  2. AUTHENTICATION — "who are you?"                              │
│     validates your client cert/token → identity: your user        │
│                        │                                          │
│                        ▼                                          │
│  3. AUTHORIZATION — "can you POST/PATCH Deployments here?"        │
│     RBAC check against your Roles/ClusterRoles                    │
│                        │  (allowed)                                │
│                        ▼                                          │
│  4. ADMISSION CONTROL — mutating, then validating webhooks/        │
│     built-in plugins (defaults injected, ResourceQuota checked,   │
│     etc. — per Part 3 §24)                                        │
│                        │  (passes)                                 │
│                        ▼                                          │
│  5. WRITE TO ETCD — the Deployment object is persisted             │
└────────────────────────┬────────────────────────────────────────┘
                          │  6. every watcher is notified immediately
        ┌──────────────────┼──────────────────┐
        ▼                                       ▼
┌───────────────────┐                  (nothing else needs to react
│ kube-controller-    │                   to a Deployment directly except
│ manager             │                   the Deployment controller)
│                      │
│ 7. Deployment        │
│    controller sees    │
│    the new Deployment │
│    → creates a        │
│    ReplicaSet object  │
│    (another WRITE      │
│    back through the    │
│    API server, step 5  │
│    repeats)             │
└──────────┬───────────┘
           │
           ▼
┌───────────────────┐
│ ReplicaSet          │
│ controller sees the  │
│ new ReplicaSet        │
│ → creates N Pod       │
│   objects (another    │
│   WRITE, step 5        │
│   repeats again)       │
│   — each Pod has NO    │
│   nodeName yet          │
└──────────┬───────────┘
           │
           ▼
┌───────────────────┐
│  kube-scheduler      │
│                      │
│ 8. sees unscheduled  │
│    Pods (empty        │
│    nodeName) →         │
│    filters + scores    │
│    nodes → WRITES a    │
│    Binding (sets       │
│    nodeName) — step 5  │
│    repeats yet again    │
└──────────┬───────────┘
           │
           ▼
┌───────────────────┐
│      kubelet         │  (on the specific node that was chosen)
│  (worker node)        │
│                      │
│ 9. sees a Pod now      │
│    assigned to ITS     │
│    node → calls the    │
│    container runtime   │
│    via CRI              │
└──────────┬───────────┘
           │ gRPC (CRI)
           ▼
┌───────────────────┐
│  Container Runtime   │
│  (containerd/CRI-O)   │
│                      │
│ 10. pulls the image,  │
│     creates namespaces │
│     /cgroups, starts    │
│     the container       │
│     process              │
└──────────┬───────────┘
           │
           ▼
     Container is now RUNNING.
     kubelet reports status back through the API server (step 5
     repeats one final time — Pod .status.phase = Running),
     which is what makes `kubectl get pods` finally show "Running."
```

**Notice the pattern repeats "step 5" (write to etcd through the API server) five separate times across this flow** — every single arrow that crosses a component boundary in this diagram is, underneath, another instance of the exact same authenticate → authorize → admit → persist → notify-watchers cycle. There is genuinely only one mechanism in this entire system; everything else is a specific controller reacting to a specific object type.

---

# 27. POD CREATION LIFECYCLE FROM YAML TO RUNNING CONTAINER

## Sequence Diagram A: Creating a Deployment (end-to-end)

```
User          apiserver         etcd        Deploy-Ctrl    RS-Ctrl      Scheduler      kubelet        Runtime
 │                │                │              │             │             │             │              │
 │─apply YAML────▶│                │              │             │             │             │              │
 │                │─auth/admit────▶│              │             │             │             │              │
 │                │─write Deploy──▶│              │             │             │             │              │
 │◀──201 Created──│                │              │             │             │             │              │
 │                │                │─notify──────▶│             │             │             │              │
 │                │                │              │─create RS──▶│             │             │              │
 │                │◀───────────────┼──────────────│             │             │             │              │
 │                │─write RS──────▶│              │             │             │             │              │
 │                │                │─notify───────┼────────────▶│             │             │              │
 │                │                │              │             │─create Pods▶│             │              │
 │                │◀───────────────┼──────────────┼─────────────│             │             │              │
 │                │─write Pods────▶│ (nodeName empty)           │             │             │              │
 │                │                │─notify───────┼─────────────┼────────────▶│             │              │
 │                │                │              │             │             │─filter+score│              │
 │                │                │              │             │             │─bind Pod───▶│              │
 │                │◀───────────────┼──────────────┼─────────────┼─────────────│             │              │
 │                │─write binding─▶│ (nodeName SET now)         │             │             │              │
 │                │                │─notify───────┼─────────────┼─────────────┼────────────▶│              │
 │                │                │              │             │             │             │─pull+run───▶│
 │                │                │              │             │             │             │◀─running────│
 │                │◀───────────────┼──────────────┼─────────────┼─────────────┼─────status──│              │
 │                │─write status──▶│              │             │             │             │              │
 │─kubectl get────▶│               │              │             │             │             │              │
 │◀──"Running"────│                │              │             │             │             │              │
```

## Step-by-step explanation

1. **User applies YAML** — `kubectl apply -f deployment.yaml`
2. **API server authenticates/authorizes/admits, then persists** the Deployment object to etcd
3. **Deployment controller** (watching Deployments) notices the new object, creates a matching **ReplicaSet** — written back through the API server
4. **ReplicaSet controller** (watching ReplicaSets) notices the new ReplicaSet, creates the declared number of **Pod objects** — each with an empty `.spec.nodeName` (unscheduled)
5. **kube-scheduler** (watching for exactly this — Pods with no `nodeName`) filters and scores nodes, then writes a **Binding**, setting `.spec.nodeName`
6. **kubelet** on the chosen node (watching for Pods assigned to itself specifically) notices the newly-bound Pod
7. **kubelet calls the container runtime** via CRI to pull the image and start the container
8. **kubelet reports status back** through the API server — this final status write is what makes `kubectl get pods` show `Running`

## Beginner takeaway
Four completely separate "creates" happened (Deployment → ReplicaSet → Pods → Binding) before a single container actually started — and every one of them was written by an independent controller reacting to the previous write, exactly as described in Part 3.

---

# 28. SCHEDULING LIFECYCLE

## Sequence Diagram C: Scheduling a Pod (zoomed in on step 5 above)

```
Scheduler                    apiserver                    etcd
    │                             │                          │
    │─watch (unscheduled Pods)───▶│                          │
    │                             │─watch stream─────────────▶│
    │◀────── Pod event: ADDED ────┼──────────────────────────│
    │  (nodeName empty)           │                          │
    │                             │                          │
    │─GET /nodes (via Informer's local cache, not a live call)
    │                             │                          │
    │─FILTER phase:                                            │
    │   for each node, check:                                  │
    │     - enough allocatable CPU/memory? (Resource Requests   │
    │       guide, Part 4)                                      │
    │     - taints tolerated?                                   │
    │     - node selector / affinity satisfied?                 │
    │     - port conflicts?                                     │
    │   → produces a shortlist of FEASIBLE nodes                │
    │                             │                          │
    │─SCORE phase:                                             │
    │   for each feasible node, run scoring plugins:            │
    │     - least-requested (prefer emptier nodes)               │
    │     - balanced resource allocation                         │
    │     - inter-pod affinity/anti-affinity preferences          │
    │     - topology spread constraints                           │
    │   → pick the highest total score                            │
    │                             │                          │
    │─POST Binding (Pod → chosen node)──────────────────────▶│
    │                             │─write────────────────────▶│
    │◀────────────── 201 Created ─┤                          │
```

## What a "Binding" actually is
A `Binding` is a tiny, special sub-resource — essentially just `{podName, nodeName}` — that, once written, sets the Pod's `.spec.nodeName` field. **This is the entire scheduling decision, in full.** The scheduler's job is complete the instant this write succeeds; it has no further involvement in that Pod's life (barring re-scheduling if the Pod is deleted and recreated later).

## What actually happens internally when scheduling fails
If no node passes the filtering phase, the Pod remains `Pending`, and the scheduler records **why**, per node, as an Event:
```bash
kubectl describe pod <pod>
```
```
Events:
  Warning  FailedScheduling  0/5 nodes are available:
           2 Insufficient memory, 3 node(s) had taint {special: true},
           that the pod didn't tolerate.
```
The scheduler re-attempts on a backoff/retry basis and whenever cluster state relevant to this Pod changes (a node is added, another Pod is deleted freeing capacity) — it isn't a one-shot attempt that simply gives up.

---

# 29. CONTROLLER RECONCILIATION — WALKED THROUGH CONCRETELY

## Scenario: a Pod under a Deployment crashes

```
t=0s   Pod "my-app-7d4f-xk2p9" (one of 3 desired) crashes (OOMKilled)
t=0s   kubelet detects the container exited, reports Pod phase → Failed
       (written through the API server, per the universal mechanism)
t=0.1s ReplicaSet controller's Informer cache is updated by the watch event
t=0.1s ReplicaSet controller's reconcile loop runs:
         desired: 3 (from .spec.replicas)
         actual:  2 (counts Pods matching its label selector that are
                     NOT terminating/failed)
         gap: 1 → CREATE one new Pod object
t=0.15s New Pod object written (nodeName empty)
t=0.2s  Scheduler's watch fires, filters/scores, writes a Binding
t=0.3s  Target node's kubelet notices, calls the container runtime
t=1-5s  Image pull (if not cached) + container start
t=5s    kubelet reports Running; ReplicaSet controller's next reconcile
        confirms actual == desired again → loop goes quiet until the
        next discrepancy
```

**Total observable "self-healing" time here is typically single-digit seconds**, achieved with zero human involvement and zero special-case code — this is just the same generic reconciliation loop from Part 3, running exactly as it always does, reacting to a gap that happened to be caused by a crash instead of, say, someone editing `.spec.replicas` upward.

## Fun fact
This is precisely why scaling a Deployment (`kubectl scale deployment my-app --replicas=5`) and recovering from a crashed Pod are, internally, **the literal same code path** in the ReplicaSet controller — both are just "actual count differs from desired count," and the controller has no idea (and doesn't need to know) which of the two situations it's currently handling.

---

## What's coming in Part 5

Part 5 covers the remaining sequence diagrams (**D: Pod restart, E: Node failure, F: Deployment update/rollout**), etcd data storage/consistency in depth, Kubernetes networking between components, and the full **failure scenario analysis** (API server down, etcd unavailable, scheduler unavailable, controller-manager unavailable, kubelet unavailable, node failure) with concrete timelines for each.

Say **"continue"** whenever you're ready.
