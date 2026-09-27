# Kubernetes Architecture — Complete Masterclass
## Part 5 of 6: Pod Restart, Node Failure & Rollout Diagrams, etcd Internals, and Failure-Scenario Analysis

*(Continuing from Parts 1–4. This part delivers Sequence Diagrams D–F, then covers topics 30–32 in full depth: etcd storage/consistency, inter-component networking, and every major failure scenario with concrete timelines.)*

---

# SEQUENCE DIAGRAM D: POD RESTART

There are two entirely different kinds of "Pod restart," and conflating them is a common source of confusion.

## D1: Container restart within the SAME Pod (kubelet-local, no scheduler involved)

```
kubelet                  Container Runtime              apiserver
   │                            │                            │
   │─runs liveness probe───────▶│                            │
   │◀──probe FAILS──────────────│                            │
   │  (kubelet decides locally, │                            │
   │   NO round-trip to the      │                            │
   │   control plane needed       │                            │
   │   for this decision)          │                            │
   │─stop container────────────▶│                            │
   │─start container (same Pod)▶│                            │
   │◀──running───────────────────│                            │
   │─report restartCount++──────┼───────────────────────────▶│
```
**Step-by-step:** the kubelet runs the configured liveness probe itself, on its own schedule, entirely locally. If it fails past the configured threshold, the kubelet — **without asking the API server for permission** — restarts the container in place, inside the *same* Pod object (same IP, same UID, same everything except the container process itself). Only *after* acting does it report the updated `restartCount` back through the API server. This is exactly why basic self-healing keeps working even during an API server outage (Part 6).

```bash
kubectl get pod my-pod
# RESTARTS column increments — same Pod, same age, same IP
```

## D2: Pod replacement (ReplicaSet-driven, full scheduling cycle)

This happens when the **Pod itself** is gone (not just one unhealthy container) — e.g., `kubectl delete pod`, an OOM kill severe enough to take the whole Pod down, or eviction.

```
ReplicaSet-Ctrl        apiserver          Scheduler          kubelet(new node)     Runtime
      │                    │                   │                     │                │
      │◀──Pod DELETED event┤                   │                     │                │
      │─count: 2, want 3───┤                   │                     │                │
      │─create NEW Pod────▶│                   │                     │                │
      │                    │─notify───────────▶│                     │                │
      │                    │                   │─bind to a node─────▶│                │
      │                    │◀──write binding───┤                     │                │
      │                    │─notify─────────────┼────────────────────▶│                │
      │                    │                   │                     │─pull+run──────▶│
      │                    │                   │                     │◀──running──────│
      │                    │◀──status: Running─┼─────────────────────┤                │
```
**Key distinction from D1:** this is a **brand-new Pod object** — new UID, new IP, possibly a completely different node — not the same Pod with a restarted container. `kubectl get pods` shows a Pod with `AGE: 2s` and `RESTARTS: 0`, not an incrementing restart count on the old one, because the old Pod object is simply gone.

## Beginner mental model for telling D1 vs D2 apart
Check `RESTARTS` and `AGE` together: a climbing `RESTARTS` count with unchanged `AGE`/`NAME` is D1 (same Pod, kubelet-local). A brand-new `NAME` with `AGE: 0s`/`RESTARTS: 0` replacing a Pod that's now gone entirely is D2 (full scheduling cycle, ReplicaSet-driven).

---

# SEQUENCE DIAGRAM E: NODE FAILURE

```
Node-3 (crashes/network-partitions — kubelet stops sending heartbeats)
   │
   ▼ (t=0s)
Node-Controller (in kube-controller-manager)      apiserver         etcd
   │                                                    │               │
   │─(waiting for heartbeat, --node-monitor-             │               │
   │  grace-period, default 40s)                          │               │
   │  ... 40s pass with NO heartbeat ...                    │               │
   │─t=40s: marks Node-3 condition Ready=Unknown──────────▶│               │
   │                                                    │─write────────▶│
   │  (Pods on Node-3 are NOT yet evicted — the node        │               │
   │   might just be a transient network blip)               │               │
   │                                                    │               │
   │─(waiting for --pod-eviction-timeout, default 5m        │               │
   │  total from when Ready became Unknown)                   │               │
   │  ... additional time passes ...                            │               │
   │─t=5m: begins evicting every Pod that was on Node-3────▶│               │
   │  (sets deletionTimestamp on each; SINCE the node is       │               │
   │   unreachable, there's no kubelet to actually confirm       │               │
   │   graceful termination — the control plane proceeds          │               │
   │   anyway, based on its own authority over the object)          │               │
   │                                                    │─write────────▶│
   │                                                    │─notify───────┼───────────▶│
                                                                    ┌──────────────┴──────────┐
                                                                    ▼                            
                                                  ReplicaSet controllers (for every
                                                  ReplicaSet that had Pods on Node-3)
                                                    │
                                                    │─sees actual < desired for each
                                                    │  affected ReplicaSet
                                                    │─creates replacement Pods
                                                    ▼
                                              Scheduler places them on HEALTHY nodes
                                              (normal scheduling cycle, per Diagram C)
```

## Step-by-step explanation
1. Node-3 stops sending heartbeats (crash, kernel panic, network partition — the control plane cannot distinguish these from each other at this stage)
2. After `--node-monitor-grace-period` (default **40s**) with no heartbeat, the **Node controller** marks the node's `Ready` condition as `Unknown`
3. The node is **not** immediately treated as dead — Pods keep showing as running on it, because a network blip that self-heals in a few seconds shouldn't trigger a full eviction-and-reschedule storm
4. After the total `--pod-eviction-timeout` (default **5 minutes**) has elapsed since the node stopped being `Ready`, the Node controller evicts every Pod that was scheduled on it
5. Each affected **ReplicaSet controller** independently notices its own actual-vs-desired gap and creates replacement Pods
6. The **scheduler** places these new Pods on healthy nodes through the completely ordinary scheduling cycle (Diagram C) — nothing node-failure-specific happens at the scheduler level; it just sees new unscheduled Pods, same as any other day

## Why the delay is deliberate, not a bug
Immediately evicting on the first missed heartbeat would cause **eviction storms** from routine, harmless network blips — the 40s + 5min defaults are a deliberate trade-off between "detect real failures reasonably fast" and "don't thrash the entire cluster over a two-second network hiccup." Production clusters sometimes tune these values down for faster failover, at the cost of being more trigger-happy about transient issues.

## Beginner misunderstanding worth flagging
People often expect Pods on a dead node to "move" to another node. **They don't move — they're evicted and entirely new Pod objects are created elsewhere.** Anything not captured in a PersistentVolume backed by network storage (local writes, in-memory state) is simply gone.

---

# SEQUENCE DIAGRAM F: DEPLOYMENT UPDATE (ROLLING UPDATE)

```
User: kubectl set image deployment/my-app app=myapp:2.0

Deployment-Ctrl              apiserver                RS-A (old, v1.0)    RS-B (new, v2.0)
      │                          │                            │                    │
      │◀──Deployment updated─────┤                             │                    │
      │  (.spec.template.spec.    │                             │                    │
      │   containers[0].image      │                             │                    │
      │   changed)                  │                             │                    │
      │                          │                            │                    │
      │─creates NEW ReplicaSet────┼──────────────────────────────────────────────▶│
      │  "RS-B" with the new       │                            │                    │
      │  Pod template (image v2.0) │                            │                    │
      │                          │                            │                    │
      │─ROLLING UPDATE LOOP (respecting maxSurge / maxUnavailable):                    │
      │    scale RS-B up by 1 ────┼────────────────────────────────────────────────▶│
      │                          │                            │                    │─Pod created,
      │                          │                            │                    │  scheduled,
      │                          │                            │                    │  started
      │    wait for new Pod Ready (readinessProbe passing) ─────────────────────────│
      │    scale RS-A down by 1───┼──────────────▶│                    │
      │                          │                            │─Pod terminated      │
      │  ... repeat until RS-A=0, RS-B=desired replica count ...                       │
      │                          │                            │                    │
      │  Rollout complete. RS-A kept (scaled to 0) for rollback history.               │
```

## Step-by-step explanation
1. You update the Deployment's Pod template (new image tag)
2. The **Deployment controller** notices the template hash changed and creates an entirely **new ReplicaSet** ("RS-B") — the old one ("RS-A") is never edited, only scaled
3. It then drives both ReplicaSets' replica counts incrementally, governed by two settings:
```yaml
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1          # how many EXTRA Pods (above desired count) are allowed during rollout
      maxUnavailable: 0    # how many Pods below desired count are tolerated during rollout
```
4. Each step: scale the new ReplicaSet up, **wait for the new Pod(s) to pass their readiness probe** (this is the gate — a Pod that's merely `Running` but not yet `Ready` does not count toward "safe to remove an old one"), then scale the old ReplicaSet down by the same amount
5. This repeats until the old ReplicaSet is at 0 and the new one is at full desired count
6. The **old ReplicaSet is kept, scaled to zero**, not deleted — this is exactly what powers `kubectl rollout undo`, which is simply "scale the previous ReplicaSet back up, scale the current one down" — the same rolling mechanism, run in reverse

## Real-world example
```bash
kubectl rollout status deployment/my-app       # watch the rollout live
kubectl rollout history deployment/my-app       # see every retained ReplicaSet revision
kubectl rollout undo deployment/my-app          # instant rollback — reuses the old ReplicaSet, no rebuild needed
```
This is why rollbacks are typically much faster than the original rollout — the old ReplicaSet's Pod template, and often its already-pulled images (if nodes cached them), are still right there; rolling back is "scale it back up," not "figure out what it used to be."

---

# 30. etcd DATA STORAGE AND CONSISTENCY

## Storage model, precisely
etcd stores every object as a flat key under a hierarchical path, with the **entire object serialized** (as JSON or Protobuf) as the value:
```
Key:   /registry/pods/default/my-pod
Value: {full Pod object: metadata, spec, status — everything}
```
There is no relational structure, no joins, no foreign keys — Kubernetes' API server is what imposes structure and relationships (via label selectors, owner references) on top of what is, underneath, an extremely simple key-value store.

## Consistency model: Raft
etcd guarantees **linearizable reads and writes** by default — every client sees a single, globally-agreed-upon order of operations, achieved via the **Raft consensus algorithm**:
```
        ┌─────────┐
        │  LEADER   │  ← all writes go through the leader
        │ (etcd-1)  │
        └────┬────┘
             │ replicates every write to a MAJORITY of followers
             │ before acknowledging it as committed
    ┌────────┴────────┐
    ▼                   ▼
┌─────────┐        ┌─────────┐
│ FOLLOWER │        │ FOLLOWER │
│ (etcd-2) │        │ (etcd-3) │
└─────────┘        └─────────┘
```
A write is only considered successful once it's been replicated to a **quorum** (a strict majority) of the etcd cluster — this is exactly why 3 nodes tolerate 1 failure (quorum = 2 of 3) and 5 tolerate 2 (quorum = 3 of 5), covered already in Part 2 §7.

## Watch mechanism, from etcd's side
etcd maintains a **revision history** (a monotonically increasing global counter for every change) — this is the actual mechanism underneath Kubernetes' `resourceVersion` (Part 3 §22): when a controller reconnects a dropped watch and says "resume from revision X," it's etcd's own change log making that replay possible.

## Compaction
etcd's revision history grows forever unless periodically **compacted** (old revisions discarded, keeping only the latest value plus a configurable retention window) — production etcd deployments configure auto-compaction; neglecting this is a classic cause of etcd's on-disk database file growing unboundedly, eventually threatening the `--quota-backend-bytes` limit and causing etcd to reject all writes until compacted and defragmented.

```bash
ETCDCTL_API=3 etcdctl compact <revision>
ETCDCTL_API=3 etcdctl defrag
```

---

# 31. KUBERNETES NETWORKING BETWEEN COMPONENTS (CONTROL-PLANE FOCUSED)

*(Pod/Service/CNI-level networking is covered exhaustively in the dedicated Kubernetes Networking guide — this section is scoped specifically to how control-plane components reach each other.)*

```
                    ┌─────────────────────────┐
                    │   Load Balancer (HA)      │  ← optional, fronts multiple
                    │                            │    API server replicas
                    └────────────┬──────────────┘
                                 │  HTTPS :6443
        ┌────────────────────────┼────────────────────────┐
        ▼                         ▼                         ▼
  ┌───────────┐            ┌───────────┐            ┌───────────┐
  │apiserver-1 │            │apiserver-2 │            │apiserver-3 │
  └─────┬─────┘            └─────┬─────┘            └─────┬─────┘
        │                         │                         │
        └────────────┬────────────┴────────────┬────────────┘
                     ▼                          ▼
              ┌──────────────────────────────────────┐
              │   etcd cluster (own client port :2379,  │
              │   peer port :2380 between etcd members)  │
              └──────────────────────────────────────┘

  Every worker node's kubelet:
      kubelet ──HTTPS :6443──▶ (the load balancer / any apiserver replica)

  kubelet also exposes its OWN API (port :10250) for:
      - kube-apiserver → kubelet: kubectl exec/logs/port-forward
        (this is a DIRECT connection, one of the few exceptions to
         "everything goes through the API server" — because for
         exec/logs, the API server is deliberately proxying a
         live stream TO a specific kubelet, not reading/writing
         an object)
```

## The kubectl exec/logs exception, explained
`kubectl exec`, `kubectl logs`, and `kubectl port-forward` are the **one architectural exception** worth calling out explicitly: for these, the API server acts as a **reverse proxy**, opening a direct connection *to the specific kubelet* hosting that Pod, which in turn connects to the container runtime to attach to the live process/stream. This is different in kind from the object watch/reconcile model — there's no "exec" object sitting in etcd; it's a live, ephemeral, streamed connection, proxied once through the API server for auth/authz purposes and then held open directly.

```bash
kubectl exec -it my-pod -- sh
# kubectl → apiserver (authenticated/authorized) → DIRECT connection to
# that Pod's node's kubelet (:10250) → container runtime → attached shell
```

---

# 32. FAILURE SCENARIOS

For each, the framing is always the same: **what breaks immediately, what keeps working, and what's the recovery path.**

## API server down

| | |
|---|---|
| **Breaks immediately** | All `kubectl` commands, all new scheduling, all reconciliation (controllers' watches disconnect) |
| **Keeps working** | Already-running Pods and their containers; kubelet's local liveness-probe restarts (Diagram D1); kube-proxy's already-installed iptables/IPVS rules; existing Service traffic routing |
| **Recovery** | Restart the API server process/Pod; in HA, other replicas absorb load with zero cluster-wide impact; controllers automatically reconnect their watches and resume exactly where they left off (via `resourceVersion`) |

## etcd unavailable

| | |
|---|---|
| **Breaks immediately** | Everything the API server does depends on etcd — so effectively, the entire control plane becomes read/write-dead the moment quorum is lost |
| **Keeps working** | Exactly the same as API server down (that's the actual mechanism — API server can't do anything useful without etcd) — running workloads are untouched |
| **Recovery** | Restore quorum (bring failed etcd members back, or restore from `etcdctl snapshot` backup onto a fresh member) — this is the single most critical backup/recovery procedure in all of Kubernetes operations |

## Scheduler unavailable

| | |
|---|---|
| **Breaks immediately** | New Pods pile up `Pending`, forever, until it recovers |
| **Keeps working** | Every already-scheduled, already-running Pod — completely unaffected |
| **Recovery** | HA setups fail over to a standby replica via leader election, typically within seconds; single-instance setups need the process restarted |

## Controller-manager unavailable

| | |
|---|---|
| **Breaks immediately** | No reconciliation of any kind — crashed Pods under a ReplicaSet are NOT replaced, Deployments don't roll out, dead nodes' Pods are never evicted/rescheduled |
| **Keeps working** | Existing Pods, Services, and all currently-correct state — nothing actively regresses, it just stops self-healing/progressing |
| **Recovery** | Same HA leader-election failover pattern as the scheduler |

## kubelet unavailable (on one node)

| | |
|---|---|
| **Breaks immediately** | No new Pods can start on that node; liveness/readiness probes stop being evaluated locally; status reporting stops |
| **Keeps working** | Containers already running on that node keep running at the OS/runtime level for a while — they're independent Linux processes, not tied to kubelet's own liveness |
| **Recovery** | After the heartbeat timeout (Diagram E), the node is marked `NotReady`/`Unknown`, and eventually its Pods are evicted and rescheduled elsewhere — a full node-failure-style recovery, not a quick fix, unless the kubelet process itself is restarted promptly |

## Full node failure

Already covered exhaustively in Sequence Diagram E above — summarized: 40s to `Unknown`, +5min (default) to eviction, then ordinary rescheduling onto healthy nodes. **Anything not in durable, network-attached storage is lost** — this is the single most important operational consequence of a full node failure, and exactly why StatefulSets pair with PersistentVolumes rather than node-local storage for anything that must survive this scenario.

## The one unifying insight across every failure scenario
**Control-plane failures degrade the cluster's ability to change; they never (by themselves) stop the cluster's ability to keep doing what it was already doing.** This single property — the separation of "deciding" from "doing" from Part 1 — is the entire reason Kubernetes clusters are as resilient as they are in practice, and it's worth re-deriving from first principles rather than memorizing as a fact: every component's failure story in this section follows directly from which side of that divide it sits on.

---

## What's coming in Part 6 (final part)

Part 6 closes out the masterclass: **High availability architecture, production cluster architecture, security boundaries, performance considerations, common beginner misunderstandings (consolidated), real-world production examples, Minikube hands-on labs, and the complete finale** — architecture cheat sheet, component comparison table, end-to-end request flow recap, a mental model for remembering it all, and the full interview question bank (20 beginner + 20 intermediate + 20 advanced + 5 troubleshooting scenarios).

Say **"continue"** whenever you're ready.
