# Kubernetes DaemonSets — Complete Masterclass (Beginner → Production)

---

## PART 1 — CONCEPTS

### What a DaemonSet is

A **DaemonSet** ensures that a copy of a specific Pod runs on **every node** (or a selected subset of nodes) in the cluster — automatically, all the time. When a new node joins, the DaemonSet controller schedules a Pod onto it without you doing anything. When a node leaves, that Pod is garbage collected with it.

```
Deployment mindset:  "I want N copies of my app, anywhere in the cluster"
DaemonSet mindset:   "I want exactly ONE copy of my agent on EVERY node,
                       no matter how many nodes there are or become"
```

### Why DaemonSets exist

Some workloads are fundamentally **node-scoped infrastructure**, not application replicas:

- A log collector needs to read `/var/log` **on every node** to ship logs from all Pods running there
- A monitoring agent (node-exporter) needs to expose **that specific node's** hardware/OS metrics
- A CNI plugin (Calico, Cilium) needs to configure networking **on every node** it runs on
- A security/runtime-protection agent needs eyes on every node's kernel/processes

None of these fit the "N interchangeable replicas, scheduler picks where" model of a Deployment — they need **exactly one instance per node, tied to that node's identity**, and they need to automatically appear/disappear as the cluster's node inventory changes. That's precisely what DaemonSet automates.

### DaemonSet vs Deployment

| | Deployment | DaemonSet |
|---|---|---|
| Replica count | You specify `replicas: N` | Automatic — one per matching node |
| Scheduler involvement | Normal scheduler picks node based on scoring | Historically bypassed scheduler; since 1.17+, uses scheduler with a `NodeAffinity` auto-injected per Pod |
| Scaling with cluster size | Doesn't auto-scale with node count | Auto "scales" — new node = new Pod automatically |
| Use case | Stateless apps, APIs | Node-level infrastructure agents |
| `replicas` field | Yes | Does not exist — no such field on DaemonSet |
| Node removal | No special handling | Pod on that node is deleted automatically |

### One Pod Per Node Concept

```
┌─────────── Cluster ───────────┐
│  Node A   Node B   Node C      │
│  ┌────┐   ┌────┐   ┌────┐      │
│  │agent│   │agent│   │agent│    │
│  └────┘   └────┘   └────┘      │
└────────────────────────────────┘
    exactly ONE agent Pod per node, always
```

If Node D joins the cluster tomorrow, a 4th `agent` Pod appears there automatically — no manual scaling action needed.

---

## PART 2 — SCHEDULING INTERNALS

### DaemonSet Controller

The **DaemonSet controller** (a control loop in `kube-controller-manager`) reconciles like this:

```
loop forever:
    nodes = list all Nodes (filtered by nodeSelector/affinity if specified)
    for each node in nodes:
        if node doesn't already have a Pod for this DaemonSet:
            create one, with nodeName/nodeAffinity pinning it to that node
    for each existing DaemonSet Pod:
        if its node no longer exists, or no longer matches selector/affinity:
            delete that Pod
```

Since Kubernetes 1.17, DaemonSet Pods are **not** directly bound via `spec.nodeName` at creation — instead, the controller injects a **required nodeAffinity** matching that specific node's name into the Pod spec, and lets the **normal scheduler** place it. This was a deliberate design change so DaemonSet Pods go through the same admission/scheduling pipeline (respecting resource pressure, etc.) as any other Pod, rather than being force-injected and potentially overcommitting a node.

```yaml
# What the controller effectively injects into each Pod (simplified):
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchFields:
        - key: metadata.name
          operator: In
          values: ["node-a"]     # this exact node, per Pod instance
```

### Node Selectors

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd          # DaemonSet Pod only scheduled to nodes with this label
```

Restricts the DaemonSet to a **subset** of nodes — e.g., you might only want a GPU-monitoring agent on GPU nodes.

### Affinity

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/worker
                operator: Exists
```

Same node affinity mechanism as any Pod — used for more expressive targeting than a flat `nodeSelector` (e.g., "any node with this label key present, any value").

### Taints and Tolerations

By default, a DaemonSet Pod is scheduled like any other Pod — meaning it will **NOT** land on a tainted node (e.g., a control-plane node tainted `node-role.kubernetes.io/control-plane:NoSchedule`) unless it has a matching toleration.

Most infra DaemonSets (logging, monitoring, CNI) need to run on **every** node, including control-plane/master nodes and even nodes that are `NotReady` or under pressure — so they carry broad tolerations:

```yaml
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
        operator: Exists
      - key: node.kubernetes.io/not-ready
        effect: NoExecute
        operator: Exists
        tolerationSeconds: 0      # tolerate immediately, don't wait to evict
      - key: node.kubernetes.io/unreachable
        effect: NoExecute
        operator: Exists
        tolerationSeconds: 0
      - key: node.kubernetes.io/disk-pressure
        effect: NoSchedule
        operator: Exists
      - key: node.kubernetes.io/memory-pressure
        effect: NoSchedule
        operator: Exists
```

**Why this matters:** a log collector that *stops running* the moment a node has disk pressure is exactly backwards — that's when you most need it running to diagnose the problem. Critical DaemonSets deliberately tolerate almost everything.

---

## PART 3 — YAML EXAMPLES (line by line)

### A. Basic DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: node-agent            # must match template labels (same rule as Deployment)
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
      - name: agent
        image: my-node-agent:1.0
```

**Note there's no `replicas` field** — that's the biggest structural difference from Deployment YAML. Count is implicit: number of matching nodes.

### B. Logging Agent (Fluent Bit) — real-world pattern

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: kube-system
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      tolerations:
      - operator: Exists           # tolerate ALL taints — run absolutely everywhere
      serviceAccountName: fluent-bit
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            memory: 256Mi           # no CPU limit — avoid throttling a latency-sensitive agent
        volumeMounts:
        - name: varlog
          mountPath: /var/log             # node's log directory — READ access to host logs
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log                   # bind-mounts the NODE's filesystem path
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

**Why `hostPath`:** this is the defining trait of most infra DaemonSets — they need to read/write **the node's actual filesystem** (logs, container runtime state, `/proc`, `/sys`), not an isolated Pod-local filesystem. `hostPath` volumes bind-mount a path from the node directly into the container.

**Why `tolerations: [operator: Exists]`:** a logging agent that skips tainted/cordoned/pressured nodes creates blind spots exactly when you need visibility most.

### C. Monitoring Agent (Node Exporter)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true              # use the NODE's network namespace directly (no Pod IP)
      hostPID: true                  # see the NODE's process tree, not just this Pod's
      tolerations:
      - operator: Exists
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.7.0
        args:
        - --path.rootfs=/host
        ports:
        - containerPort: 9100
          hostPort: 9100               # exposes directly on the node's own IP:port
        volumeMounts:
        - name: root
          mountPath: /host
          readOnly: true
      volumes:
      - name: root
        hostPath:
          path: /
```

**Why `hostNetwork: true` and `hostPID: true`:** node-exporter needs to see **the node's own** network stats and process list, not a Pod-isolated view — without these, it would only report on the "pause container's" isolated namespace, not the real node. This is a classic DaemonSet pattern for true node-level observability tools.

### D. Security Agent (Falco-style runtime detection)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: security
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      tolerations:
      - operator: Exists
      containers:
      - name: falco
        image: falcosecurity/falco:0.38.0
        securityContext:
          privileged: true          # needs kernel-level syscall visibility
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: boot
          mountPath: /host/boot
          readOnly: true
      volumes:
      - name: dev
        hostPath: { path: /dev }
      - name: proc
        hostPath: { path: /proc }
      - name: boot
        hostPath: { path: /boot }
```

Security agents typically need `privileged: true` plus deep `hostPath` access into `/proc`, `/dev`, kernel headers — DaemonSets are the standard vehicle for deploying this kind of node-embedded tooling consistently across the fleet.

### E. CNI Component (conceptual — e.g., Calico node agent)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      - operator: Exists
        effect: NoExecute
      containers:
      - name: calico-node
        image: calico/node:v3.28.0
        securityContext:
          privileged: true       # manipulates node's network interfaces/routing tables
        env:
        - name: NODENAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
```

CNI DaemonSets **must** run before any other Pod can get networking on that node — this is why they carry the broadest possible tolerations, including tolerating a node that's not yet `Ready` (chicken-and-egg: the node isn't network-ready until this Pod runs).

### F. Rolling Update Strategy

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1        # or a percentage, e.g. "10%"
      maxSurge: 0                # DaemonSets historically couldn't surge; now optional (1.22+)
```

#### RollingUpdate (default)
Updates node-by-node: terminate old Pod on a node → create new Pod on that same node → wait until Ready → move to next node, up to `maxUnavailable` nodes in flight at once.

```
Update triggered:
  Node A: [old] → [terminating] → [new, starting] → [new, Ready]
  Node B: [old] → [terminating] → [new, starting] → [new, Ready]   (starts once A frees a slot)
  ...
```

#### OnDelete
```yaml
spec:
  updateStrategy:
    type: OnDelete
```
Identical philosophy to StatefulSet's `OnDelete` — new Pod template only applies when **you manually delete** the Pod on a given node. Useful for very careful, staged rollout of critical node agents (e.g., updating a CNI plugin one node at a time with manual verification).

---

## PART 4 — INTERNALS: NODE LIFECYCLE EVENTS

### What happens when a new node joins

```
New Node registers with API server (kubelet --register-node)
       │
       ▼
Node object created in etcd, status eventually Ready
       │
       ▼
DaemonSet controller's watch loop notices a new Node
       │
       ▼
For EACH existing DaemonSet whose selector/affinity matches this node:
    controller creates a new Pod object,
    with nodeAffinity pinned to this node's name
       │
       ▼
Scheduler binds the Pod to that node (trivial — only one valid node)
       │
       ▼
kubelet on the new node creates sandbox, pulls images, starts containers
       │
       ▼
Node now running: node-exporter, fluent-bit, CNI agent, security agent, etc.
```

### What happens when a node is removed

```
Node deleted (kubectl delete node, or cloud autoscaler deregisters it)
       │
       ▼
Kubernetes garbage collector notices the Node object is gone
       │
       ▼
All Pods that were bound to that node (including DaemonSet Pods)
   are removed from etcd — there's nowhere for them to "move to";
   DaemonSet Pods are NEVER rescheduled elsewhere, by design
       │
       ▼
DaemonSet controller's desired-state count naturally drops by one
   (no explicit action needed — it just stops "seeing" that node)
```

### What happens when a DaemonSet Pod crashes

Same as any Pod: kubelet applies `restartPolicy` (DaemonSets almost always use `Always`) and restarts the container **on the same node**, in place — no rescheduling, no new Pod object, same name/IP. This is identical to normal Pod crash-restart behavior; DaemonSet-specific logic doesn't kick in for a mere container crash.

### What happens when a Pod is manually deleted

```bash
kubectl delete pod fluent-bit-x7k2p
```
```
Pod object deleted
       │
       ▼
DaemonSet controller's reconcile loop notices: node still exists,
   still matches selector, but has NO Pod for this DaemonSet
       │
       ▼
Controller creates a brand new Pod for that same node
       │
       ▼
New Pod scheduled (trivially, to that same node) and started
```
Net effect: near-immediate self-healing, same as a ReplicaSet replacing a deleted Pod — except the "slot" is a specific node, not an arbitrary one.

---

## PART 5 — ARCHITECTURE DIAGRAM

```
┌───────────────────────────── Cluster ─────────────────────────────┐
│                                                                       │
│   ┌─────────────┐         ┌──────────────────────┐                  │
│   │ DaemonSet    │────────►│ DaemonSet Controller  │                  │
│   │ "fluent-bit" │         │ (watch Nodes + Pods)  │                  │
│   └─────────────┘         └───────────┬───────────┘                  │
│                                         │ ensures 1 Pod per matching  │
│                                         │ node                         │
│         ┌───────────────────────────────┼───────────────────────────┐│
│         ▼                               ▼                           ▼│
│   ┌───────────┐                  ┌───────────┐                ┌───────────┐
│   │  Node A    │                  │  Node B    │                │  Node C    │
│   │ ┌────────┐ │                  │ ┌────────┐ │                │ ┌────────┐ │
│   │ │fluent-  │ │                  │ │fluent-  │ │                │ │fluent-  │ │
│   │ │bit Pod  │ │                  │ │bit Pod  │ │                │ │bit Pod  │ │
│   │ └────────┘ │                  │ └────────┘ │                │ └────────┘ │
│   │ hostPath:  │                  │ hostPath:  │                │ hostPath:  │
│   │ /var/log   │                  │ /var/log   │                │ /var/log   │
│   └───────────┘                  └───────────┘                └───────────┘
└──────────────────────────────────────────────────────────────────────┘
```

---

## PART 6 — RESOURCE MANAGEMENT

DaemonSet Pods run on **every node**, competing for resources with whatever application workloads also land there — this makes resource discipline more important than for a typical Deployment:

```yaml
resources:
  requests:
    cpu: 50m           # keep LOW — you're paying this cost on every single node
    memory: 64Mi
  limits:
    memory: 128Mi        # cap to prevent one node's agent from starving app Pods
```

**Guidance:**
- Keep `requests` small and predictable — they're subtracted from **every** node's allocatable capacity, cluster-wide, whether or not the agent needs it at that moment
- Avoid CPU `limits` on latency-sensitive agents (log shippers, metrics scrapers) to prevent throttling during bursts — but always set memory limits to protect the node from a leak
- Use `PriorityClass` for critical infra DaemonSets so they aren't the first thing evicted under node pressure:

```yaml
spec:
  template:
    spec:
      priorityClassName: system-node-critical   # built-in, reserved for core infra
```

---

## PART 7 — PRODUCTION USE CASES (summary)

| Category | Example tools | Why DaemonSet |
|---|---|---|
| Logging agents | Fluent Bit, Fluentd, Filebeat | Must read every node's container logs |
| Monitoring agents | node-exporter, Datadog agent | Must report per-node hardware/OS metrics |
| Security agents | Falco, Aqua, Sysdig | Must observe syscalls/processes on every node |
| CNI components | Calico, Cilium, Flannel | Must configure networking on every node |
| Storage components | Ceph/Rook OSD agents, CSI node plugins | Must manage local disks/mounts per node |
| kube-proxy | Built-in | Must program iptables/IPVS rules on every node |

---

## PART 8 — HANDS-ON MINIKUBE LABS

```bash
minikube start --nodes=3 --cpus=2 --memory=3072
```

### Lab 1 — Deploy a basic DaemonSet, observe 1-per-node
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: demo-agent
spec:
  selector:
    matchLabels: { app: demo-agent }
  template:
    metadata:
      labels: { app: demo-agent }
    spec:
      tolerations:
      - operator: Exists
      containers:
      - name: agent
        image: busybox
        command: ["sh", "-c", "while true; do echo alive; sleep 30; done"]
EOF
kubectl get pods -o wide -l app=demo-agent
# one Pod per node, including control-plane if tolerations allow
```

### Lab 2 — Add a node, watch auto-scheduling
```bash
minikube node add
kubectl get nodes -w
# in another terminal:
kubectl get pods -o wide -l app=demo-agent -w
# a new demo-agent Pod appears automatically on the new node
```

### Lab 3 — Remove a node
```bash
minikube node list
minikube node delete <node-name>
kubectl get pods -o wide -l app=demo-agent
# the Pod that was on that node is simply gone — not rescheduled elsewhere
```

### Lab 4 — Manual Pod deletion → self-heal
```bash
kubectl delete pod <one-of-the-demo-agent-pods>
kubectl get pods -o wide -l app=demo-agent -w
# a replacement Pod appears, pinned to the SAME node
```

### Lab 5 — nodeSelector restriction
```bash
kubectl label node <node-name> disktype=ssd
kubectl patch daemonset demo-agent --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/nodeSelector","value":{"disktype":"ssd"}}]'
kubectl get pods -o wide -l app=demo-agent
# Pods now only exist on the labeled node; others are terminated
```

### Lab 6 — Rolling update
```bash
kubectl set image daemonset/demo-agent agent=busybox:1.36
kubectl rollout status daemonset/demo-agent
kubectl rollout history daemonset/demo-agent
```

### Lab 7 — Taint tolerance behavior
```bash
kubectl taint node <node-name> dedicated=infra:NoSchedule
kubectl get pods -o wide -l app=demo-agent
# demo-agent Pod on that node is GONE (no toleration for "dedicated" taint)
# Fix: add a matching toleration and re-apply
```

---

## PART 9 — TROUBLESHOOTING

### DaemonSet Pod missing on a specific node

```bash
kubectl get daemonset demo-agent -o wide
kubectl describe daemonset demo-agent   # check "Desired/Current/Ready" counts + Events
kubectl get nodes --show-labels          # does the node match nodeSelector/affinity?
kubectl describe node <node-name>        # check Taints section
```

Common causes: node has a taint the DaemonSet doesn't tolerate, `nodeSelector`/`affinity` mismatch, node itself `NotReady` and DaemonSet lacks `not-ready` toleration, insufficient resources on that node for the Pod's `requests`.

### DaemonSet stuck mid-rollout

```bash
kubectl rollout status daemonset/demo-agent
kubectl get pods -l app=demo-agent -o wide   # which nodes are still on old version?
kubectl describe pod <stuck-pod>              # CrashLoopBackOff? ImagePull error?
```

`maxUnavailable` limits how many nodes update in parallel — a single stuck/crashing Pod on one node halts progress for the rest by design (fail-fast rather than break the whole fleet at once).

### hostPath permission errors

```bash
kubectl logs <pod> -n kube-system
```
Common cause: container's `securityContext` (non-root user) can't read/write the mounted `hostPath` due to file ownership on the node — often needs `privileged: true` or matching UID/GID, common with security/CNI agents.

---

## PART 10 — PRODUCTION BEST PRACTICES

- Use broad tolerations (`operator: Exists`) for infra DaemonSets that must run everywhere, including tainted and not-yet-ready nodes
- Set `priorityClassName: system-node-critical` (or `system-cluster-critical`) for essential infra agents so they're never evicted before regular app Pods
- Keep resource `requests` minimal and consistent — they're a permanent tax on every node's allocatable capacity
- Use `nodeSelector`/`affinity` to scope specialized DaemonSets (GPU monitoring, SSD-specific agents) rather than running them uselessly everywhere
- Prefer `RollingUpdate` with a conservative `maxUnavailable` (often `1`) for cluster-wide safety; use `OnDelete` for extremely sensitive components (CNI) where manual, verified node-by-node rollout is warranted
- Always test what happens when a DaemonSet Pod is missing on one node (simulate via taint) — many teams don't realize their DaemonSet silently skips certain nodes until an incident reveals it
- Namespace infra DaemonSets under `kube-system` or a dedicated `monitoring`/`security` namespace, and lock down RBAC tightly given the privileged access most of them need
- Monitor `DESIRED` vs `CURRENT` vs `READY` counts from `kubectl get daemonset` as a standing dashboard metric — a persistent mismatch means fleet-wide blind spots

### Common Mistakes

- Forgetting tolerations, so the DaemonSet silently never runs on control-plane or tainted nodes (common blind spot for logging/monitoring gaps)
- Over-requesting resources — DaemonSet Pods multiply the resource "tax" by node count, unlike Deployments
- Using `hostNetwork`/`hostPID`/`privileged` without understanding the security implications (necessary for some agents, but should be scoped and reviewed deliberately)
- Assuming a DaemonSet Pod will be "rescheduled" like a Deployment Pod when a node fails — it won't; there's no other valid node for that specific Pod, and it disappears with the node
- Not testing rolling updates against a `maxUnavailable` that's too high for a large cluster, risking simultaneous outage of a critical agent (e.g., CNI) across too many nodes at once
- Not setting a `PriorityClass`, letting critical infra Pods get evicted before regular app Pods under pressure — exactly backwards for something like a security agent

---

## PART 11 — 25 INTERVIEW QUESTIONS

1. What problem does DaemonSet solve that Deployment cannot?
2. Why doesn't a DaemonSet YAML have a `replicas` field?
3. How does the DaemonSet controller decide which nodes get a Pod?
4. Since which Kubernetes version does DaemonSet use the normal scheduler, and why was that changed?
5. How are DaemonSet Pods pinned to specific nodes internally?
6. What's the difference between `nodeSelector` and `nodeAffinity` for a DaemonSet?
7. Why do most infrastructure DaemonSets carry broad tolerations?
8. What taint would prevent a DaemonSet Pod from running on a control-plane node by default?
9. What's the purpose of `tolerationSeconds: 0` on a `not-ready`/`unreachable` toleration?
10. What is `hostPath`, and why do many DaemonSets rely on it?
11. Why would a DaemonSet use `hostNetwork: true`?
12. Why would a DaemonSet use `hostPID: true`?
13. What happens to a DaemonSet Pod when its node is deleted?
14. Does Kubernetes reschedule a DaemonSet Pod to another node if its node fails? Why or why not?
15. What happens automatically when a brand-new node joins the cluster?
16. What happens when you manually delete a single DaemonSet Pod?
17. Explain the DaemonSet `RollingUpdate` strategy and what `maxUnavailable` controls.
18. When would you choose `OnDelete` over `RollingUpdate` for a DaemonSet?
19. Why is resource `requests` sizing especially important for DaemonSets compared to Deployments?
20. What is `system-node-critical` `priorityClassName`, and why is it used for core infra DaemonSets?
21. Give three real-world categories of DaemonSet workloads.
22. Why does kube-proxy typically run as a DaemonSet?
23. How would you scope a DaemonSet to only run on GPU nodes?
24. What's a realistic troubleshooting path when a DaemonSet Pod is missing from exactly one node?
25. Design a rollout strategy for updating a CNI plugin DaemonSet across a 200-node production cluster with minimal risk.

---

That completes the DaemonSets chapter. Ready to continue into the next batch of the masterclass whenever you'd like — happy to proceed to Services & Networking, ConfigMaps/Secrets in depth, or wherever the approved plan sequences next.
