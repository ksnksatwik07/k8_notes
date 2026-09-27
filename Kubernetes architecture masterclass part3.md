# Kubernetes Architecture — Complete Masterclass
## Part 3 of 5+: Controllers, the Declarative Model & How Everything Actually Communicates

*(Continuing from Parts 1–2. This part covers topics 15–24 — the mechanical "how" behind everything you've learned conceptually so far. This is the part that makes "no component talks directly to another" click.)*

---

# 15. CONTROLLERS

## Beginner explanation
A controller is a small, focused program with exactly one job: **watch some type of object, and continuously nudge reality toward what that object says it should be.** Kubernetes has dozens of these running at once, each responsible for a narrow slice of behavior.

## Intermediate explanation
Every controller — whether built into `kube-controller-manager` or a custom Operator you write yourself — follows the identical shape, called the **control loop**:

```
      ┌──────────────────────────────────────────┐
      │                                            │
      ▼                                            │
  1. OBSERVE                                        │
  (watch the API server for objects I care about)   │
      │                                            │
      ▼                                            │
  2. COMPARE                                        │
  (desired state, from spec, vs actual state,       │
   from status)                                     │
      │                                            │
      ▼                                            │
  3. ACT                                            │
  (create/update/delete objects to close the gap)   │
      │                                            │
      └──────────────────────────────────────────┘
              repeat, forever, for the life of the cluster
```

## Advanced internals
Controllers **never** call each other's code directly. A Deployment controller doesn't know the ReplicaSet controller exists, doesn't call any function belonging to it — it simply **creates a ReplicaSet object** through the API server, and the ReplicaSet controller *independently* notices that new object (because it's watching for ReplicaSets) and reacts to *it*. This is a layered, cascading reaction, not a direct call chain:

```
Deployment controller:  sees Deployment → creates/updates a ReplicaSet object
                                              │
                                              ▼ (ReplicaSet controller is
                                                 independently watching for
                                                 exactly this object type)
ReplicaSet controller:  sees ReplicaSet → creates/deletes Pod objects
                                              │
                                              ▼ (kubelet is independently
                                                 watching for Pods assigned
                                                 to ITS node)
kubelet:                sees Pod assigned to its node → starts the container
```
Three completely independent watchers, three completely independent reactions, chained only by each one observing what the previous one wrote to etcd. Nobody called anybody. This layered-reaction pattern is *the* defining architectural idea of Kubernetes, and it's why the system scales to so many controllers without becoming an unmanageable web of direct dependencies.

## Fun fact
This is exactly why writing your own Kubernetes "Operator" is not learning some separate, exotic skill — an Operator is just another controller, written by you, watching your own Custom Resource Definition (CRD) instead of a built-in type, running the exact same observe-compare-act loop as the Deployment controller.

---

# 16. PODS AND WORKLOADS

## What a Pod actually is
A Pod is the **smallest deployable unit** in Kubernetes — one or more containers that are always scheduled together, onto the same node, sharing the same network namespace (same IP, same port space) and optionally the same storage volumes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
```
**Why "one or more" containers, not always exactly one?** Multi-container Pods exist for tightly-coupled helper patterns — a **sidecar** (e.g., a log shipper reading the main container's logs), an **init container** (runs to completion before the main containers start, e.g., a DB migration), or an **ambassador/adapter** — cases where two processes genuinely need to share network and lifecycle, not just be "related."

## Why you almost never create bare Pods directly
A bare Pod has **no self-healing** — if the node it's on dies, or the container crashes past what the kubelet can restart locally, nothing recreates it. This is why real workloads are managed through **workload controllers** layered on top of Pods:

| Workload object | Adds on top of a bare Pod |
|---|---|
| **Deployment** | Replica count, rolling updates, rollback history — for stateless apps |
| **StatefulSet** | Stable per-Pod identity/DNS/storage — for stateful apps (databases, etc.) |
| **DaemonSet** | Exactly one Pod per (matching) node — for node-level agents (kube-proxy, log collectors) |
| **Job** | Run-to-completion semantics, retries — for batch work |
| **CronJob** | Scheduled, recurring Jobs |

Each of these is, itself, just another controller (per §15) that ultimately creates and manages ordinary Pod objects underneath — a Deployment doesn't run anything itself; it creates a ReplicaSet, which creates Pods.

```
Deployment ──creates──▶ ReplicaSet ──creates──▶ Pod ──creates──▶ Pod ──creates──▶ Pod
   (you edit this)         (managed for you)      (the actual units the kubelet runs)
```

## Beginner misunderstanding worth flagging here
People often think "editing a Deployment's image updates the Pods." It doesn't, directly — editing a Deployment creates a **brand-new ReplicaSet** with the new Pod template, scales it up, and scales the old ReplicaSet down to zero. The "old" Pods are never modified in place; they're replaced entirely. This matters for understanding rollbacks (Part 4 covers this with a full sequence diagram).

---

# 17. HOW ALL COMPONENTS COMMUNICATE

## The single rule, restated precisely
**Every component-to-component interaction in Kubernetes is: read from, or write to, an object via the kube-apiserver's REST API — nothing more, nothing less.** There is no RPC framework, no message queue, no direct socket between a controller and a kubelet. It's all just HTTP against a shared object store, mediated by one gatekeeper.

```
              ┌───────────────────────────────┐
              │        kube-apiserver           │
              │   (the only thing that reads/   │
              │    writes etcd; everyone else    │
              │    reads/writes THROUGH it)      │
              └───────┬───────────────┬────────┘
       watches/writes │               │ watches/writes
           ┌──────────┘               └──────────┐
           ▼                                       ▼
  ┌─────────────────┐                    ┌─────────────────┐
  │  kube-scheduler   │                    │  kube-controller- │
  │                    │                    │  manager           │
  └─────────────────┘                    └─────────────────┘
           ▲                                       ▲
           │  (all four of these watch/write        │
           │   independently — none call each        │
           │   other directly)                        │
  ┌────────┴────────┐                    ┌───────────┴──────┐
  │     kubelet       │                    │   kube-proxy       │
  │  (on every node)  │                    │  (on every node)   │
  └───────────────────┘                    └────────────────────┘
```

## Why this design, specifically
1. **Decoupling** — any component can be restarted, upgraded, or replaced independently; nobody holds a direct reference or open connection to anybody else's internals
2. **Security** — a single, consistent enforcement point for authentication/authorization/admission (Part 3, §24) rather than every component needing its own security logic
3. **Extensibility** — a brand-new controller (yours, or a vendor's) can be added simply by having it watch/write objects through the same API — zero changes required to any existing component
4. **Auditability** — every state change in the entire cluster passes through one place, which is exactly where Kubernetes' audit logging hooks in

---

# 18. THE KUBERNETES API

## What it is
A **RESTful, resource-oriented API** — everything in Kubernetes (Pods, Services, Deployments, Nodes, even cluster-wide settings) is represented as a resource with a standard URL shape:

```
/api/v1/namespaces/{namespace}/pods/{name}                        # core, namespaced
/apis/apps/v1/namespaces/{namespace}/deployments/{name}            # named API group, namespaced
/apis/rbac.authorization.k8s.io/v1/clusterroles/{name}              # named group, cluster-scoped
```

## API groups and versioning
- `/api/v1` — the original "core" group (Pods, Services, ConfigMaps, Secrets, Nodes)
- `/apis/<group>/<version>` — every newer resource type lives under a named group (`apps`, `batch`, `networking.k8s.io`, `rbac.authorization.k8s.io`, etc.)
- Versions carry an explicit maturity signal: `v1alpha1` (experimental, may change/vanish) → `v1beta1` (more stable, still evolving) → `v1` (stable, backward-compatibility guaranteed)

```bash
kubectl api-resources                 # list every resource type this cluster knows about
kubectl api-versions                  # list every API group/version available
kubectl explain deployment.spec        # inline schema documentation, straight from the API
```

## Extensibility: Custom Resource Definitions (CRDs)
Kubernetes lets you **define entirely new object types** that behave exactly like built-in ones — same REST conventions, same `kubectl get/describe/apply`, same watch support — via a `CustomResourceDefinition`. This is the foundation of the entire Operator ecosystem (cert-manager's `Certificate`, the External Secrets Operator's `ExternalSecret` from the Secrets guide, Cilium's `CiliumNetworkPolicy`, and thousands more) — none of these required changes to Kubernetes core; they're all just CRDs plus a controller watching them.

---

# 19. THE DECLARATIVE MODEL

## Declarative vs imperative — the core distinction

| | Imperative | Declarative |
|---|---|---|
| You specify | The exact steps to take | The end state you want |
| Example | "Start container X, then Y, then set up networking between them" | "I want 3 replicas of this Pod template, running, always" |
| Who figures out *how* | You | Kubernetes' controllers |
| Kubernetes command style | `kubectl create` (closer to imperative — see §25) | `kubectl apply` (fully declarative) |

## Why Kubernetes is built declarative-first
Imperative systems require you to account for every possible current state before acting ("is it already running? partially running? crashed?"). Declarative systems sidestep this entirely — you always describe the *end goal*, and the same reconciliation loop handles every starting condition uniformly, including ones nobody anticipated when the manifest was written (a node dying, a Pod being manually deleted, a config drifting) — the controller doesn't need special-case logic for each; it just keeps reconciling toward the same declared goal regardless of how reality got knocked off course.

---

# 20. DESIRED STATE VS CURRENT STATE

Every Kubernetes object has exactly this shape:
```yaml
spec:      # DESIRED state — what YOU want; only a human (or CI/CD) writes this
  replicas: 3
status:    # CURRENT (actual, observed) state — only CONTROLLERS write this
  replicas: 2
  readyReplicas: 2
```

**The entire job of every controller is closing the gap between `.spec` and `.status`.** This split is enforced structurally, not just by convention — Kubernetes' RBAC model commonly grants regular users write access to `.spec` while restricting `.status` writes to controllers via a separate "status subresource," specifically to prevent a user from ever lying about the cluster's actual observed state.

```bash
kubectl get deployment my-app -o jsonpath='{.spec.replicas}'    # desired
kubectl get deployment my-app -o jsonpath='{.status.replicas}'  # actual
```

---

# 21. RECONCILIATION LOOPS

## The loop, precisely
```
for {
    desired := getDesiredStateFromSpec()
    actual  := getActualStateFromCluster()
    if desired != actual {
        takeActionToConverge(desired, actual)
    }
    sleepOrWaitForNextEvent()
}
```
This runs **continuously, forever**, for the entire lifetime of the cluster — it's not a one-shot script that runs once at creation and stops. This is why deleting a Pod that's managed by a ReplicaSet gets it instantly recreated: the reconciliation loop notices the gap (desired: 3, actual: 2) within moments and closes it — with no human, and no explicit "recreate" command, ever involved.

## Level-based, not edge-based
A critical, subtle design choice: reconciliation is **level-triggered, not edge-triggered** — it reacts to the *current* gap between desired and actual state, not to the *specific event* that caused it. Whether a Pod died from an OOM kill, a manual `kubectl delete`, or a node crash, the ReplicaSet controller's response is identical: "actual is less than desired, create one more." It doesn't need — and deliberately doesn't have — different logic for each possible cause. This makes controllers dramatically simpler and more robust than if they had to correctly handle every possible event history.

---

# 22. WATCHES

## The mechanism
Rather than every controller **polling** "has anything changed?" every few seconds (wasteful, slow to react, and murderous on the API server at scale), Kubernetes uses **watches** — a long-lived HTTP connection where the API server *pushes* change notifications the instant they happen.

```
Controller: "GET /api/v1/pods?watch=true"
              (opens one long-lived HTTP connection, held open indefinitely)

API server: [Pod created]  → pushes: {"type": "ADDED",   "object": {...}}
            [Pod updated]  → pushes: {"type": "MODIFIED","object": {...}}
            [Pod deleted]  → pushes: {"type": "DELETED",  "object": {...}}
```
Each event arrives within milliseconds of the change actually happening in etcd — this is *why* Kubernetes feels near-instantaneous despite being a fundamentally distributed, decoupled system: nobody is waiting out a polling interval.

## Resilience: resourceVersion
Every object carries a `resourceVersion`, an opaque, monotonically-changing marker. If a watch connection drops (network blip, API server restart), the controller reconnects and says "resume from `resourceVersion: X`" — the API server (backed by etcd's own change history) can replay everything that happened while the connection was down, so **no event is silently missed**, even across a real disconnection.

---

# 23. INFORMERS AND CONTROLLER BEHAVIOR

## Why raw watches aren't used directly
Handling a raw watch stream naively in every controller (reconnect logic, resync, local caching, deduplication) would mean every single controller reimplementing the same complex, error-prone plumbing. Kubernetes' client libraries solve this once, generically, with a component called an **Informer**.

## What an Informer actually does
```
             ┌─────────────────────────────────────────┐
             │              Informer                     │
             │                                             │
   watch ───▶│  1. Lists all objects once at startup      │
  events     │  2. Populates a local in-memory cache        │──▶ Controller's
             │     (a "thread-safe store")                   │    reconcile
             │  3. Keeps that cache updated via the watch     │    function
             │     stream, forever                             │    (reads
             │  4. Fires callbacks (Add/Update/Delete) into   │     from the
             │     a work queue for the controller to process │     LOCAL
             │  5. Automatically handles reconnects/resync    │     cache,
             │                                                 │     never
             └─────────────────────────────────────────┘     hits the API
                                                                 server per-read)
```

**The key performance insight:** a controller's actual reconcile logic reads from the **Informer's local cache**, not by making a live API call every time it needs to check something. This is why Kubernetes can run *hundreds* of controllers, each reconciling constantly, without collapsing the API server under read load — writes go through the API server, but the overwhelming majority of reads are served from memory, locally, per-controller.

## Fun fact
If you write a custom controller using `client-go` (Kubernetes' official Go client) or the popular `controller-runtime` library (which underlies the Operator SDK and Kubebuilder), you get this exact Informer machinery for free — you write only the "compare desired vs actual, then act" logic; the watch/cache/resync plumbing is entirely handled by the shared library, identical to what every built-in controller uses.

---

# 24. API REQUESTS: AUTHENTICATION, AUTHORIZATION, ADMISSION

Every single request hitting the API server — whether from `kubectl`, a controller, or a kubelet — passes through the same three-stage gauntlet, in this exact order, before anything is written to etcd:

```
Request arrives
      │
      ▼
┌─────────────────────┐
│ 1. AUTHENTICATION     │  "Who are you?"
│  (AuthN)              │  client certs, bearer tokens, OIDC, service account tokens
└──────────┬───────────┘
           ▼  (attaches an identity: user/group, or system:serviceaccount:...)
┌─────────────────────┐
│ 2. AUTHORIZATION      │  "Are you ALLOWED to do this specific thing?"
│  (AuthZ)              │  RBAC (Roles/ClusterRoles + Bindings) is the dominant mode
└──────────┬───────────┘
           ▼  (only if allowed)
┌─────────────────────┐
│ 3. ADMISSION CONTROL   │  "Should this SPECIFIC request be permitted or modified?"
│  - Mutating webhooks    │  (e.g., inject a sidecar, apply defaults)
│  - Validating webhooks  │  (e.g., reject if a required label is missing)
│  - Built-in controllers │  (ResourceQuota, LimitRange, NamespaceLifecycle, etc.)
└──────────┬───────────┘
           ▼  (only if it passes ALL admission checks)
      ┌──────────┐
      │   etcd    │  ← finally, and only now, persisted
      └──────────┘
```

## Why admission is a separate stage from authorization
Authorization is a **yes/no** decision about the *caller's* general permissions ("can this user create Pods in this namespace at all?"). Admission is about the **specific content** of *this* request, and can even **mutate** it — this is exactly the mechanism behind the LimitRange default-injection from the Resource Limits guide (a `MutatingAdmissionWebhook`/built-in admission plugin silently adds `resources.requests` to a Pod spec that omitted them) and ResourceQuota enforcement (a `ValidatingAdmissionWebhook`/built-in plugin rejects a Pod that would push the namespace over its cap).

## Real-world example tying this together
When `cert-manager` automatically issues certificates (from the Ingress guide), it commonly relies on a **mutating webhook** to inject default fields, and its own controller (watching `Certificate` CRDs, per §15/§18) to actually do the ACME work — the entire feature is built from nothing but standard Kubernetes extension points: a CRD, a controller, and an admission webhook, with zero core-Kubernetes code changes required.

## Troubleshooting authentication/authorization
```bash
kubectl auth can-i create pods --namespace=production --as=system:serviceaccount:default:my-sa
kubectl auth can-i --list                       # everything the current identity can do
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

---

## What's coming in Part 4

Part 4 covers topics 25–29 with **full sequence diagrams**: exactly what happens internally for `kubectl create`, `kubectl apply`, `kubectl get`, and `kubectl delete`; the complete `kubectl → apiserver → auth → admission → etcd → controllers/scheduler → kubelet → runtime` flow end-to-end; and the full Pod-creation-to-running-container lifecycle, plus the scheduling lifecycle and controller reconciliation walked through concretely with a real Deployment.

Say **"continue"** whenever you're ready.
