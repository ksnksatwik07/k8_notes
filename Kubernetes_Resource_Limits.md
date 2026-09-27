# Kubernetes Resource Limits — Complete Mastery Guide
### From Absolute Beginner to Production Resource-Enforcement Internals

---

# PART 1 — REQUESTS VS LIMITS

## The one-sentence distinction

**A request is what the scheduler reserves for you. A limit is the hard ceiling the kernel enforces against you.** Requests answer "where can this Pod run?" Limits answer "what happens when this container tries to use more than it's allowed?"

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```
- Request: guaranteed floor, used only at scheduling time (see the Resource Requests guide for full scheduler mechanics)
- Limit: enforced ceiling, checked continuously at runtime by the kernel via **cgroups** (control groups) — the same Linux primitive Docker and every other container runtime is built on

**Critically: these are enforced completely differently for CPU vs memory**, because of the same compressible/incompressible distinction from the Requests guide. This guide is about exactly how each enforcement actually works, mechanically, inside the kernel.

---

# PART 2 — CPU LIMITS AND THROTTLING

## How CPU limiting actually works (CFS internals)

A CPU limit is implemented via the Linux kernel's **Completely Fair Scheduler (CFS) bandwidth control**. When you set:
```yaml
resources:
  limits:
    cpu: "500m"
```
The kubelet translates this into two cgroup values:
```
cfs_quota_us  = 50000     # microseconds of CPU time allowed per period
cfs_period_us = 100000    # the period length (default 100ms)
```
This means: **in every rolling 100ms window, this container may consume at most 50ms of total CPU time** — across however many cores it actually touches. If it tries to use more, the kernel simply **stops scheduling it** for the remainder of that period — the process isn't killed, it's just paused, then resumes at the start of the next period.

```
Period (100ms):  [0ms────────────────────────────────100ms]
Container's usage:  [████████████████████]░░░░░░░░░░░░░░░░
                     ↑ used its full 50ms quota by the 50ms mark
                                          ↑ THROTTLED for the remaining 50ms
                                            — no CPU time granted, process paused
Next period begins → quota resets to 50ms, process resumes
```

## Numerical example

```
Container limit: 500m CPU (50ms per 100ms period)
Container actually wants: 800m worth of continuous work

Result: it gets 50ms of real work done, then is throttled for 50ms,
        every single 100ms window — from the app's perspective, its
        effective throughput caps at 500m no matter how much work
        is queued up, and every period has a visible "stall."
```

This is precisely why CPU-bound applications under a tight limit exhibit **periodic latency spikes** even though `kubectl top pod` might show average usage well under the limit — averages hide the fact that usage is bursty and getting clipped every period.

```bash
kubectl exec -it <pod> -- cat /sys/fs/cgroup/cpu.stat
# nr_throttled: 452       ← number of periods this container was throttled
# throttled_usec: 8342198 ← total microseconds spent throttled
```
A high or climbing `nr_throttled` is the single most direct signal that a CPU limit is actively hurting an application's real-world latency, even when `kubectl top` "looks fine" on average.

---

# PART 3 — MEMORY LIMITS AND OOMKILL

## How memory limiting actually works (cgroup memory controller)

Memory is fundamentally different: there's no way to "throttle" a memory allocation the way you can pause CPU scheduling. A memory limit is enforced as a **hard ceiling** via the cgroup memory controller:
```yaml
resources:
  limits:
    memory: "512Mi"
```
This becomes `memory.max` (or `memory.limit_in_bytes` on cgroup v1) — `512Mi` in bytes — in the container's cgroup.

## What happens when a container exceeds its memory limit

1. The container's memory usage (including page cache attributed to it, in most configurations) approaches the cgroup's `memory.max`
2. The kernel first tries to reclaim what it can (evict clean page cache, etc.) — this happens silently and doesn't kill anything
3. If the container's usage still can't be brought under the limit — i.e., it genuinely needs more anonymous (non-reclaimable) memory than allowed — the kernel's **OOM killer** is invoked, but scoped to *that container's own cgroup*, not the whole node
4. The OOM killer selects a process inside that cgroup (usually the largest resident-memory consumer, weighted by an `oom_score_adj` value) and sends it `SIGKILL` — immediate, unrecoverable termination, no graceful shutdown, no chance to flush buffers or close connections cleanly
5. The container exits; the kubelet observes the container exit and marks the Pod's container status:
```bash
kubectl describe pod <pod>
```
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```
- **Exit code 137** = `128 + 9`, where `9` is `SIGKILL`'s signal number — this exact exit code is a memory-limit tell, distinct from a normal application crash

```
Container memory usage over time, limit = 512Mi:

512Mi ┤                                    ✕ ← SIGKILL fired here, instantly
      │                              ╱╱╱╱
      │                        ╱╱╱╱╱
256Mi │              ╱╱╱╱╱╱╱╱╱
      │      ╱╱╱╱╱╱╱╱
   0  └──────────────────────────────────────▶ time
      startup    steady growth (leak, or genuine
                  spike under load) crosses the ceiling
```

There is **no warning period, no graceful degradation** for memory the way there is for CPU throttling — this is the single most important asymmetry to understand. A container living right at the edge of its memory limit is one allocation spike away from an unceremonious kill.

## Node-level OOM vs container-level OOM

If a container has **no memory limit set at all**, it can consume memory up to the **node's** total available memory — at which point the **node-level** OOM killer activates instead, which is far more disruptive: it evaluates *every* process on the node (not just one container) and may kill an entirely unrelated Pod's process, including critical system daemons in the worst case, based on the node's own OOM scoring. This is exactly why unlimited-memory containers are dangerous in shared multi-tenant clusters — one leaking Pod with no limit can take down neighbors that had nothing to do with the problem.

---

# PART 4 — QoS CLASSES (RECAP + LIMIT-SPECIFIC DETAIL)

QoS class is determined entirely by the **relationship between requests and limits** — covered fully in the Resource Requests guide, but the limit-specific angle matters here:

## Guaranteed
`requests == limits`, for **both** CPU and memory, on **every** container in the Pod.
```yaml
resources:
  requests: { cpu: "500m", memory: "512Mi" }
  limits:   { cpu: "500m", memory: "512Mi" }
```
Because request equals limit exactly, this Pod is scheduled onto a node with the *exact* amount of resources it will ever be allowed to use — no bursting above what was reserved, no risk of being surprised by its own growth. Last to be OOM-killed under node pressure.

## Burstable
Requests set, but lower than limits (or a limit is omitted for one resource).
```yaml
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits:   { cpu: "1000m", memory: "1Gi" }
```
Can legitimately use up to 4x its requested CPU and memory when the node has spare capacity — this is the deliberate overcommitment pattern from the Requests guide. Evicted before Guaranteed, in rough proportion to how far usage exceeds its own request.

## BestEffort
No requests or limits at all.
```yaml
resources: {}
```
Can use as much as the node has spare — and is killed first, without hesitation, the instant the node needs that memory back for anyone else.

```
Node under memory pressure — kubelet eviction priority (first killed → last killed):
  BestEffort  →  Burstable (worst offender relative to its own request first)  →  Guaranteed
```

---

# PART 5 — LIMITRANGE

A **LimitRange** is a namespace-level policy object that sets **defaults** and **min/max bounds** for requests/limits on containers/Pods that don't specify their own — this is how platform teams prevent both "forgot to set anything" (BestEffort by accident) and "asked for way too much" mistakes cluster-wide.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"          # applied automatically if a container omits `limits.cpu`
        memory: "512Mi"
      defaultRequest:
        cpu: "250m"          # applied automatically if a container omits `requests.cpu`
        memory: "256Mi"
      max:
        cpu: "2"             # no container in this namespace may request/limit above this
        memory: "2Gi"
      min:
        cpu: "100m"          # no container may go below this
        memory: "128Mi"
      maxLimitRequestRatio:
        cpu: "4"             # limit can be at most 4x the request (bounds burst headroom)
```
**Line-by-line:**
- `type: Container` — this LimitRange applies per-container (a `type: Pod` block can additionally bound Pod-level totals)
- `default`/`defaultRequest` — auto-injected onto any container manifest in this namespace that doesn't specify its own values — this is the mechanism that prevents accidental BestEffort Pods namespace-wide
- `max`/`min` — hard validation bounds; a Pod manifest requesting `cpu: 4` in a namespace capped at `max: 2` is **rejected outright at admission time**, not silently clamped
- `maxLimitRequestRatio` — caps how aggressively a workload can overcommit itself (e.g., prevents someone from requesting `10m` but setting a limit of `8000m`, which would look tiny to the scheduler but be a massive noisy-neighbor risk in practice)

```bash
kubectl describe limitrange default-limits -n production
```

---

# PART 6 — RESOURCEQUOTA

While LimitRange governs individual containers/Pods, **ResourceQuota** caps the **total** resource consumption across an entire namespace — the aggregate ceiling a team/tenant cannot exceed regardless of how many Pods they create.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"                       # even a raw object COUNT can be capped
    persistentvolumeclaims: "10"
```
**Line-by-line:**
- `requests.cpu` / `requests.memory` — sum across every Pod in the namespace cannot exceed these; a new Pod that would push the namespace over this line is **rejected at admission**, not queued or partially applied
- `limits.cpu` / `limits.memory` — same idea, applied to the sum of limits instead
- `pods: "50"` — a hard cap on the raw Pod count, independent of their resource footprint — useful for preventing namespace sprawl even with tiny Pods
- `persistentvolumeclaims` — quotas aren't CPU/memory-only; storage and other countable objects can be capped too

**Crucial interaction: once a ResourceQuota exists in a namespace covering `requests.cpu`/`requests.memory` (or `limits.*`), every Pod created in that namespace MUST explicitly specify requests/limits for the covered resources** — Kubernetes cannot enforce a numeric quota against Pods that don't declare a number. This is one of the most common "why is my Pod suddenly rejected" surprises after a platform team introduces quotas.

```bash
kubectl describe resourcequota team-quota -n production
```
```
Resource          Used   Hard
----------------  -----  ----
requests.cpu      14     20
requests.memory   28Gi   40Gi
pods              37     50
```

---

# PART 7 — DIAGRAM: LIMITRANGE + RESOURCEQUOTA + SCHEDULER TOGETHER

```
             Namespace "production"
┌─────────────────────────────────────────────────────────────────┐
│  ResourceQuota: requests.cpu ≤ 20, requests.memory ≤ 40Gi        │
│  LimitRange: default 500m/512Mi, max 2/2Gi per container         │
│                                                                    │
│  New Pod manifest submitted, omits resources{}                    │
│         │                                                          │
│         ▼                                                          │
│  1. LimitRange admission controller INJECTS defaults               │
│     (requests: 250m/256Mi, limits: 500m/512Mi)                     │
│         │                                                          │
│         ▼                                                          │
│  2. ResourceQuota admission controller checks running total        │
│     Used so far: 14 CPU requested → +250m = 14.25 ≤ 20 ✓           │
│         │                                                          │
│         ▼                                                          │
│  3. Pod is admitted, now visible to the SCHEDULER                  │
│         │                                                          │
│         ▼                                                          │
│  4. Scheduler bin-packs against node Allocatable (separate check,  │
│     covered in the Resource Requests guide) — per-NODE, not        │
│     per-namespace                                                   │
└─────────────────────────────────────────────────────────────────┘
```
Note these are **three separate, sequential gates**: LimitRange (defaulting/bounds) → ResourceQuota (namespace aggregate) → Scheduler (per-node fit). A Pod can pass all namespace-level checks and still end up `Pending` if no individual node has room — namespace quota and node capacity are entirely independent constraints.

---

# PART 8 — TROUBLESHOOTING COMMANDS

```bash
# Confirm a Pod's actual QoS class
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'

# Check for OOMKilled history
kubectl describe pod <pod> | grep -A5 "Last State"
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'

# Live resource usage vs configured requests/limits
kubectl top pod <pod> --containers
kubectl describe pod <pod> | grep -A2 "Limits\|Requests"

# CPU throttling evidence, from inside the container (cgroup v2)
kubectl exec -it <pod> -- cat /sys/fs/cgroup/cpu.stat
# cgroup v1 equivalent:
kubectl exec -it <pod> -- cat /sys/fs/cgroup/cpu,cpuacct/cpu.stat

# Memory usage vs cgroup limit, from inside the container
kubectl exec -it <pod> -- cat /sys/fs/cgroup/memory.current    # cgroup v2
kubectl exec -it <pod> -- cat /sys/fs/cgroup/memory.max
# cgroup v1:
kubectl exec -it <pod> -- cat /sys/fs/cgroup/memory/memory.usage_in_bytes
kubectl exec -it <pod> -- cat /sys/fs/cgroup/memory/memory.limit_in_bytes

# LimitRange / ResourceQuota state
kubectl describe limitrange -n <namespace>
kubectl describe resourcequota -n <namespace>

# Node-wide allocation view
kubectl describe node <node> | grep -A10 "Allocated resources"

# Cluster-wide OOMKill audit (via events)
kubectl get events -A --field-selector reason=OOMKilling
```

---

# PART 9 — PRODUCTION BEST PRACTICES

1. **Always set both requests and limits** — never ship a BestEffort Pod to production intentionally; enforce this with a namespace-wide LimitRange as a safety net, not just team discipline.
2. **Size memory limits with real headroom above observed peak usage**, not average usage — memory has zero tolerance for exceeding the limit (instant SIGKILL), unlike CPU's graceful throttling.
3. **Prefer Guaranteed QoS for latency-sensitive or stateful workloads** (databases, control-plane-adjacent services) where throttling or eviction is unacceptable.
4. **Be deliberate about CPU limits on latency-sensitive services** — a tight CPU limit can introduce periodic throttling stalls invisible in averaged dashboards; consider omitting CPU limits (while still setting a request) for latency-critical paths, accepting the overcommitment trade-off instead.
5. **Watch `nr_throttled`/`throttled_usec`, not just `kubectl top`**, for any service with real latency SLOs — average CPU usage hides throttling entirely.
6. **Use ResourceQuota per namespace/team** so one team's runaway workload growth can't silently starve out capacity from every other team sharing the cluster.
7. **Pair ResourceQuota with LimitRange defaults** — a quota alone will start rejecting Pods that don't declare resources at all, so ship the default-injection safety net at the same time you introduce quotas.
8. **Set `maxLimitRequestRatio` deliberately** rather than leaving burst headroom unbounded — an extreme ratio (request 10m, limit 8) looks tiny to the scheduler while being a genuine noisy-neighbor risk once it actually bursts.
9. **Monitor OOMKilled events cluster-wide as a leading indicator**, not just a per-incident annoyance — a rising OOMKill rate across many unrelated Pods often signals systemic under-provisioning of memory limits, not "many separate app bugs."
10. **Treat exit code 137 as diagnostic gold** — it immediately narrows the investigation to memory-limit-related SIGKILL, versus other application crash codes.

---

# PART 10 — INTERVIEW QUESTIONS

**Fundamentals**
1. In one sentence each, what problem does a request solve vs what problem does a limit solve?
2. Why can CPU be "throttled" but memory cannot?

**CPU internals**
3. Explain, mechanically, how a `500m` CPU limit is enforced via CFS quota/period.
4. What does a high `nr_throttled` value tell you that `kubectl top pod`'s average usage number does not?
5. Why might a CPU-bound service show fine average CPU usage but still have periodic latency spikes?

**Memory internals**
6. Walk through exactly what happens, step by step, when a container's memory usage crosses its limit.
7. Why is exit code 137 specifically meaningful, and what does it decompose into?
8. What's the difference between a container-scoped OOM kill and a node-level OOM kill, and why is the latter more dangerous?

**QoS**
9. Why does `requests == limits` produce the Guaranteed QoS class specifically?
10. In what order does the kubelet evict Pods under memory pressure, and why does that order make sense given each class's guarantees?

**LimitRange / ResourceQuota**
11. What's the difference in scope between LimitRange and ResourceQuota?
12. What happens if you apply a ResourceQuota to a namespace where existing Pod manifests don't specify resource requests?
13. What does `maxLimitRequestRatio` protect against?

**Troubleshooting (scenario-based)**
14. A Pod keeps restarting with exit code 137 — what's your investigation process?
15. A service has fine average CPU usage in dashboards but users report intermittent slow responses — what do you check?
16. A team's new Pod is rejected at creation with no scheduling event at all — what's likely happening, and how is that different from a Pod stuck `Pending`?

---

*This guide covers Kubernetes resource limits from CFS-based CPU throttling and cgroup-based OOMKill internals through QoS-driven eviction, LimitRange/ResourceQuota governance, and production troubleshooting — the complete arc from beginner to resource-enforcement internals expert.*
