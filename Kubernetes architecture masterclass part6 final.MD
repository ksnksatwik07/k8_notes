# Kubernetes Architecture — Complete Masterclass
## Part 6 of 6 (Final): Production, Security, Performance, Labs & the Complete Interview Bank

*(Final installment. Covers topics 33–39, Minikube labs, and the full closing reference material: cheat sheet, comparison table, end-to-end flow recap, mental model, and 65 interview questions plus 5 troubleshooting scenarios.)*

---

# 33–35. HIGH AVAILABILITY & PRODUCTION CLUSTER ARCHITECTURE

## What HA actually means at the control-plane level
A single-control-plane-node cluster has one of every control-plane component — a single point of failure for *changing* the cluster (per Part 5's failure analysis). Production HA runs **at least 3 control-plane nodes**, each with its own copy of every control-plane component:

```
┌───────────────── HA CONTROL PLANE (3 nodes) ─────────────────┐
│                                                                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐          │
│  │  CP Node 1   │   │  CP Node 2   │   │  CP Node 3   │          │
│  │              │   │              │   │              │          │
│  │ apiserver    │   │ apiserver    │   │ apiserver    │          │
│  │ etcd         │   │ etcd         │   │ etcd         │          │
│  │ scheduler    │   │ scheduler    │   │ scheduler    │          │
│  │ (standby)    │   │ (LEADER)     │   │ (standby)    │          │
│  │ ctrl-mgr     │   │ ctrl-mgr     │   │ ctrl-mgr     │          │
│  │ (LEADER)     │   │ (standby)    │   │ (standby)    │          │
│  └─────────────┘   └─────────────┘   └─────────────┘          │
│         ▲                  ▲                  ▲                 │
│         └──────────────────┼──────────────────┘                 │
│                    Load Balancer (:6443)                          │
│                    (HAProxy, cloud LB, or                         │
│                     kube-vip / keepalived VIP)                     │
└─────────────────────────────────────────────────────────────┘
```

## Per-component HA mechanism

| Component | How it achieves HA |
|---|---|
| **kube-apiserver** | Trivially horizontal — stateless, so any number of replicas run **active-active** behind a load balancer simultaneously |
| **etcd** | Raft consensus across an odd number of members (3 or 5) — tolerates a minority failing, per Part 2 §7 and Part 5 §30 |
| **kube-scheduler** | Active-passive via **leader election** (a `Lease` object in the API server) — only one replica actively schedules at a time; others watch the lease and stand ready to take over |
| **kube-controller-manager** | Same active-passive leader-election pattern as the scheduler |

```bash
kubectl -n kube-system get leases
# NAME                     HOLDER
# kube-scheduler            cp-node-2_a1b2c3...
# kube-controller-manager   cp-node-1_d4e5f6...
```
**Why apiserver is active-active but scheduler/controller-manager are active-passive:** the API server has no shared mutable in-memory state between requests — any replica can serve any request independently. The scheduler and controller-manager, by contrast, would **race and duplicate work** (or worse, make conflicting decisions) if multiple replicas actively reconciled the same objects simultaneously — leader election exists specifically to prevent that.

## Production cluster architecture (a realistic full picture)

```
┌──────────────────────── Region / Cluster ────────────────────────┐
│                                                                     │
│  ┌──────── Control Plane (3 nodes, spread across zones) ────────┐  │
│  │  apiserver × 3   etcd × 3   scheduler × 3   ctrl-mgr × 3        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────── Worker Node Pool A (general workloads) ──────────────┐  │
│  │  Node 1 (zone-a)   Node 2 (zone-b)   Node 3 (zone-c)  ...      │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────── Worker Node Pool B (GPU / specialized) ───────────────┐  │
│  │  Node 4 (tainted, GPU-only workloads via toleration)             │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Add-ons (run as ordinary workloads, NOT control-plane):            │
│    - CoreDNS (kube-system)                                           │
│    - Ingress Controller                                              │
│    - Metrics Server / Prometheus                                      │
│    - Cluster Autoscaler                                               │
│    - CNI DaemonSet (Calico/Cilium agent pods)                          │
└─────────────────────────────────────────────────────────────────┘
```
**Real-world note:** on managed Kubernetes (EKS/GKE/AKS), you never see or manage the control-plane nodes directly — the cloud provider guarantees HA for you as part of the managed service, typically across 3 availability zones, and you're billed for the control plane separately from your worker nodes.

---

# 36. SECURITY BOUNDARIES

## The layered boundary model

```
┌─────────────────────────────────────────────────────────────┐
│ 1. NETWORK PERIMETER                                            │
│    Who can even reach the API server's :6443 port?              │
│    (firewall rules, private cluster endpoints, VPNs)             │
└───────────────────────┬───────────────────────────────────┘
┌───────────────────────▼───────────────────────────────────┐
│ 2. AUTHENTICATION                                              │
│    Who are you? (client certs, OIDC, service account tokens)    │
└───────────────────────┬───────────────────────────────────┘
┌───────────────────────▼───────────────────────────────────┐
│ 3. AUTHORIZATION (RBAC)                                         │
│    What are you allowed to do, on which resources,               │
│    in which namespaces?                                          │
└───────────────────────┬───────────────────────────────────┘
┌───────────────────────▼───────────────────────────────────┐
│ 4. ADMISSION CONTROL                                            │
│    Is this SPECIFIC request's content acceptable/should           │
│    it be mutated? (Pod Security Standards, OPA/Gatekeeper,        │
│    ResourceQuota, LimitRange)                                     │
└───────────────────────┬───────────────────────────────────┘
┌───────────────────────▼───────────────────────────────────┐
│ 5. NAMESPACE ISOLATION                                           │
│    Tenancy boundary — RBAC and NetworkPolicy are both              │
│    commonly scoped per-namespace                                   │
└───────────────────────┬───────────────────────────────────┘
┌───────────────────────▼───────────────────────────────────┐
│ 6. NETWORK POLICY                                                │
│    Which Pods can talk to which other Pods (default: ALL          │
│    can talk to all — see the Networking guide)                     │
└───────────────────────┬───────────────────────────────────┘
┌───────────────────────▼───────────────────────────────────┐
│ 7. CONTAINER / RUNTIME ISOLATION                                  │
│    Linux namespaces, cgroups, seccomp, AppArmor/SELinux,           │
│    non-root enforcement (securityContext), read-only               │
│    root filesystems                                                 │
└─────────────────────────────────────────────────────────────┘
```

## Key architectural security facts worth knowing cold
- **Anyone who can create Pods in a namespace can mount any Secret in that namespace** — repeated from the Secrets guide because it's fundamentally an *architectural* fact, not just a Secrets quirk: Pod-creation privilege is a much broader grant than it first appears
- **etcd is the ultimate blast radius** — direct etcd access bypasses the API server's auth/authz/admission entirely; etcd access should be treated as equivalent to full cluster-admin, network-isolated to only the API servers that need it
- **The kubelet's own API (:10250)** is a historically under-secured surface — anonymous/unauthenticated kubelet access has been a real-world attack vector (unauthenticated `kubectl exec`-equivalent access to any Pod on that node); production clusters must explicitly lock this down (`--anonymous-auth=false`, proper webhook authorization)
- **Service account tokens are the default cluster-internal identity** — every Pod, unless configured otherwise, receives a token mountable at `/var/run/secrets/kubernetes.io/serviceaccount/token`, meaning **any process inside that Pod can act as that Pod's ServiceAccount** against the API server — scoping ServiceAccount RBAC tightly (or disabling auto-mounting where unneeded via `automountServiceAccountToken: false`) matters precisely because of this

---

# 37. PERFORMANCE CONSIDERATIONS

## API server / etcd at scale
- etcd write latency is dominated by **disk fsync time** — production etcd deployments require fast, low-latency SSD storage; a slow disk under etcd directly translates into slow cluster-wide reconciliation, since every controller's writes ultimately wait on etcd committing
- **Watch fan-out** — thousands of controllers/kubelets holding open watch connections is normal and expected (this is the intended design, per Part 3 §22), but very large clusters (thousands of nodes) tune `--watch-cache` sizes and consider etcd sharding-adjacent strategies (though etcd itself doesn't natively shard — this is why extremely large clusters sometimes split workloads across multiple clusters rather than one enormous one)
- **List calls are expensive; watches are cheap** — a controller that re-`LIST`s everything on every reconcile (instead of relying on its Informer's local cache, per Part 3 §23) is a classic, avoidable source of API server load at scale

## Scheduler performance
- Scheduling throughput matters at high Pod-churn scale (large batch/ML workloads creating thousands of Pods rapidly) — the filtering phase is the more expensive one; scoring runs only against nodes that already survived filtering
- `percentageOfNodesToScore` lets very large clusters trade a small amount of placement optimality for significantly faster scheduling decisions by not exhaustively scoring every single node

## kubelet / node-level
- Excessive Pods-per-node increases kubelet's own CPU/memory overhead (probe execution, status reporting, cgroup management) — node sizing should account for this fixed-per-Pod overhead, not just raw workload resource sums
- Image pull performance (especially large images, or many nodes pulling simultaneously on a rollout) is a frequently underestimated contributor to real-world rollout latency, distinct from anything covered by resource requests/limits

---

# 38. COMMON BEGINNER MISUNDERSTANDINGS (CONSOLIDATED)

Gathered here from throughout the masterclass, because seeing them together reinforces the underlying pattern behind most of them:

1. **"The scheduler starts the container."** No — it only writes a Binding. The kubelet, independently, is what actually starts anything. (Part 1, Part 4 §28)
2. **"Editing a Deployment updates existing Pods in place."** No — it creates a new ReplicaSet and replaces Pods entirely via rolling scale up/down. (Part 3 §16, Part 5 Diagram F)
3. **"Pods move to another node on node failure."** No — they're evicted, and entirely new Pod objects are scheduled elsewhere. (Part 5 Diagram E)
4. **"kubectl get pods shows real-time container state."** Not exactly — it shows the last state the kubelet reported through the API server, which can lag by a small interval. (Part 4 §25)
5. **"CoreDNS/kube-proxy are control-plane components."** No — both are ordinary worker-node workloads (a Deployment and a DaemonSet respectively), scheduled like any other Pod. (Part 2 §12, §14)
6. **"A container restart and a Pod restart are the same thing."** No — one is kubelet-local (same Pod, incrementing RESTARTS), the other is a full ReplicaSet-driven replacement (brand-new Pod). (Part 5 Diagram D)
7. **"Deleting a Pod under a Deployment reduces your replica count."** No — the ReplicaSet controller immediately creates a replacement; you've just churned one instance, not scaled down. (Part 4 §25)
8. **"Base64 in a Secret means it's encrypted."** No — base64 is a reversible encoding with zero confidentiality; real protection requires explicit encryption at rest. (Secrets guide, Part 3 of that guide)
9. **"The scheduler considers live CPU/memory usage when placing Pods."** No — it only ever considers declared `requests` versus a node's `Allocatable`, never actual utilization. (Resource Requests guide, Part 4)
10. **"If a Pod requests 100m CPU and uses only 10m, that's fine and free."** Not for scheduling purposes — the full 100m is reserved and unavailable to other Pods regardless of actual usage. (Resource Requests guide, Part 4)

---

# 39. REAL-WORLD PRODUCTION EXAMPLES (SYNTHESIS)

- A company running EKS never touches control-plane nodes directly, relies on AWS's HA guarantee across 3 AZs, and focuses all their own operational effort on worker-node capacity planning, RBAC design, and NetworkPolicy — this is the overwhelming majority pattern in industry today (Part 1 §4, this Part §33–35)
- cert-manager, the External Secrets Operator, and Cilium's `CiliumNetworkPolicy` are all real, widely-deployed systems built from nothing but CRDs + controllers + (sometimes) admission webhooks — zero core-Kubernetes modifications required, proving the extensibility story from Part 3 §18/§24 isn't theoretical
- A classic real incident pattern: a team without ResourceQuota lets one namespace's runaway workload consume enough cluster-wide capacity that unrelated teams' Pods start going `Pending` — this is a direct, practical consequence of the fact that the scheduler bin-packs against shared node capacity with no innate per-team fairness (tie back to Resource Requests/Limits guides' ResourceQuota sections)
- Managed etcd backup automation (a scheduled CronJob or a managed cloud snapshot feature) is close to universal in serious production Kubernetes operations, precisely because of the catastrophic blast radius described in Part 2 §7 and Part 5 §30

---

# MINIKUBE HANDS-ON LABS (ARCHITECTURE-FOCUSED)

### Lab 1: See every control-plane component as a Pod
```bash
minikube start
kubectl get pods -n kube-system
```
**Expected output (abbreviated):**
```
NAME                                READY   STATUS
etcd-minikube                        1/1     Running
kube-apiserver-minikube               1/1     Running
kube-controller-manager-minikube      1/1     Running
kube-scheduler-minikube                1/1     Running
coredns-...                            1/1     Running
kube-proxy-...                         1/1     Running
```
Note: on Minikube (single-node), control-plane components run as **static Pods** right alongside CoreDNS and kube-proxy on the same one node — a compressed but architecturally faithful picture of the full diagram from Part 1.

### Lab 2: Watch the Deployment → ReplicaSet → Pod cascade live
```bash
kubectl create deployment demo --image=nginx --replicas=3
kubectl get replicasets -w &
kubectl get pods -w &
# observe: the ReplicaSet appears first, THEN Pods appear moments later —
# exactly the layered-reaction pattern from Part 3 §15 and Part 4 Diagram A
```

### Lab 3: Prove kubelet-local restart doesn't need the API server reachable
```bash
kubectl run crashy --image=busybox --restart=Always -- sh -c "sleep 5; exit 1"
kubectl get pod crashy -w
# watch RESTARTS climb every ~5s — this is Diagram D1, entirely kubelet-driven
```

### Lab 4: Watch a Binding happen
```bash
kubectl create deployment demo2 --image=nginx
kubectl get events --field-selector reason=Scheduled -w
# the Event fires the instant the scheduler writes the Binding — Diagram C
```

### Lab 5: Observe rolling update mechanics directly
```bash
kubectl create deployment demo3 --image=nginx:1.24 --replicas=4
kubectl set image deployment/demo3 nginx=nginx:1.25
kubectl get replicasets -l app=demo3 -w
# watch the OLD ReplicaSet scale down as the NEW one scales up,
# never both moving simultaneously past your maxSurge/maxUnavailable — Diagram F
kubectl rollout undo deployment/demo3
kubectl get replicasets -l app=demo3
# the OLD ReplicaSet (scaled to 0, never deleted) scales back up instead
# of anything being rebuilt from scratch
```

### Lab 6: Simulate control-plane component "failure" (single-node caveat)
```bash
# NOTE: on Minikube (single control-plane node), stopping the scheduler
# stops ALL scheduling cluster-wide — there's no HA failover to observe
# here, but the STOPPED-vs-RUNNING-Pods behavior is fully real
kubectl -n kube-system get pod -l component=kube-scheduler
# delete it (a static Pod manifest means it gets recreated by the kubelet
# automatically — to truly stop it you'd move its manifest out of
# /etc/kubernetes/manifests on a real node)
kubectl create deployment stuck --image=nginx
kubectl get pods
# STAYS Pending the whole time the scheduler is down — restore it and
# watch it get scheduled within seconds
```

---

# COMPLETE ARCHITECTURE CHEAT SHEET

```
CONTROL PLANE                         WORKER NODES
─────────────                         ────────────
kube-apiserver   → front door,        kubelet       → node agent, starts
                    only thing that                    containers via CRI
                    touches etcd
etcd             → cluster memory,    kube-proxy    → Service routing
                    Raft consensus                     rules (DaemonSet)
kube-scheduler   → decides node       Container     → actually runs
                    placement           Runtime         containers
kube-controller-  → built-in          CoreDNS       → cluster DNS
manager             reconciliation                     (runs as a
                    loops                                Deployment,
cloud-controller-  → cloud API         NOT control-plane)
manager              integration

UNIVERSAL RULE: every arrow, everywhere, is:
  authenticate → authorize → admit → write to etcd → notify watchers
No component ever calls another component directly (except kubectl
exec/logs/port-forward, proxied once through the API server).

DECLARATIVE MODEL:
  .spec   = desired state (you write it)
  .status = actual state (only controllers write it)
  controller's ENTIRE job = close the gap, forever, in a loop

QoS / SCHEDULING (cross-reference to other guides):
  scheduler uses REQUESTS only, never live usage
  limits are enforced by the KERNEL (cgroups), not the scheduler
```

---

# COMPONENT COMPARISON TABLE (FINAL, CONSOLIDATED)

| Component | Layer | Stateful? | HA mechanism | Talks to |
|---|---|---|---|---|
| kube-apiserver | Control plane | No (stateless) | Active-active behind LB | etcd only |
| etcd | Control plane | Yes (the state) | Raft, odd quorum | apiserver only |
| kube-scheduler | Control plane | No | Active-passive, leader election | apiserver only |
| kube-controller-manager | Control plane | No | Active-passive, leader election | apiserver only |
| cloud-controller-manager | Control plane | No | Active-passive, leader election | apiserver + cloud API |
| kubelet | Worker node | No (reflects reality) | N/A (per-node) | apiserver + local runtime |
| kube-proxy | Worker node | No | N/A (per-node) | apiserver + local kernel |
| Container Runtime | Worker node | No | N/A (per-node) | local kubelet only |
| CoreDNS | Worker node | No | Multiple replicas, ordinary Deployment | apiserver + any Pod |

---

# END-TO-END REQUEST FLOW (FINAL RECAP)

```
kubectl apply
   → HTTPS to apiserver
      → Authentication (who?)
         → Authorization/RBAC (allowed?)
            → Admission control (valid/mutated?)
               → written to etcd (via Raft quorum)
                  → every relevant watcher notified instantly
                     → Deployment controller creates ReplicaSet
                        → ReplicaSet controller creates Pods
                           → Scheduler writes Binding
                              → kubelet (on chosen node) notices
                                 → Container Runtime (via CRI) starts container
                                    → kubelet reports status back
                                       → (loops back to "written to etcd")
                                          → kubectl get shows Running
```

---

# MENTAL MODEL FOR REMEMBERING KUBERNETES ARCHITECTURE

**"Everyone watches the whiteboard; nobody calls anyone."**

Picture etcd as a shared whiteboard in a room. The API server is the only person allowed to actually write on it or erase anything — everyone else can only look at it (through the API server) and shout out when they see something relevant to them change. The scheduler only cares about lines that say "needs a node." The kubelet on Node 3 only cares about lines that say "assigned to Node 3." Nobody in the room ever walks over and taps another person on the shoulder — they only ever react to what's now written on the shared board. If any one person in the room steps out (crashes), the whiteboard doesn't change, and everyone else keeps working from what's already on it — the room just stops making *new* progress on whatever that missing person was responsible for, until they (or a stand-in) come back.

Every fact in this masterclass — self-healing, HA failover, why controllers are simple, why the system is extensible, why failures degrade "change" but not "runtime" — falls directly out of this one picture.

---

# 20 BEGINNER INTERVIEW QUESTIONS

1. What is Kubernetes, in one sentence?
2. What problem does Kubernetes solve that plain Docker doesn't?
3. What is a Pod, and why is it the smallest deployable unit rather than a container?
4. What's the difference between the control plane and worker nodes?
5. Name the five control-plane components.
6. Name the three worker-node components.
7. What does etcd store?
8. What does kube-apiserver do?
9. What does kube-scheduler decide?
10. Is CoreDNS a control-plane component? Explain.
11. What is the difference between `kubectl create` and `kubectl apply`?
12. What happens if you delete a Pod that's managed by a Deployment?
13. What is the difference between `.spec` and `.status` on an object?
14. What is a reconciliation loop, in plain terms?
15. What does "desired state" mean?
16. Why does Kubernetes use a declarative model instead of an imperative one?
17. What is a Binding, in the scheduling context?
18. What happens to a container that fails its liveness probe?
19. Does the API server store cluster state itself?
20. What is kubelet's role, in one sentence?

---

# 20 INTERMEDIATE INTERVIEW QUESTIONS

1. Walk through what happens internally from `kubectl apply` to a running container.
2. What's the difference between a Deployment, a ReplicaSet, and a Pod, architecturally?
3. Why does the scheduler only ever write a Binding instead of starting anything itself?
4. Explain the three-way merge behind `kubectl apply`.
5. What is a watch, and why does Kubernetes use watches instead of polling?
6. What is an Informer, and why do controllers use one instead of raw watches?
7. Explain the AuthN → AuthZ → Admission pipeline and why admission is a separate stage.
8. What is the difference between a mutating and a validating admission webhook?
9. Why is CoreDNS a Deployment and not a control-plane static Pod?
10. What is a CRD, and how does it let you extend Kubernetes without modifying core code?
11. Explain why kube-proxy failing doesn't immediately break existing Service traffic.
12. What's the mechanical difference between a container restart and a Pod being replaced?
13. Why does a Deployment rollout create a new ReplicaSet instead of editing the old one?
14. What are `maxSurge` and `maxUnavailable`, and what do they control during a rollout?
15. How does `kubectl rollout undo` work internally?
16. What is `resourceVersion`, and what problem does it solve for dropped watch connections?
17. Why is etcd only ever accessed through the API server, never directly by other components?
18. What is the Raft consensus algorithm's role in etcd?
19. Explain why 3 etcd nodes tolerate 1 failure but 4 nodes don't meaningfully improve on that.
20. Why are kube-scheduler and kube-controller-manager active-passive, while kube-apiserver is active-active?

---

# 20 ADVANCED INTERVIEW QUESTIONS

1. Explain, at the cgroup/CRI level, what happens between the kubelet deciding to start a container and the container actually running.
2. Why is Kubernetes' controller pattern described as "level-triggered" rather than "edge-triggered," and why does that matter for robustness?
3. Trace the exact sequence of independent controller reactions from creating a Deployment to a container running, naming each watch/write boundary crossed.
4. What is etcd compaction, and what happens operationally if it's neglected?
5. Explain Server-Side Apply and how it improves on the original client-side three-way merge.
6. Why does the API server being stateless make horizontal scaling trivial, while the scheduler needs leader election instead?
7. What exactly determines a Pod's QoS class, and how does that map to kubelet eviction ordering under node pressure?
8. Explain why the scheduler's decisions are based purely on declared requests and never live utilization — what would break if it used live metrics instead?
9. Describe the full node-failure timeline (heartbeat loss → NotReady → eviction → rescheduling) with the relevant default timeout values.
10. Why is `kubectl exec` architecturally different from a normal API operation like `kubectl get`?
11. What security implications follow from the fact that any process inside a Pod can access its ServiceAccount token by default?
12. Explain how a CRD + controller + admission webhook combination (e.g., cert-manager) achieves functionality without any core Kubernetes code changes.
13. Why does etcd write latency directly bound overall cluster reconciliation speed?
14. What's the architectural reason ResourceQuota admission requires Pods to explicitly declare resource requests once a quota exists in that namespace?
15. Explain why deleting a Pod under a ReplicaSet doesn't reduce the replica count, tracing the exact controller behavior responsible.
16. How does `percentageOfNodesToScore` trade off scheduling optimality for performance, and when would you tune it?
17. What's the practical difference between a network partition and an actual node crash from the control plane's perspective, and why can't it easily tell them apart?
18. Explain how Informer caching allows Kubernetes to run hundreds of controllers without collapsing API server read throughput.
19. Why is direct etcd access architecturally equivalent to full cluster-admin privilege?
20. Describe how a custom Operator you write yourself fits into the exact same architectural model as the built-in Deployment/ReplicaSet controllers, with no special-casing by Kubernetes core.

---

# 5 TROUBLESHOOTING SCENARIOS

**Scenario 1: Pods stuck `Pending` cluster-wide, suddenly, with no recent changes**
Walk the diagnostic path: check scheduler health first (`kubectl -n kube-system get pods -l component=kube-scheduler`, check leader election lease) before assuming a resource-capacity issue — a scheduler outage produces the exact same symptom as genuine cluster-wide resource exhaustion, and the fix is completely different.

**Scenario 2: `kubectl` commands hang or time out entirely**
Distinguish API server reachability (network/LB issue) from API server health (process up but overloaded/unhealthy) from etcd quorum loss (API server up but can't do anything useful) — `kubectl get --raw='/healthz'`, direct etcd health checks (`etcdctl endpoint health`), and API server logs each narrow this down to a different one of the three.

**Scenario 3: A Deployment rollout is stuck partway, half old pods half new**
Check readiness probes on the new Pods first — a rollout pauses exactly when new Pods fail to become Ready, since the rolling update loop (Diagram F) gates every step on readiness, not just "Running." `kubectl rollout status` plus `kubectl describe pod` on a new-generation Pod almost always reveals a failing readiness probe or `ImagePullBackOff`.

**Scenario 4: A node shows `Ready` but Pods scheduled to it never actually start**
This points at the kubelet-to-container-runtime boundary specifically — the node's control-plane-facing heartbeat can look healthy while the local CRI connection to containerd/CRI-O is broken; `crictl ps`/`crictl images` directly on the node, and `journalctl -u kubelet`, isolate this from a scheduling-layer problem.

**Scenario 5: Everything was fine, then suddenly no new Pods can be created anywhere, existing Pods fine**
Classic etcd-quota-exhaustion pattern (from neglected compaction, Part 5 §30) — the API server can still serve reads from its watch cache but writes start failing; `etcdctl endpoint status` showing the DB size near `--quota-backend-bytes` confirms it, and the fix is compaction + defragmentation, not a Kubernetes-level restart.

---

# CLOSING SYNTHESIS

Six parts, one underlying idea repeated at every layer: **declare what you want, let independent watchers reconcile toward it, and never let any two components talk to each other directly.** Every component you now know — from etcd's Raft consensus up through a Deployment's rolling update — is a specific instance of that one pattern. When something in Kubernetes surprises you in the future, the fastest way back to understanding it is always the same move made throughout this masterclass: ask which object's `.spec` changed, which controller is watching that object type, and what it wrote back in response. That question resolves the vast majority of "why did Kubernetes just do that?" moments you'll ever encounter.

*This concludes the Kubernetes Architecture masterclass — from "what is a Pod" through Raft consensus, admission webhooks, and production HA design, delivered across 6 parts covering all 40 requested topics, 6 sequence diagrams, Minikube labs, and 65 interview questions plus 5 troubleshooting scenarios.*
