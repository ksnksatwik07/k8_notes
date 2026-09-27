# Kubernetes Resource Requests — Complete Mastery Guide
### From Absolute Beginner to Production Scheduling Expertise

---

# PART 1 — WHY RESOURCE REQUESTS EXIST

## The core problem

A cluster is a fixed pool of physical (or virtual) CPU and memory, spread across a set of nodes. Kubernetes must decide **which node each Pod runs on** — and it needs a number to reason with. Without any declared resource need, the scheduler is flying blind: it can't tell whether a node has "enough room" for a new Pod, and it can't guarantee fairness when nodes get busy.

**A resource request is a Pod's declared minimum requirement** — "I need at least this much CPU and memory to function." It exists to answer two questions:

1. **Scheduling**: which nodes even have room for this Pod?
2. **Fairness under contention**: when a node runs out of actual resources, who gets protected and who gets sacrificed?

Without requests, Kubernetes would either wildly overpack nodes (crashing everything when real usage spikes) or have no rational way to bin-pack Pods at all.

## CPU

CPU is a **compressible** resource — if a container asks for more CPU than is available, the kernel scheduler just gives it a smaller time-slice. The process slows down but keeps running. Nobody gets killed for CPU pressure.

## Memory

Memory is an **incompressible** resource — you cannot "slow down" memory usage; a byte is either allocated or it isn't. When memory runs out, something has to be killed (the OOM killer) to reclaim it. This single distinction — compressible vs incompressible — is the reason CPU and memory are enforced completely differently, covered in Part 5.

---

# PART 2 — UNITS

## CPU units: CPU, millicores, 100m, 500m, 1 CPU

CPU is measured in **CPU units**, where `1` CPU unit equals:
- 1 physical CPU core, or
- 1 virtual core (a vCPU on AWS/GCP/Azure), or
- 1 hyperthread on a hyperthreaded physical core

Fractional CPU is expressed in **millicores** (thousandths of a CPU), using the `m` suffix:

| Notation | Meaning |
|---|---|
| `1` or `1000m` | one full CPU core |
| `500m` | half a CPU core |
| `100m` | one tenth of a CPU core |
| `2` or `2000m` | two full CPU cores |
| `250m` | a quarter of a CPU core |

```yaml
resources:
  requests:
    cpu: "250m"     # needs at least a quarter core to function acceptably
```
There is no upper bound tied to any single number — `4` CPU just means 4 full cores' worth of compute time, achievable across multiple physical cores in parallel (CPU is not pinned to one core by default).

## Memory units: Mi, Gi, M, G

Memory has **two different unit families** that look similar but are **not** interchangeable:

| Suffix | Base | Meaning | Example |
|---|---|---|---|
| `M` (Megabyte) | Decimal, base 1000 | 1 M = 1,000,000 bytes | `500M` |
| `Mi` (Mebibyte) | Binary, base 1024 | 1 Mi = 1,048,576 bytes | `512Mi` |
| `G` (Gigabyte) | Decimal, base 1000 | 1 G = 1,000,000,000 bytes | `2G` |
| `Gi` (Gibibyte) | Binary, base 1024 | 1 Gi = 1,073,741,824 bytes | `2Gi` |

**Kubernetes convention overwhelmingly uses the binary (`Mi`/`Gi`) forms** — this matches how memory is actually addressed and how tools like `free -h`, `docker stats`, and most monitoring dashboards report it. Using `M`/`G` isn't wrong, but mixing conventions across a team causes confusing off-by-~5% discrepancies when comparing numbers (e.g., `500M` = 476.8Mi, not 500Mi) — pick one convention and standardize on it.

```yaml
resources:
  requests:
    memory: "256Mi"    # ~268 million bytes
```

---

# PART 3 — YAML: REQUESTS IN PRACTICE

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-app
spec:
  containers:
    - name: app
      image: myapp:1.0
      resources:
        requests:
          cpu: "250m"       # scheduler guarantee: at least a quarter core
          memory: "256Mi"   # scheduler guarantee: at least 256 MiB
        limits:
          cpu: "500m"       # hard ceiling: throttled beyond this
          memory: "512Mi"   # hard ceiling: OOM-killed beyond this
```

**Line-by-line:**
- `resources.requests` — what the scheduler uses to find a node with room, and what's *reserved* for this container even if it's not actively using it
- `resources.limits` — the enforced ceiling (covered fully in Part 5's QoS discussion, since limits directly determine QoS class); this guide's primary focus is `requests`, but the two are inseparable in practice
- Every container in a multi-container Pod declares its **own** `resources` block — the Pod's total footprint is the **sum** across all containers (including init containers, handled slightly differently — see Part 8)

---

# PART 4 — HOW THE SCHEDULER ACTUALLY USES REQUESTS

## Node capacity vs allocatable resources

Every node has:
```
Capacity        = the node's total physical/virtual resources
Allocatable     = Capacity − (resources reserved for the OS, kubelet, and system daemons)
```
```bash
kubectl describe node <node-name>
```
```
Capacity:
  cpu:     4
  memory:  16Gi
Allocatable:
  cpu:     3800m       # some reserved for kubelet/OS via --kube-reserved / --system-reserved
  memory:  15Gi
```
**The scheduler only ever bin-packs against `Allocatable`, never `Capacity`.** This reserved slice exists so that node-level system processes (kubelet, container runtime, OS) never get starved out by aggressively-packed Pods.

## Requested resources (already committed on a node)

```bash
kubectl describe node <node-name>
```
```
Non-terminated Pods:  (12 in total)
  Namespace     Name          CPU Requests   Memory Requests
  default       app-a         250m            256Mi
  default       app-b         500m            512Mi
  ...
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource     Requests       Limits
  cpu          2500m (65%)    4000m (105%)
  memory       6Gi (40%)      10Gi (66%)
```
This is the running tally the scheduler consults: **sum of all requests from Pods already scheduled onto this node.**

## The actual scheduling decision (numerical walkthrough)

```
Node "worker-1"
  Allocatable:  CPU 3800m,  Memory 15Gi
  Already scheduled (sum of requests):  CPU 3200m,  Memory 12Gi
  Remaining "room":  CPU 600m,  Memory 3Gi

New Pod requests:  CPU 250m,  Memory 256Mi
  → 250m ≤ 600m  ✓
  → 256Mi ≤ 3Gi  ✓
  → FITS. Scheduler may place it here.

Another new Pod requests:  CPU 700m,  Memory 512Mi
  → 700m > 600m  ✗
  → Does NOT fit on worker-1, regardless of memory.
  → Scheduler looks at other nodes instead.
```

This is a pure **bin-packing** decision based entirely on declared requests — **the scheduler never looks at actual live CPU/memory usage** when deciding placement. A node could be sitting at 5% real CPU utilization but still be considered "full" if the sum of requests already scheduled onto it equals its allocatable capacity. This surprises almost everyone the first time they see it.

```
┌─────────────────────── Node "worker-1" (Allocatable: 3800m CPU) ───────────────────────┐
│ ██████████████████████████████████████████░░░░░░░░░░░░░░░░░                            │
│ [ app-a: 250m ][ app-b: 500m ][ app-c: 1000m ][ app-d: 1450m ]  ← 3200m requested (84%) │
│                                                        [ room: 600m free ]              │
└──────────────────────────────────────────────────────────────────────────────────────────┘
  ↑ This bar represents REQUESTED capacity, not actual live usage — a Pod using
    only 10m of its 1000m request still "occupies" the full 1000m for scheduling purposes.
```

## Overcommitment

**Requests** are what the scheduler enforces at placement time. **Limits** can be set *higher* than requests (or omitted entirely), which means the **sum of limits across all Pods on a node can legitimately exceed the node's actual capacity** — this is called overcommitment, and it's deliberate, not a bug.

```
Node capacity: 4 CPU
Pod A: request 500m, limit 2000m
Pod B: request 500m, limit 2000m
Pod C: request 500m, limit 2000m
Pod D: request 500m, limit 2000m
                        ↑
Sum of REQUESTS: 2000m (fits comfortably — scheduler is happy)
Sum of LIMITS:   8000m (200% overcommitted — fine, AS LONG AS not all 4 burst simultaneously)
```
This is exactly how cloud providers (and Kubernetes clusters) achieve efficient utilization — most workloads don't use their peak burst capacity simultaneously, so overcommitting limits (while still scheduling conservatively on requests) lets you run more workloads per node than a naive 1:1 allocation would allow. The risk: if enough Pods *do* burst simultaneously, CPU gets throttled cluster-wide (compressible, tolerable) or memory pressure triggers OOM kills (incompressible, more disruptive) — see Part 5.

---

# PART 5 — QoS CLASSES

Every Pod is automatically assigned one of three **Quality of Service** classes based purely on how its `requests`/`limits` are set — this determines **eviction priority** when a node comes under actual resource pressure.

## Guaranteed
**Every container** in the Pod has `requests == limits` for **both** CPU and memory, explicitly set.
```yaml
resources:
  requests: { cpu: "500m", memory: "512Mi" }
  limits:   { cpu: "500m", memory: "512Mi" }   # identical to requests
```
Highest protection — last to be evicted under node pressure. Used for critical, latency-sensitive workloads (databases, control-plane components).

## Burstable
At least one container has a request set, but requests ≠ limits (or limits are unset) for at least one resource.
```yaml
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits:   { cpu: "500m", memory: "512Mi" }   # different from requests
```
Middling protection — evicted before Guaranteed, after BestEffort, roughly proportional to how far actual usage exceeds requests.

## BestEffort
**No requests or limits set at all**, on any container.
```yaml
resources: {}    # nothing specified
```
Lowest protection — **first to be killed** under any node memory pressure, regardless of how little memory it's actually using at that moment. This is the most common accidental production mistake: a team forgets to set requests "because the app is small," and it becomes the first thing sacrificed under any pressure, however unrelated to that Pod.

```
Node under memory pressure — kubelet's eviction order:
  1. BestEffort Pods first (no guarantees given, none owed)
  2. Burstable Pods exceeding their own requests, worst-offender first
  3. Guaranteed Pods — only as an absolute last resort
```

---

# PART 6 — WHAT HAPPENS IF REQUESTS ARE WRONG

## Requests too high

```
Pod requests: CPU 4000m, Memory 8Gi
Node's total allocatable: CPU 3800m, Memory 15Gi
```
- The Pod **cannot be scheduled anywhere** if no node has that much *allocatable* CPU free — it sits in `Pending` state indefinitely
- Wastes cluster capacity: a Pod requesting far more than it actually uses "reserves" that headroom uselessly, preventing other Pods from being scheduled onto that space even though the resource sits idle
- Inflated cloud bills — teams commonly over-provision "just to be safe," directly translating to paying for unused reserved capacity across every node

## Requests too low

- The Pod schedules easily (looks like it needs almost nothing) but then, under real load, actually consumes far more CPU/memory than declared
- For CPU: the container simply gets throttled against its **limit** (if lower than actual demand) or competes fairly for spare cycles — degraded performance, not a crash
- For memory: if actual usage exceeds the **limit**, the container is OOM-killed outright, regardless of how low the request was — a Pod that requested 64Mi but actually needs 512Mi will be repeatedly killed the moment it crosses whatever limit was set (or the node's own memory ceiling, if no limit was set at all)
- Also **destabilizes other Pods on the same node**: since the scheduler trusted a too-low request when packing the node, actual usage exceeding it can starve neighboring Pods of real CPU/memory the scheduler thought was still "free"

## Node has insufficient resources (Pod stuck Pending)

```bash
kubectl describe pod <pod>
```
```
Events:
  Warning  FailedScheduling  0/5 nodes are available:
           3 Insufficient cpu, 2 Insufficient memory.
```
This message tells you exactly how many nodes were rejected and why — the scheduler evaluated every node, and none had enough *allocatable minus already-requested* room. Fixes: add nodes (cluster autoscaler), reduce the Pod's requests if they're inflated, or free up room by removing/rightsizing other workloads.

---

# PART 7 — DIAGRAM: THE FULL DECISION FLOW

```
                    New Pod submitted (requests: CPU 250m, Mem 256Mi)
                                     │
                                     ▼
                     ┌───────────────────────────────┐
                     │   Scheduler filtering phase    │
                     │  for each node, compute:       │
                     │  Allocatable − SumOfRequests   │
                     │  = remaining room               │
                     └───────────────┬─────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              ▼                      ▼                      ▼
        Node A: room             Node B: room            Node C: room
        CPU 600m, Mem 3Gi        CPU 100m, Mem 4Gi        CPU 2000m, Mem 8Gi
        250m ≤ 600m ✓            250m > 100m ✗             250m ≤ 2000m ✓
        256Mi ≤ 3Gi ✓            REJECTED                  256Mi ≤ 8Gi ✓
        → candidate               (insufficient CPU)        → candidate
                                     │
                                     ▼
                    ┌───────────────────────────────┐
                    │   Scheduler scoring phase       │
                    │  ranks remaining candidates      │
                    │  (e.g., least-requested,          │
                    │   balanced-allocation, spread)    │
                    └───────────────┬─────────────────┘
                                     ▼
                          Pod bound to highest-scored node
                                     │
                                     ▼
                    Node's "Allocated resources" tally updates:
                    +250m CPU, +256Mi Memory now reserved
```

---

# PART 8 — INIT CONTAINERS AND MULTI-CONTAINER PODS

- **Regular containers**: the Pod's *effective request* is the **sum** of every container's request (they all run concurrently).
- **Init containers**: run sequentially, one at a time, *before* any regular container starts — so the Pod's effective request for scheduling purposes is `max(each init container's request, sum of all regular containers' requests)`, not an additive sum with init containers included.

```yaml
spec:
  initContainers:
    - name: migrate-db
      resources: { requests: { cpu: "1000m", memory: "1Gi" } }
  containers:
    - name: app
      resources: { requests: { cpu: "250m", memory: "256Mi" } }
    - name: sidecar
      resources: { requests: { cpu: "100m", memory: "128Mi" } }
```
Effective Pod request = `max(1000m, 250m+100m)` CPU = **1000m**, and `max(1Gi, 256Mi+128Mi)` memory = **1Gi** — because the init container's peak need briefly exceeds what the regular containers need once they're running.

---

# PART 9 — TROUBLESHOOTING SCENARIOS

**Pod stuck in `Pending`**
```bash
kubectl describe pod <pod>          # check Events for "Insufficient cpu/memory"
kubectl describe nodes | grep -A5 "Allocated resources"
kubectl top nodes                   # actual live usage, for context (not what scheduler used)
```
Distinguish: is this "no node has enough *allocatable* capacity" (add nodes / lower requests) vs. a completely different scheduling constraint (taints, affinity, PodDisruptionBudget)? The Events message is explicit about which.

**Pod repeatedly OOMKilled**
```bash
kubectl describe pod <pod> | grep -A5 "Last State"
# Reason: OOMKilled
```
The container's actual memory usage exceeded its **limit** (or the node's own hard ceiling if no limit was set). Fix: profile actual usage (`kubectl top pod`), raise the limit to a realistic ceiling with headroom, and set the request close to typical steady-state usage — not the absolute peak.

**Node shows high "Allocated resources %" but `kubectl top node` shows low real usage**
This is expected and correct behavior, not a bug — it means requests are set conservatively/generously relative to actual usage. It only becomes a genuine efficiency problem if it's blocking other Pods from scheduling that could otherwise fit based on real usage; the fix is rightsizing requests downward based on observed `kubectl top` history, not just chasing lower node CPU/memory readings.

**Pod evicted despite low actual usage**
Check its QoS class:
```bash
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'
```
If `BestEffort`, this is expected under any memory pressure — it has zero eviction protection regardless of how little it was actually using. Fix: add at least a memory request/limit to move it to `Burstable` or `Guaranteed`.

**Cluster autoscaler not adding nodes despite Pending Pods**
- Confirm the Pending Pods' requests are actually achievable by an available instance type in the node pool (a request for 32 CPU won't trigger a scale-up if the largest configured instance type only has 16)
- Check autoscaler logs for scheduling simulation failures unrelated to sheer size (taints/tolerations, node selectors, zone constraints)

---

# PART 10 — INTERVIEW QUESTIONS

**Fundamentals**
1. What is a resource request, and what two things does it directly influence?
2. Why is CPU called "compressible" and memory "incompressible" — and why does that distinction matter operationally?
3. What's the difference between a node's Capacity and its Allocatable resources?

**Units**
4. What does `500m` CPU mean, and how does it relate to `0.5` CPU?
5. What's the actual byte difference between `1G` and `1Gi` of memory?

**Scheduling**
6. Does the scheduler consider a Pod's actual live resource usage when deciding placement? Explain your answer.
7. Walk through, numerically, how the scheduler decides whether a Pod fits on a given node.
8. What does it mean for a node to be "overcommitted," and why is this often done deliberately?

**QoS**
9. Name the three QoS classes and what determines which one a Pod receives.
10. Under memory pressure, in what order does the kubelet evict Pods, and why?
11. Why is a Pod with zero resources specified more dangerous in production than one that's simply Burstable?

**Consequences of misconfiguration**
12. What concretely happens if a Pod's request is far higher than it needs?
13. What concretely happens if a Pod's request is far lower than its real usage, both for CPU and for memory — and why do the two differ?
14. How does init container resource sizing affect a Pod's total effective request?

**Troubleshooting (scenario-based)**
15. A Pod is stuck `Pending` — what commands do you run, and what are you looking for?
16. A Pod keeps getting OOMKilled even though `kubectl top pod` shows moderate usage most of the time — what's your hypothesis and how do you confirm it?
17. Cluster autoscaler isn't scaling up despite Pending Pods — what are the possible causes?

---

*This guide covers Kubernetes resource requests from CPU/memory fundamentals through scheduler mechanics, QoS-driven eviction behavior, and production troubleshooting — the complete arc from beginner to scheduling and resource-management expert.*
