# Kubernetes Pods — Complete Masterclass (Beginner → Production)

This is a full deep-dive into Pods — the atomic unit of Kubernetes scheduling. I'll build your mental model from the ground up, then go layer by layer into internals, YAML, networking, lifecycle, troubleshooting, and production practice.

---

## PART 1 — FOUNDATIONS

### 1. What is a Pod?

A **Pod** is the smallest deployable unit in Kubernetes. It is **not** a container — it's a wrapper around one or more containers that:

- Share a network namespace (same IP, same port space)
- Share storage volumes
- Are scheduled together, onto the same node
- Live and die together (mostly)

Think of a Pod as a **logical host** — like a tiny virtual machine — that happens to run one or more containers inside it.

```
┌─────────────────────────────────────────┐
│                  POD                     │
│  IP: 10.244.1.7                          │
│  ┌───────────┐  ┌───────────┐            │
│  │Container A│  │Container B│            │
│  │ (main app)│  │ (sidecar) │            │
│  └───────────┘  └───────────┘            │
│         shared network + volumes         │
└─────────────────────────────────────────┘
```

### 2. Why Kubernetes uses Pods instead of directly managing containers

Kubernetes needed an abstraction above "container" because real applications often need **tightly coupled helper processes** — log shippers, proxies, config reloaders — that must:

- Share localhost networking with the main app (no service discovery overhead)
- Share a filesystem (for logs, sockets, configs)
- Start/stop together
- Be scheduled as one atomic unit onto one node

If Kubernetes only understood single containers, patterns like "sidecar proxy" (Envoy, Istio), "log shipper" (Fluentd), and "init setup step" would require external orchestration hacks. The Pod groups these into one schedulable unit, so the scheduler makes **one placement decision** for the whole group.

Also: Kubernetes is designed to be container-runtime-agnostic (Docker, containerd, CRI-O). Pod is a **Kubernetes-native abstraction**, decoupled from any particular container engine's model.

### 3. Pod vs Container

| Aspect | Container | Pod |
|---|---|---|
| Definition | A single running process, isolated via namespaces/cgroups | A group of one or more containers sharing network/storage |
| Scheduling unit | Not scheduled directly by k8s | The unit the scheduler places on a node |
| IP address | N/A alone | Pod gets one cluster IP, shared by all its containers |
| Lifecycle | Managed within Pod | Pod manages its containers' collective lifecycle |
| Networking | Isolated by default | Shared namespace across containers in Pod |

### 4. Pod vs Docker Container

A "Docker container" is just an OS-level process isolated using Linux namespaces + cgroups. A Kubernetes Pod, when run under a Docker/containerd-based node, is actually implemented as:

- One **infrastructure container** (the "pause" container) that holds the network namespace
- One or more **application containers** that join that same network namespace

So technically, a Pod *is* multiple Docker/OCI containers under the hood — Kubernetes just presents them to you as a single unit. This is a key internals fact (explained fully in section 5 below).

### 5. Pod Structure (internals)

When you create a Pod, the container runtime (via the **kubelet** and **CRI**) does this:

```
1. kubelet asks CRI (containerd/CRI-O) to create the Pod "sandbox"
   → This creates the "pause" container:
       - Creates a new network namespace
       - Creates a new IPC namespace
       - (optionally) UTS namespace (hostname)
   → Pause container's only job: hold these namespaces open,
     do nothing else (literally just calls pause() syscall)

2. kubelet pulls each container image (via CRI)

3. For each container in the Pod spec (initContainers first, then containers):
   kubelet asks CRI to create the container
   → Passed the network namespace of the pause container
   → Container therefore shares: IP, port space, loopback (localhost)
   → Container does NOT share PID namespace by default
     (each container has its own PID 1, unless shareProcessNamespace: true)

4. Volumes are mounted into each container per volumeMounts spec
```

```
Node
 └── Pod Sandbox (network namespace "netns-abc")
       ├── pause container  (owns netns, IPC ns)
       ├── app container    (joins netns-abc)
       └── sidecar container(joins netns-abc)
```

This is *why* killing/restarting one container doesn't lose the Pod IP — the **pause container** owns the network namespace, and it only dies when the whole Pod is torn down.

---

## PART 2 — LIFECYCLE, PHASES, STATUS

### 6. Pod Lifecycle (full internal flow)

```
User applies Pod YAML
        │
        ▼
 API Server validates + admission webhooks
        │
        ▼
   etcd stores Pod object (phase: Pending)
        │
        ▼
   Scheduler watches for unscheduled Pods
        │  picks a Node based on:
        │   - resource requests
        │   - affinity/anti-affinity
        │   - taints/tolerations
        │   - nodeSelector
        ▼
   Pod.spec.nodeName is set (binding)
        │
        ▼
  kubelet on that node notices the new Pod
        │
        ▼
  kubelet → CRI: create sandbox (pause container)
        │
        ▼
  kubelet → CRI: pull images
        │
        ▼
  initContainers run SEQUENTIALLY, each must exit 0
        │
        ▼
  main containers start (can be parallel)
        │
        ▼
  readinessProbe / livenessProbe / startupProbe begin
        │
        ▼
  Pod phase → Running
        │
        ▼
  (container exits / crashes / evicted / deleted)
        │
        ▼
  Pod phase → Succeeded / Failed / Terminating
```

### 7. Pod Phases

The `status.phase` field — a **coarse, high-level** summary:

| Phase | Meaning |
|---|---|
| `Pending` | Accepted by cluster, but not all containers created yet (image pulling, waiting for scheduling) |
| `Running` | Pod bound to a node, at least one container running |
| `Succeeded` | All containers terminated successfully (exit 0), won't restart |
| `Failed` | All containers terminated, at least one failed (non-zero exit) |
| `Unknown` | Node unreachable, kubelet can't report status |

Note: `Phase` does NOT tell you about individual container health — for that you need **Conditions** and **container statuses**.

### 8. Pod Conditions

`status.conditions` is a list of booleans with timestamps, giving finer detail:

| Condition | Meaning |
|---|---|
| `PodScheduled` | Pod has been assigned to a node |
| `Initialized` | All init containers completed successfully |
| `ContainersReady` | All containers report ready |
| `Ready` | Pod ready to serve traffic (used by Services/Endpoints) |

```bash
kubectl get pod mypod -o jsonpath='{.status.conditions}'
```

A Pod can be `Running` (phase) but **not** `Ready` (condition) — e.g., app started but readiness probe failing. This is the #1 confusion point for beginners.

### 9. Pod Status (full object)

`kubectl describe pod` shows the aggregate: phase + conditions + per-container statuses (`state`, `lastState`, `restartCount`, `ready`). Container `state` is one of:

- `Waiting` (reason: e.g. `ContainerCreating`, `ImagePullBackOff`, `CrashLoopBackOff`)
- `Running` (with `startedAt`)
- `Terminated` (with `exitCode`, `reason`, `finishedAt`)

---

## PART 3 — NETWORKING

### 10. Pod IP

Every Pod gets **one unique IP address** from the cluster's Pod CIDR, assigned by the CNI plugin (Calico, Cilium, Flannel, etc.) when the sandbox is created. This IP:

- Is routable within the cluster (in most CNI setups) without NAT
- Is ephemeral — changes every time the Pod is recreated
- Is shared by ALL containers in that Pod

```bash
kubectl get pod mypod -o wide   # shows Pod IP
```

### 11. Pod Networking

```
      Cluster Network (Pod CIDR e.g. 10.244.0.0/16)
 ┌───────────────┐        ┌───────────────┐
 │ Node A         │        │ Node B         │
 │ ┌───────────┐  │        │ ┌───────────┐  │
 │ │Pod 10.244 │  │  CNI   │ │Pod 10.244 │  │
 │ │  .1.5     │◄─┼────────┼─►│  .2.9     │  │
 │ └───────────┘  │ overlay│ └───────────┘  │
 └───────────────┘  /route └───────────────┘
```

Each Pod behaves like a node on a flat virtual network — Pod A can reach Pod B's IP directly (no NAT), regardless of which node they're on, as long as the CNI plugin implements this (all standard CNIs do — it's a Kubernetes networking model requirement: "every Pod can reach every other Pod without NAT").

### 12. Pod Namespaces (Linux kernel namespaces, not to be confused with Kubernetes `Namespace` objects)

Per Pod, by default:

| Namespace | Shared across containers in Pod? |
|---|---|
| Network (`netns`) | ✅ Yes — same IP, ports, interfaces |
| IPC | ✅ Yes — can use shared memory/semaphores |
| UTS (hostname) | ✅ Yes — same hostname |
| PID | ❌ No (unless `spec.shareProcessNamespace: true`) |
| Mount | ❌ No — each container has its own filesystem view, except shared volumes |
| User | ❌ No, by default |

### 13-15. Containers inside the same Pod / Shared network namespace / localhost communication

Because containers in a Pod share the network namespace:

```yaml
containers:
- name: app
  image: myapp:1.0
  ports:
  - containerPort: 8080
- name: sidecar
  image: envoy:latest
  ports:
  - containerPort: 9090
```

The `sidecar` container can reach the `app` container at `localhost:8080` — **no Service, no DNS needed** — because they're on the same loopback interface. This is the single biggest reason multi-container Pods exist: near-zero-latency inter-process communication.

⚠️ Caveat: **ports must not collide** — since it's one network namespace, two containers can't both bind to port 8080.

### 16. Shared Volumes

Containers in a Pod can mount the **same volume** at different paths, enabling file-based communication:

```
Pod
 ├── Container A writes logs to /var/log/app
 └── Container B (log shipper) reads from /var/log/app
        (same emptyDir volume, mounted in both)
```

---

## PART 4 — MULTI-CONTAINER PATTERNS

### 17. Init Containers

Run **before** app containers, **sequentially**, each must complete (exit 0) before the next starts. Used for setup: waiting for a dependency, running migrations, downloading config.

```
initContainer-1 → exits 0 → initContainer-2 → exits 0 → main containers start
```

If an init container fails, kubelet retries it according to `restartPolicy` (Pod-level) — the Pod stays `Pending`/`Init:Error` until it succeeds (or is capped by backoff).

### 18. Sidecar Containers

A helper container that runs **alongside** the main container for the Pod's whole lifetime: log shippers, service mesh proxies (Envoy), metrics exporters. Since Kubernetes 1.28+, you can mark an init container as a true **native sidecar** using `restartPolicy: Always` inside `initContainers` — it starts before app containers, but keeps running (and is stopped last on termination).

```yaml
initContainers:
- name: istio-proxy
  image: envoyproxy/envoy
  restartPolicy: Always   # this makes it a "native sidecar" (K8s 1.29+ stable)
```

### 19. Ephemeral Containers

Injected into a **running** Pod temporarily for debugging (no restart needed) — can't be added via normal spec update, only via `kubectl debug`:

```bash
kubectl debug -it mypod --image=busybox --target=app-container
```

Useful when your app image has no shell/debugging tools (distroless images).

### 20. Multi-container Pods — when to use

| Pattern | Example |
|---|---|
| Sidecar | Envoy proxy, log shipper |
| Ambassador | Proxy to simplify connecting to external service |
| Adapter | Normalize app output for monitoring |
| Init | DB migration, wait-for-dependency |

Rule of thumb: **only** co-locate containers that must share fate (start/stop together) and benefit from localhost/volume sharing. Otherwise, use separate Pods/Deployments.

---

## PART 5 — YAML DEEP DIVES (line-by-line)

### A. Basic Pod

```yaml
apiVersion: v1              # Core API group, "v1" — Pods have been stable since early k8s
kind: Pod                   # Object type
metadata:
  name: basic-pod            # Unique name within namespace
  namespace: default         # Logical partition of cluster resources
  labels:
    app: basic-pod            # Key-value used for selection (Services, controllers)
spec:
  containers:
  - name: app                 # Container name, unique within Pod
    image: nginx:1.25          # Image + tag (always pin a tag in production!)
    ports:
    - containerPort: 80         # Documentation only — doesn't actually open the port;
                                 # informs tooling/other devs what the app listens on
```

**Internals:** API server validates schema → admission controllers run (e.g. PodSecurity, ResourceQuota) → object persisted to etcd → scheduler assigns `nodeName` → kubelet on that node creates sandbox + pulls `nginx:1.25` + starts container.

### B. Multi-container Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: web
    image: nginx:1.25
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html   # nginx serves files from here
  - name: content-generator
    image: busybox
    command: ["/bin/sh", "-c", "while true; do date > /data/index.html; sleep 10; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data                    # same volume, different mount path
  volumes:
  - name: shared-data
    emptyDir: {}                          # ephemeral volume, lives as long as the Pod
```

**Internals:** kubelet creates the `emptyDir` as a directory on the node's disk (or tmpfs if `medium: Memory`), bind-mounts it into both containers' mount namespaces at their respective `mountPath`. The `content-generator` writes a file every 10s; `web` (nginx) serves it — classic sidecar file-sharing pattern.

### C. Init Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db-service 5432; do echo waiting; sleep 2; done']
    # Runs BEFORE app container. Must exit 0 before proceeding.
  containers:
  - name: app
    image: myapp:1.0
```

**Internals:** kubelet runs `wait-for-db`, blocks the Pod in `Init:0/1` status until it exits 0. Only then does it start `app`. If `wait-for-db` crashes, kubelet applies `restartPolicy` and retries with exponential backoff (visible as `Init:CrashLoopBackOff`).

### D. Sidecar (native, K8s 1.29+)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  initContainers:
  - name: log-shipper
    image: fluent-bit:latest
    restartPolicy: Always     # marks this as a sidecar, not a one-shot init container
    volumeMounts:
    - name: logs
      mountPath: /var/log
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir: {}
```

**Internals:** kubelet starts `log-shipper` first (like a normal init container), but because `restartPolicy: Always` is set, it does NOT block waiting for it to exit — it starts main containers once `log-shipper`'s own startup/readiness is satisfied, and keeps `log-shipper` running for the Pod's whole life. On shutdown, sidecars are stopped **last** (after main containers), so they can flush final logs.

### E. Pod with Volume (persistent)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: postgres:16
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pg-data-claim     # must already exist (PVC bound to a PV)
```

**Internals:** kubelet asks the CSI driver to attach & mount the volume backing `pg-data-claim` onto the node, then bind-mounts it into the container's mount namespace. Data survives Pod restarts/recreation (unlike `emptyDir`).

### F. Pod with Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: LOG_LEVEL
      value: "debug"                          # literal value
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password                        # pulled from a Secret at container start
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name              # Downward API — injects Pod's own metadata
    envFrom:
    - configMapRef:
        name: app-config                        # bulk-import all keys from a ConfigMap
```

**Internals:** kubelet resolves all `env`/`envFrom` values (fetching Secret/ConfigMap data from API server, resolving Downward API fields from the Pod object itself) **before** starting the container, and passes them as the container process's environment — a one-time snapshot; they do NOT auto-update if the Secret/ConfigMap changes later.

### G. Pod with Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    startupProbe:
      httpGet:
        path: /startupz
        port: 8080
      failureThreshold: 30
      periodSeconds: 2          # gives slow-starting apps up to 60s before liveness kicks in
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
      failureThreshold: 3        # 3 consecutive failures → kubelet kills & restarts container
    readinessProbe:
      httpGet:
        path: /readyz
        port: 8080
      periodSeconds: 5
      failureThreshold: 3        # failing → removed from Service Endpoints, NOT restarted
```

**Internals:** kubelet itself executes these probes (HTTP GET/TCP/exec) directly against the container on a timer — no separate agent involved. `startupProbe` suppresses liveness/readiness checks until it succeeds (protects slow-boot apps from being killed prematurely). `livenessProbe` failure → kubelet kills the container (SIGTERM then SIGKILL) and restarts per `restartPolicy`. `readinessProbe` failure → kubelet updates the container's `ready` status → Endpoint controller removes the Pod IP from the Service's Endpoints (no restart, just traffic removal).

### H. Pod with Resource Requests/Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    resources:
      requests:
        cpu: "250m"        # 0.25 vCPU — used by scheduler for node placement decision
        memory: "256Mi"     # guaranteed minimum memory
      limits:
        cpu: "500m"          # hard cap enforced via CFS cgroup quota (throttling, not OOM)
        memory: "512Mi"       # hard cap — exceeding this → OOMKilled by kernel cgroup OOM killer
```

**Internals:** `requests` are what the **scheduler** sums against node's allocatable capacity to decide placement — pure bookkeeping, no runtime enforcement by itself. `limits` are enforced by the kernel via cgroups: CPU limit → CFS bandwidth throttling (process gets throttled, not killed); memory limit → if the container's cgroup exceeds it, the **kernel OOM killer** sends SIGKILL to the process (visible as `OOMKilled` in Pod status).

### I. Pod with SecurityContext

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:              # Pod-level — applies to all containers
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000                # volumes' ownership set to this GID
  containers:
  - name: app
    image: myapp:1.0
    securityContext:            # Container-level — overrides Pod-level for this container
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]             # drop all Linux capabilities (least privilege)
```

**Internals:** kubelet passes these settings to the CRI/OCI runtime spec when creating the container: `runAsUser`/`runAsNonRoot` set the container process's UID (enforced at container start — kubelet refuses to start if `runAsNonRoot: true` but image's default user is root and no `runAsUser` given); `fsGroup` triggers a **recursive chown/chgrp** of mounted volumes to that GID at mount time; `capabilities.drop` removes Linux capabilities from the container's process via the OCI runtime spec (removes ability to do privileged syscalls even as root).

---

## PART 6 — DIAGRAMS

### Pod Architecture

```
┌────────────────────────────── Node ──────────────────────────────┐
│  kubelet                                                           │
│    │                                                                │
│    ▼                                                                │
│  CRI (containerd/CRI-O)                                            │
│    │                                                                │
│    ▼                                                                │
│  ┌───────────────── Pod Sandbox (netns) ─────────────────┐         │
│  │  pause container (owns network+IPC ns)                │         │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐             │         │
│  │  │ init-c1  │→ │  app     │  │ sidecar  │             │         │
│  │  │ (exits)  │  │(main proc│  │(long-run)│             │         │
│  │  └──────────┘  └──────────┘  └──────────┘             │         │
│  │           shared volumes, shared localhost             │         │
│  └──────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### Container Communication

```
   Container A ──── localhost:8080 ────► Container B
        │                                      │
        └──────── shared volume /data ─────────┘
```

### Pod Scheduling Flow

```
 kubectl apply -f pod.yaml
       │
       ▼
  API Server ── validate ──► etcd (Pending, no nodeName)
       │
       ▼
  Scheduler watch loop
   ├─ Filter nodes (resources, taints, nodeSelector, affinity)
   ├─ Score remaining nodes (bin-packing, spread, etc.)
   └─ Pick best node → PATCH Pod.spec.nodeName
       │
       ▼
  kubelet on chosen node (watching for Pods bound to it)
       │
       ▼
  Pod creation as described above
```

---

## PART 7 — LIFECYCLE EVENTS: RESTART, DELETE, TERMINATION

### 21. Container Lifecycle (hooks)

```yaml
lifecycle:
  postStart:
    exec:
      command: ["/bin/sh", "-c", "echo Started > /tmp/started"]
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]   # give load balancer time to deregister
```

- `postStart`: fired right after container start (async — not guaranteed to run before the container's main process's first instructions)
- `preStop`: fired **before** SIGTERM is sent — commonly used to drain in-flight connections

### 22. Restart Policies

Pod-level, applies to **all containers**:

| Policy | Behavior |
|---|---|
| `Always` (default) | Container always restarted on exit, regardless of exit code |
| `OnFailure` | Restarted only on non-zero exit code |
| `Never` | Never restarted |

Used with exponential backoff (10s, 20s, 40s... capped at 5min) — this is the mechanism behind `CrashLoopBackOff`.

### 33. Graceful Shutdown (the full termination sequence)

```
kubectl delete pod  (or controller replaces Pod)
       │
       ▼
1. API server sets deletionTimestamp on Pod object
       │
       ▼
2. Pod removed from Service Endpoints immediately
   (so no NEW traffic is routed to it)
       │
       ▼
3. kubelet runs preStop hook (if defined) — synchronous, blocks next step
       │
       ▼
4. kubelet sends SIGTERM to container's PID 1
       │
       ▼
5. App should catch SIGTERM and gracefully finish in-flight requests
       │
       ▼
6. Wait up to terminationGracePeriodSeconds (default 30s)
       │
       ▼
7. If still running → kubelet sends SIGKILL (forceful)
       │
       ▼
8. Pod object deleted from etcd
```

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 10"]
```

⚠️ Common bug: Pod removed from Endpoints happens **in parallel** with SIGTERM being sent — there's a race window where a client can still send a request to a Pod that's shutting down. Always add a short `preStop sleep` + handle SIGTERM properly.

### 31-32. Pod Deletion vs Termination

- **Deletion** = intent expressed via API (`deletionTimestamp` set)
- **Termination** = actual process of stopping containers (steps 3-7 above)

A Pod can sit in `Terminating` state for a while if it doesn't respond to SIGTERM (grace period runs out → SIGKILL).

### 34. Pod Eviction

Kubelet or the cluster (via **Node Pressure Eviction** or the **Descheduler**) can evict Pods when a node is under resource pressure (memory, disk):

```
kubelet monitors node resource pressure
   → memory.available < eviction threshold
   → kubelet ranks Pods by QoS class + usage-over-request
   → evicts lowest-priority Pods first (BestEffort → Burstable → Guaranteed)
```

Also: `PodDisruptionBudget`-aware evictions happen via the **Eviction API** (used by `kubectl drain`, cluster-autoscaler).

### 35. Pod Restart vs Pod Recreation

| | Restart | Recreation |
|---|---|---|
| What happens | Same Pod object, container process restarted in place | Old Pod object deleted, brand-new Pod object created |
| Pod IP | Unchanged | New IP assigned |
| Pod UID | Unchanged | New UID |
| Volumes (emptyDir) | Preserved | Lost (new empty volume) |
| Triggered by | Liveness probe failure, crash (restartPolicy) | Deployment rollout, Node failure, manual delete |

### 36. Static Pods

Pods defined by a YAML file directly on a **node's filesystem** (usually `/etc/kubernetes/manifests/`), managed directly by that node's kubelet — **not** through the API server/scheduler. The kubelet watches that directory and creates a "mirror Pod" in the API server for visibility (but you can't fully control it via `kubectl delete` — kubelet will just recreate it, since the source of truth is the local file).

Used for: control plane components themselves (`kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager` often run as static Pods on control-plane nodes).

```bash
# On a control-plane node:
ls /etc/kubernetes/manifests/
# kube-apiserver.yaml, etcd.yaml, kube-scheduler.yaml, kube-controller-manager.yaml
```

---

## PART 8 — QOS, DISRUPTION, SCHEDULING PREFERENCES

### 37. Pod QoS Classes

Derived automatically from `requests`/`limits` — you never set this directly:

| Class | Condition | Eviction priority |
|---|---|---|
| `Guaranteed` | requests == limits for **all** resources, on **every** container | Evicted last |
| `Burstable` | At least one container has requests set, but not equal to limits | Evicted second |
| `BestEffort` | No requests/limits set at all | Evicted first |

```bash
kubectl get pod mypod -o jsonpath='{.status.qosClass}'
```

### 38. Pod Disruption

- **Voluntary disruption**: node drains, rolling updates, manual deletes — controllable via `PodDisruptionBudget`
- **Involuntary disruption**: node crashes, kernel panic, hardware failure — not controllable

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2       # or maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

This tells the cluster: "never voluntarily evict Pods if it would drop available replicas below this threshold" — respected by `kubectl drain` and cluster-autoscaler.

### 39. Pod Affinity / Anti-Affinity

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values: ["myapp"]
      topologyKey: "kubernetes.io/hostname"   # never co-locate 2 replicas on same node
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values: ["ssd"]
```

- `podAffinity`: prefer/require scheduling **near** Pods matching a label (e.g. co-locate cache next to app)
- `podAntiAffinity`: prefer/require scheduling **away** from Pods matching a label (e.g. spread replicas across nodes/zones for HA)
- `nodeAffinity`: constrain which nodes a Pod can go to, based on node labels

`requiredDuringScheduling...` = hard constraint (Pod stays Pending if unsatisfiable); `preferredDuringScheduling...` = soft, best-effort.

---

## PART 9 — KUBECTL COMMANDS FOR INSPECTION & TROUBLESHOOTING

```bash
# Basic inspection
kubectl get pods                          # list Pods
kubectl get pods -o wide                  # + node, IP
kubectl get pods --show-labels            # + labels
kubectl describe pod <name>               # full details, events (MOST USEFUL command)
kubectl get pod <name> -o yaml            # full raw spec+status

# Logs
kubectl logs <pod>                        # current container's logs
kubectl logs <pod> -c <container>         # specific container (multi-container Pod)
kubectl logs <pod> --previous             # logs from PREVIOUS crashed instance
kubectl logs -f <pod>                     # follow/stream

# Exec / debug
kubectl exec -it <pod> -- /bin/sh         # shell into container
kubectl exec -it <pod> -c <container> -- sh
kubectl debug -it <pod> --image=busybox --target=<container>   # ephemeral container debug
kubectl debug node/<nodename> -it --image=busybox               # debug node directly

# Port-forward / copy
kubectl port-forward pod/<name> 8080:80
kubectl cp <pod>:/path/in/pod ./local/path

# Events (crucial for troubleshooting)
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events --field-selector involvedObject.name=<pod>

# Resource usage
kubectl top pod <name>                    # needs metrics-server

# Delete / force delete
kubectl delete pod <name>
kubectl delete pod <name> --grace-period=0 --force   # last resort, can orphan resources

# Status fields directly
kubectl get pod <name> -o jsonpath='{.status.phase}'
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[*].restartCount}'
```

---

## PART 10 — COMMON ERRORS (deep diagnosis)

### CrashLoopBackOff

**Meaning:** Container starts, exits (crash or clean exit with `restartPolicy: Always`), kubelet restarts it, it crashes again — backoff delay increases each time.

**Diagnose:**
```bash
kubectl logs <pod> --previous     # see why it crashed last time
kubectl describe pod <pod>        # check Last State, Exit Code, Events
```

**Common causes:** app crashes on bad config/missing env var, misconfigured entrypoint, `command` exits immediately (e.g., you forgot to run a foreground process), liveness probe too aggressive killing a slow-starting app.

### ImagePullBackOff / ErrImagePull

**Meaning:** kubelet can't pull the image.

**Diagnose:**
```bash
kubectl describe pod <pod>   # look for Events: "Failed to pull image..."
```

**Common causes:** typo in image name/tag, private registry without `imagePullSecrets`, image doesn't exist, registry auth expired, rate limiting (Docker Hub).

### Pending

**Meaning:** Pod accepted but not scheduled/started.

**Diagnose:**
```bash
kubectl describe pod <pod>   # look at Events for scheduling failures
```

**Common causes:** insufficient cluster resources (no node has enough CPU/mem to satisfy `requests`), unsatisfiable node affinity/taints, PVC not bound, `nodeSelector` matching no node.

### OOMKilled

**Meaning:** container exceeded its memory `limit`, kernel cgroup OOM killer sent SIGKILL.

**Diagnose:**
```bash
kubectl describe pod <pod>   # Last State: Terminated, Reason: OOMKilled
```

**Common causes:** memory limit set too low, memory leak in app, spike in load without matching limit headroom.

### FailedMount

**Meaning:** volume couldn't be attached/mounted.

**Diagnose:**
```bash
kubectl describe pod <pod>   # Events: "Unable to attach or mount volumes..."
```

**Common causes:** PVC not bound / doesn't exist, wrong `claimName`, node can't reach storage backend, permission issues (fsGroup mismatch), volume already attached to another node (RWO conflict) — common with EBS/cloud block storage during a rolling update.

### CreateContainerError / CreateContainerConfigError

**Meaning:** kubelet/CRI failed to actually create the container (before it even starts running).

**Common causes:** referenced ConfigMap/Secret doesn't exist, invalid `securityContext` (e.g., `runAsNonRoot: true` but image only has root user), invalid volume mount path conflicts, invalid resource values.

---

## PART 11 — HANDS-ON MINIKUBE LABS

### Setup
```bash
minikube start --cpus=4 --memory=4096
kubectl config use-context minikube
```

### Lab 1 — Basic Pod + Inspection
```bash
kubectl run nginx-pod --image=nginx:1.25 --port=80
kubectl get pods -o wide
kubectl describe pod nginx-pod
kubectl port-forward pod/nginx-pod 8080:80
# visit http://localhost:8080
kubectl delete pod nginx-pod
```

### Lab 2 — CrashLoopBackOff (intentional)
```bash
kubectl run crash-pod --image=busybox --restart=Always -- sh -c "exit 1"
kubectl get pods -w             # watch it cycle through backoff
kubectl logs crash-pod --previous
kubectl delete pod crash-pod
```

### Lab 3 — Multi-container + shared volume
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: shared-vol-demo
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh","-c","while true; do date >> /data/log.txt; sleep 5; done"]
    volumeMounts:
    - name: data
      mountPath: /data
  - name: reader
    image: busybox
    command: ["sh","-c","tail -f /data/log.txt"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
EOF
kubectl logs shared-vol-demo -c reader -f
```

### Lab 4 — Init container gating startup
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-lab
spec:
  initContainers:
  - name: delay
    image: busybox
    command: ["sh","-c","echo waiting...; sleep 15; echo done"]
  containers:
  - name: app
    image: nginx:1.25
EOF
kubectl get pods -w    # observe Init:0/1 → Running
```

### Lab 5 — OOMKilled
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-lab
spec:
  containers:
  - name: hog
    image: polinux/stress
    resources:
      limits:
        memory: "50Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
EOF
kubectl get pod oom-lab -w
kubectl describe pod oom-lab   # Reason: OOMKilled
```

### Lab 6 — Probes & Readiness
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probe-lab
spec:
  containers:
  - name: app
    image: nginx:1.25
    readinessProbe:
      httpGet:
        path: /nonexistent
        port: 80
      periodSeconds: 5
EOF
kubectl get pod probe-lab -w    # READY 0/1 forever — see condition
kubectl describe pod probe-lab | grep -A5 Conditions
```

---

## PART 12 — CHEAT SHEET, INTERVIEW QUESTIONS, MISTAKES, CHECKLIST, MENTAL MODEL

### Pod Cheat Sheet

```
Smallest deployable unit = Pod (not container)
Pod = shared network ns + IPC ns + volumes, group of containers
Pod gets ONE IP, shared by all containers → use localhost between them
initContainers: run sequentially before app containers, must succeed
Native sidecars: initContainer + restartPolicy: Always
emptyDir: ephemeral, node-local, dies with Pod
PVC: persistent, survives Pod recreation
restartPolicy: Always | OnFailure | Never (Pod-level, all containers)
Probes: liveness=restart, readiness=remove from Service, startup=delay others
QoS: Guaranteed > Burstable > BestEffort (eviction priority, reverse)
Termination: Endpoints removed → preStop → SIGTERM → grace period → SIGKILL
Static Pods: node-local YAML, kubelet-managed, not via API server scheduling
```

### Important kubectl Commands
```
kubectl get pods -o wide
kubectl describe pod <name>
kubectl logs <pod> [-c container] [--previous] [-f]
kubectl exec -it <pod> -- sh
kubectl debug -it <pod> --image=busybox --target=<container>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pod <name>
kubectl delete pod <name> --grace-period=0 --force
```

### 30 Interview Questions

1. What is a Pod and why does Kubernetes use it instead of raw containers?
2. What's the difference between a Pod and a container?
3. How many IP addresses does a multi-container Pod have?
4. What is the "pause" container and why does it exist?
5. What Linux namespaces are shared between containers in a Pod, and which aren't?
6. How do containers within a Pod communicate?
7. What's the difference between Pod phase and Pod condition?
8. Name all Pod phases.
9. What does `Ready` condition mean vs `Running` phase?
10. What's the difference between a liveness probe and a readiness probe?
11. What does a startup probe do and why is it needed?
12. What happens when a liveness probe fails? A readiness probe?
13. Explain the full Pod termination sequence.
14. Why can there be a race condition during Pod termination, and how do you mitigate it?
15. What's the difference between `emptyDir` and a `PersistentVolumeClaim` volume?
16. What are init containers, and how do they differ from sidecar containers?
17. What is a "native sidecar" in newer Kubernetes versions?
18. What restart policies exist, and at what level are they set?
19. Explain requests vs limits for CPU and memory.
20. What happens when a container exceeds its memory limit? Its CPU limit?
21. What are the three QoS classes, and how are they determined?
22. Which QoS class gets evicted first under node pressure?
23. What's the difference between Pod restart and Pod recreation?
24. What is a static Pod, and how does it differ from a normal Pod?
25. What is a PodDisruptionBudget and what does it protect against?
26. Explain pod affinity vs anti-affinity vs node affinity.
27. What's the difference between `requiredDuringScheduling` and `preferredDuringScheduling`?
28. Why would a Pod stay stuck in `Pending`?
29. What's the difference between `CrashLoopBackOff` and `ImagePullBackOff`?
30. How do ephemeral containers help with debugging distroless images?

### Common Mistakes

- Not setting resource `requests`/`limits` → noisy neighbor problems, unpredictable evictions
- Assuming `Running` phase means the app is healthy (ignoring `Ready` condition)
- Not handling SIGTERM in the app → hard kills, dropped connections on every deploy
- Using `latest` tag → non-reproducible deployments
- Putting unrelated containers in one Pod → coupling failure domains unnecessarily
- Forgetting `imagePullSecrets` for private registries
- Setting `livenessProbe` too aggressively on slow-starting apps (without `startupProbe`) → restart loops
- Relying on `emptyDir` for data that must survive Pod recreation
- Not defining a `PodDisruptionBudget` for critical single-digit-replica workloads
- Running as root unnecessarily (skipping `securityContext`)

### Production Checklist

- [ ] Image tag pinned to a specific digest/version (never `latest`)
- [ ] `requests` and `limits` set for CPU and memory
- [ ] `readinessProbe`, `livenessProbe`, and (if slow-start) `startupProbe` defined
- [ ] `terminationGracePeriodSeconds` tuned to your app's shutdown time
- [ ] SIGTERM handled gracefully in app code; `preStop` hook if needed
- [ ] `securityContext`: non-root, `readOnlyRootFilesystem` where possible, capabilities dropped
- [ ] Secrets via `Secret` objects, not hardcoded env values
- [ ] Labels for `app`, `version`, `component` consistently applied
- [ ] `PodDisruptionBudget` defined for HA workloads
- [ ] Pod anti-affinity to spread replicas across nodes/zones
- [ ] Logs going to stdout/stderr (no reliance on writing to local files unless shipped via sidecar)
- [ ] Resource QoS intentionally chosen (`Guaranteed` for critical latency-sensitive workloads)

### Mental Model (the one-paragraph summary)

> A Pod is a **thin shared wrapper**, not a VM and not a container. It exists purely to let one or more tightly-coupled containers share a network identity (one IP, localhost between them) and storage, and to be scheduled/evicted/restarted as one atomic unit. Everything else — Deployments, StatefulSets, Jobs — is just a controller that creates, watches, and replaces Pods according to a policy. If you understand that **the Pod is a lifecycle+networking boundary, not a compute boundary**, all the rest (probes, QoS, termination, sidecars) falls into place as mechanisms serving that one idea.

---

That's the full Pods chapter — batched as requested for the masterclass. Let me know if you want me to continue to the next chapter/batch, or go deeper on any specific section here (e.g., a dedicated CNI networking deep-dive, or the scheduler's filtering/scoring algorithm internals).
