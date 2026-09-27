# Kubernetes Resource Management — Complete SRE Guide
### Node Capacity, Scheduling, QoS, and Eviction as One Unified System

*(This guide assumes familiarity with the mechanics already covered in the Resource Requests and Resource Limits guides — CPU/memory units, the scheduler's bin-packing behavior, and the three QoS classes. Rather than re-deriving those, this guide's job is to connect them into one end-to-end chain and add the piece neither of those guides covered in depth: node-level pressure and the eviction manager.)*

---

# THE FULL CHAIN, AS ONE PICTURE

```
Node Capacity
   │  (physical/virtual hardware totals)
   ▼
Allocatable
   │  (Capacity minus kube-reserved/system-reserved/eviction-thresholds)
   ▼
Requests (sum of all scheduled Pods)
   │  (what the SCHEDULER bin-packs against — Resource Requests guide)
   ▼
Limits (sum of all scheduled Pods' limits — may exceed Allocatable: overcommitment)
   │  (what the KERNEL enforces at runtime — Resource Limits guide)
   ▼
Actual Usage (live, real-time consumption — what kubectl top shows)
   │
   ▼
QoS Class (Guaranteed / Burstable / BestEffort — derived from requests vs limits)
   │  (determines eviction PRIORITY, not eviction TRIGGER)
   ▼
Eviction (triggered by NODE PRESSURE — a separate axis entirely from QoS)
```

**The single insight this entire guide is built around:** these are **two orthogonal systems**, not one continuous pipeline. The scheduler-and-limits chain (Capacity → Allocatable → Requests → Limits → Usage) governs *whether a Pod gets placed and what it's allowed to consume*. QoS and eviction govern an entirely separate question: *when a node is actually running out of a resource right now, who gets sacrificed first*. QoS class doesn't cause eviction — **node pressure** does. QoS only decides the *order*.

---

# PART 1 — RECAP: THE FIRST FOUR LINKS (BRIEF — SEE THE DEDICATED GUIDES FOR FULL DEPTH)

## Node Capacity → Allocatable
```bash
kubectl describe node <node>
```
```
Capacity:
  cpu:                4
  memory:             16Gi
  ephemeral-storage:  100Gi
  pods:               110
Allocatable:
  cpu:                3800m
  memory:             15Gi
  ephemeral-storage:  95Gi
  pods:               110
```
`Allocatable = Capacity − kube-reserved − system-reserved − eviction-thresholds` (the last term is new territory this guide covers in Part 3 — it's not just "reserved for the OS," it's specifically held back so the eviction manager has room to react *before* the node completely runs out).

## Requests → the scheduler's bin-packing
Already covered exhaustively in the Resource Requests guide — the scheduler only ever compares a new Pod's requests against `Allocatable minus already-committed requests`, never live usage.

## Limits → kernel enforcement
Already covered exhaustively in the Resource Limits guide — CFS quota/period for CPU (throttling), cgroup `memory.max` for memory (OOMKill).

## Ephemeral storage — the resource type neither guide covered

Ephemeral storage is a container's **writable layer, logs, and `emptyDir` volumes without a medium override** — anything not backed by a persistent volume. It follows the *exact same* requests/limits mechanics as CPU/memory:

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "2Gi"
```
**The critical difference from CPU/memory:** there's no per-container cgroup enforcement as clean as memory's `memory.max` for this — the kubelet **periodically measures** actual disk usage (container writable layer + logs + `emptyDir`) and compares it against the limit, evicting the **Pod** if exceeded. This is slower and coarser than memory's near-instant cgroup-triggered OOMKill — a burst that fills disk faster than the kubelet's measurement interval can still cause real node-level disk pressure before Kubernetes reacts.

```bash
kubectl exec -it <pod> -- df -h /       # a rough proxy view from inside the container
kubectl describe node <node> | grep -A3 ephemeral-storage
```

---

# PART 2 — QoS CLASSES, RESTATED AS PART OF THE CHAIN

*(Full mechanics already in the Resource Limits guide — restated here specifically in terms of where they sit in the overall chain.)*

```
QoS class is a PURE FUNCTION of requests vs limits, computed ONCE at Pod
admission, and never changes for that Pod's lifetime:

  Guaranteed:  requests == limits, on EVERY resource, EVERY container
  Burstable:   at least one request set, but requests != limits somewhere
  BestEffort:  no requests or limits at all, anywhere
```
**QoS is not itself a mechanism that does anything — it's a label the eviction manager (Part 4) reads when it needs to decide who to kill.** This is the piece worth internalizing before Part 4: computing your Pod's QoS class tells you nothing about whether eviction will *happen* — only, if it does happen on this node, roughly where you'll be in the kill order.

---

# PART 3 — NODE PRESSURE CONDITIONS

This is the actual trigger mechanism for eviction — three independent conditions, each monitored separately by the kubelet:

## Memory pressure
```bash
kubectl describe node <node> | grep MemoryPressure
# MemoryPressure   False   ...   KubeletHasSufficientMemory
```
Triggered when available memory drops below a configurable **eviction threshold** (default `memory.available<100Mi`, though production clusters commonly tune this higher). This is checked against **actual available memory on the node**, not against any Pod's individual request/limit — it's a node-wide signal.

## Disk pressure
```bash
kubectl describe node <node> | grep DiskPressure
```
Triggered by available root filesystem or image filesystem space dropping below threshold (default `nodefs.available<10%`, `imagefs.available<15%`) — commonly caused by log accumulation, orphaned container images, or exactly the ephemeral-storage overconsumption from Part 1.

## PID pressure
```bash
kubectl describe node <node> | grep PIDPressure
```
Triggered when the node is running low on available process IDs — a less commonly hit but real production issue with workloads that fork excessively (runaway process trees, misbehaving applications leaking zombie processes) — a node can have plenty of free CPU/memory and still become unschedulable/unstable purely from PID exhaustion.

## Numerical example: memory pressure threshold in action
```
Node total memory:        16Gi
eviction-hard threshold:  memory.available<500Mi   (configured)
Current available:        480Mi
                              │
                              ▼
                    MemoryPressure condition → True
                              │
                              ▼
                    kubelet's eviction manager activates
```

---

# PART 4 — THE EVICTION MANAGER: HOW IT ACTUALLY DECIDES WHO DIES

## The algorithm, precisely

When a pressure condition is active, the kubelet's eviction manager ranks **all Pods on that node** using this exact priority order (highest eviction priority = killed first):

```
1. Does the Pod's usage of the PRESSURED resource exceed its own REQUEST?
   (e.g., under memory pressure: is this Pod using more memory than it requested?)
   → Pods exceeding their request are evicted BEFORE Pods that are within it,
     REGARDLESS of QoS class — this is the step most people forget entirely
2. Among Pods in the same "exceeds request" bucket, rank by QoS:
     BestEffort  → evicted first
     Burstable   → evicted next (worst offender relative to its own request first)
     Guaranteed  → evicted last, and only if it's ALSO exceeding its own request
                   (a true Guaranteed Pod, request==limit, technically can't
                    exceed its request without also hitting its limit and
                    being OOMKilled by the kernel directly, first)
3. Among equally-ranked candidates, prefer evicting the one that frees the
   MOST of the pressured resource (biggest offender first)
```

## Numerical walkthrough
```
Node under MemoryPressure. Three Pods:

Pod A: BestEffort,  using 50Mi   (no request declared, "using more than
                                  request" is vacuously true/not applicable —
                                  treated as maximum eviction priority)
Pod B: Burstable,   request 200Mi, using 500Mi  (exceeds request by 300Mi)
Pod C: Guaranteed,  request 500Mi=limit 500Mi, using 480Mi (within request)

Eviction order: Pod A first (BestEffort, no protections at all)
                Pod B second (Burstable, clearly exceeding its own request)
                Pod C protected (Guaranteed AND within its declared request)

If pressure persists after A and B are evicted and memory is STILL
critically low, Pod C is now the only candidate left and WOULD be
evicted too — QoS guarantees relative priority, never absolute immunity.
```

## Graceful vs immediate eviction
Pod eviction due to node pressure is **not** a SIGKILL by default — the kubelet attempts a graceful termination (the same `SIGTERM` → grace period → `SIGKILL` sequence as any Pod deletion, per the Pods masterclass), **unless** the pressure is severe enough to require immediate reclamation, in which case the kubelet may skip straight to forceful termination to relieve pressure fast enough to keep the node itself alive.

```bash
kubectl get events --field-selector reason=Evicted
kubectl describe pod <evicted-pod>
# Status: Failed
# Reason: Evicted
# Message: "The node was low on resource: memory. ..."
```

## Eviction vs OOMKill — the distinction that trips almost everyone up

| | Container OOMKill | Pod Eviction |
|---|---|---|
| Trigger | THIS container's own memory.max exceeded | The NODE overall is under pressure |
| Scope | One container, inside its own cgroup | Kubelet-level decision across ALL Pods on the node |
| Who decides | The kernel, instantly, automatically | The kubelet's eviction manager, per the ranked algorithm above |
| Outcome | Container restarted in place (same Pod, RESTARTS++) | Whole Pod terminated and, if managed by a controller, rescheduled elsewhere (Pods masterclass Diagram D1 vs D2 distinction, applied here) |

**A Pod can be perfectly within its own memory limit and still get evicted** — because eviction is a node-wide, cross-Pod decision responding to overall pressure, completely independent of whether any single container individually crossed its own cgroup ceiling. This is the most commonly confused pair of concepts in all of Kubernetes resource management.

---

# PART 5 — LimitRange AND ResourceQuota IN THE CHAIN

*(Full mechanics in the Resource Limits guide — restated here specifically as the two admission-time gates that sit BEFORE any of this runtime behavior even becomes possible.)*

```
LimitRange (namespace-level defaults/bounds, admission-time)
        │
        ▼
ResourceQuota (namespace-level aggregate cap, admission-time)
        │
        ▼
Scheduler (per-node fit check, using Requests)
        │
        ▼
Kernel enforcement (Limits, at runtime)
        │
        ▼
Eviction manager (node pressure, at runtime, ONLY if things go wrong)
```
**Practical implication for an SRE:** LimitRange/ResourceQuota are your *prevention* layer — catching misconfigured or absent requests/limits before a Pod is ever scheduled. The eviction manager is your *last resort* layer — reacting to a node that's already in trouble despite everything upstream having been "correct" at admission time (e.g., every Pod's declared limits were reasonable, but actual aggregate usage still spiked beyond what the node can sustain simultaneously — the classic overcommitment risk from the Resource Requests guide, realized).

---

# PART 6 — PRODUCTION SCENARIOS

## Scenario: "Nodes randomly evict unrelated Pods during traffic spikes"
```
Root cause pattern: heavy overcommitment (sum of LIMITS far exceeds node
capacity) + a traffic spike causing MANY Burstable Pods to simultaneously
burst toward their limits at once → real memory usage collectively exceeds
the node's actual capacity → MemoryPressure → eviction manager sacrifices
the least-protected Pods, which may have nothing to do with the actual
traffic spike's root cause
```
Diagnosis: `kubectl top nodes` during the incident window, cross-referenced with `kubectl describe node` events for `MemoryPressure`/`Evicted` timestamps, then check whether the evicted Pods' actual usage was near/over their own requests at the time (Part 4's ranking algorithm) — this tells you whether they were "innocent bystanders" (evicted purely for being BestEffort/Burstable-over-request) versus actual contributors to the pressure.

## Scenario: "Disk fills up slowly over weeks, eventually causing DiskPressure evictions"
Classic ephemeral-storage-without-limits pattern (Part 1) — logs or `emptyDir` usage growing unbounded with no `ephemeral-storage` limit set anywhere, invisible to `kubectl top pod` (which reports CPU/memory, not disk, by default) until the node's DiskPressure threshold is finally crossed, at which point eviction targets Pods somewhat arbitrarily based on which ones are using the most disk, not necessarily the actual root-cause offender.

## Scenario: "One node keeps flapping Ready/NotReady under load, unrelated to memory or disk"
Check `PIDPressure` specifically — a workload with a process-forking bug or a misconfigured process supervisor can exhaust the node's PID space while CPU/memory/disk all look completely normal in every standard dashboard, since most monitoring setups don't surface PID count prominently.

---

# PART 7 — kubectl COMMANDS FOR MONITORING AND DIAGNOSIS

```bash
# Node-level view: capacity, allocatable, and CURRENT allocation percentage
kubectl describe node <node> | grep -A10 "Allocated resources"

# Node conditions — the actual pressure signals
kubectl describe node <node> | grep -E "MemoryPressure|DiskPressure|PIDPressure"
kubectl get nodes -o custom-columns=NAME:.metadata.name,MEM:.status.conditions[3].status

# Live usage (requires metrics-server)
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory
kubectl top pods --all-namespaces --sort-by=cpu

# Evicted Pod history
kubectl get events --all-namespaces --field-selector reason=Evicted
kubectl get pods --all-namespaces --field-selector status.phase=Failed

# Per-Pod QoS and resource declarations at a glance
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].resources}'

# ResourceQuota / LimitRange current state
kubectl describe resourcequota -n <namespace>
kubectl describe limitrange -n <namespace>

# Direct cgroup inspection on a node (last resort, most precise)
kubectl exec -it <pod> -- cat /sys/fs/cgroup/memory.current
kubectl exec -it <pod> -- cat /sys/fs/cgroup/memory.max
```

---

# PART 8 — DIAGNOSTIC DECISION TREE

```
Pod terminated unexpectedly — WHY?
        │
        ▼
Check: kubectl describe pod <pod> → Last State / Events
        │
        ├── "Reason: OOMKilled", exit code 137
        │      → THIS container exceeded ITS OWN memory limit
        │      → fix: raise the limit, or fix a leak, per Resource Limits guide
        │
        ├── "Reason: Evicted"
        │      → the NODE was under pressure; check WHICH pressure
        │           (Memory/Disk/PID) from the event message
        │      → fix: reduce overcommitment, add node capacity, or
        │           fix the actual root-cause consumer, per Part 6
        │
        ├── Pod stuck "Pending"
        │      → check node Allocatable vs sum of requests
        │           (Resource Requests guide) — NOT a limits/eviction issue
        │           at all; this is a pre-scheduling problem
        │
        └── "CreateContainerError"/"FailedMount"
               → unrelated to resource pressure entirely — check volumes/
                    config (ConfigMaps/Secrets guide, Pods masterclass)
```

---

# PART 9 — HANDS-ON MINIKUBE LABS

### Lab 1: Observe node Allocatable vs Capacity
```bash
minikube start --memory=2000 --cpus=2
kubectl describe node minikube | grep -A6 "Capacity:\|Allocatable:"
```

### Lab 2: Induce memory pressure deliberately
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: memory-hog }
spec:
  containers:
    - name: hog
      image: polinux/stress
      resources:
        requests: { memory: "50Mi" }
        limits: { memory: "1500Mi" }     # deliberately huge relative to node size
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "1400M", "--vm-hang", "60"]
EOF
kubectl describe node minikube | grep MemoryPressure
kubectl get events --field-selector reason=Evicted -w
```

### Lab 3: Watch QoS-based eviction ordering
```bash
# create one BestEffort, one Burstable, one Guaranteed Pod, then repeat
# the memory-pressure stress test above and observe eviction ORDER
kubectl run besteffort --image=nginx
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: burstable }
spec:
  containers:
    - name: web
      image: nginx
      resources: { requests: { memory: "64Mi" }, limits: { memory: "256Mi" } }
EOF
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: guaranteed }
spec:
  containers:
    - name: web
      image: nginx
      resources: { requests: { memory: "128Mi", cpu: "100m" }, limits: { memory: "128Mi", cpu: "100m" } }
EOF
kubectl get pod besteffort burstable guaranteed -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.qosClass}{"\n"}{end}'
# then re-run the stress Pod from Lab 2 and watch "besteffort" get
# evicted long before "guaranteed"
```

### Lab 4: Ephemeral storage limit
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: disk-hog }
spec:
  containers:
    - name: hog
      image: busybox
      resources:
        limits: { ephemeral-storage: "50Mi" }
      command: ["sh", "-c", "dd if=/dev/zero of=/tmp/bigfile bs=1M count=100; sleep 3600"]
EOF
kubectl get pod disk-hog -w
kubectl describe pod disk-hog
# → eventually Evicted for exceeding its ephemeral-storage limit
```

---

# 20 INTERVIEW QUESTIONS

**The chain**
1. List the full chain from Node Capacity to Eviction, in order.
2. Why is "Allocatable" less than "Capacity," and what specifically is subtracted?
3. What's the relationship (or lack thereof) between a Pod's QoS class and whether eviction happens at all?

**Ephemeral storage**
4. How is ephemeral storage enforcement different from memory limit enforcement?
5. Why can a disk-filling bug go undetected by `kubectl top pod` until DiskPressure hits?

**Node pressure**
6. Name the three node pressure conditions and what each is triggered by.
7. Can a node have plenty of free CPU and memory and still be in trouble? Explain with an example.

**Eviction mechanics**
8. Walk through the eviction manager's ranking algorithm, step by step.
9. Why does "exceeds its own request" outrank QoS class in the eviction order?
10. Can a Guaranteed Pod ever be evicted? Under what circumstance?
11. What's the difference between a graceful and an immediate eviction?

**OOMKill vs Eviction**
12. What's the fundamental scope difference between an OOMKill and a Pod eviction?
13. Can a Pod be evicted while every one of its containers is within its own memory limit? Why?

**Production**
14. Describe a realistic production incident caused by overcommitment interacting with node pressure.
15. Why might LimitRange/ResourceQuota being "correctly" configured still not prevent a pressure-driven eviction incident?
16. What monitoring signal would catch a slow ephemeral-storage leak before it causes an outage?

**Diagnosis**
17. A Pod shows `Status: Failed, Reason: Evicted` — what's your investigation path?
18. How do you distinguish a scheduling problem (Pending) from an eviction problem (Evicted) at a glance?
19. What command reveals a node's current pressure conditions directly?
20. Why might two engineers debugging the "same" incident — one looking at `kubectl top`, one looking at `kubectl describe node` events — reach different conclusions about root cause?

---

*This guide unifies Kubernetes resource management into one chain — Capacity → Allocatable → Requests → Limits → Usage → QoS → Eviction — with the node-pressure and eviction-manager mechanics that neither the Resource Requests nor Resource Limits guides covered in depth, completing the full picture from scheduling admission through last-resort node self-preservation.*
