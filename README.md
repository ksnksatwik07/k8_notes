# Kubernetes Architecture — Complete Masterclass
## Part 1 of 5: Fundamentals, Cluster Concept & Complete Architecture Overview

*(This is a large masterclass — delivered in installments so each part gets full depth instead of being compressed. Part 1 covers topics 1–5 plus the full architecture diagram that every later part builds on. Say "continue" and I'll deliver Part 2: the component deep-dives — kube-apiserver through CoreDNS.)*

---

# 1. WHAT KUBERNETES IS AND WHY IT EXISTS

## Beginner explanation

Kubernetes (K8s) is a system for **running and managing containers across many machines automatically**. You describe *what you want* running (e.g., "3 copies of my web app, always"), and Kubernetes continuously works to make that true — restarting crashed containers, moving them to healthy machines, scaling them up or down, and routing traffic to them.

## Why it exists

Before Kubernetes, running containers at scale meant solving the same hard problems over and over by hand:
- Which machine should this container run on?
- What happens when that machine dies?
- How does one container find another?
- How do I roll out a new version without downtime?
- How do I scale up under load, and back down after?

Google had already solved these problems internally with a system called **Borg** (running billions of containers a week, for over a decade before Kubernetes existed). Kubernetes (started in 2014) is the open-source distillation of those lessons, donated to the newly-formed Cloud Native Computing Foundation (CNCF) in 2015.

## Intermediate explanation

Kubernetes is fundamentally an **orchestration platform built on a declarative control loop**: you tell it a *desired state*, and independent controllers continuously reconcile the *actual state* of the cluster toward it. This single idea — declare what you want, let control loops make it true, forever, without further instruction — is the architectural spine of literally everything else in this masterclass. Every component you'll learn about exists to serve some piece of this loop.

## Advanced internals preview

Under the hood, Kubernetes is a **distributed system of independent, loosely-coupled controllers**, all watching and reacting to a single shared source of truth (etcd, via the API server) — no component talks directly to another component. This design (detailed fully in Part 3) is what makes Kubernetes resilient: any individual controller can crash and restart without the others even noticing, because none of them depend on direct peer-to-peer communication.

## Fun fact
Kubernetes' name comes from the Greek word for "helmsman" or "pilot" — the person who steers a ship. The seven spokes on the Kubernetes logo are a nod to the project's original internal name, "Project Seven of Nine" (a Star Trek reference from the team that started it).

---

# 2. PROBLEMS KUBERNETES SOLVES

| Problem | Manual/pre-Kubernetes pain | Kubernetes' answer |
|---|---|---|
| **Placement** | Manually deciding which server runs which container | Scheduler automatically bin-packs Pods onto nodes with capacity |
| **Self-healing** | Someone has to notice a crash and restart it | kubelet + controllers detect and restart/reschedule automatically |
| **Scaling** | Manually spinning up more instances under load | Horizontal Pod Autoscaler adjusts replica count based on metrics |
| **Service discovery** | Hardcoding IPs that constantly change | Services + CoreDNS give stable names regardless of Pod churn |
| **Rolling updates** | Manually taking instances down one at a time | Deployments perform controlled, zero-downtime rollouts natively |
| **Configuration management** | Baking config into images, rebuilding for every environment | ConfigMaps/Secrets decouple config from images |
| **Multi-machine networking** | Manually wiring routes/firewalls between hosts | CNI plugins give every Pod a flat, routable network automatically |
| **Resource fairness** | One noisy app starving others on shared hardware | Requests/limits + QoS classes create predictable resource boundaries |

## Beginner example
Without Kubernetes: your app crashes at 3 AM, nobody notices until a customer complains, someone SSHs in and restarts it manually.
With Kubernetes: the kubelet notices the container died via its liveness check, and restarts it in seconds — often before anyone is even paged.

---

# 3. THE KUBERNETES CLUSTER CONCEPT

## What a cluster actually is

A **cluster** is a set of machines (physical or virtual), called **nodes**, working together as a single system, coordinated by a shared control plane. There are exactly two categories of node:

```
┌─────────────────────────────────────────────────────────────┐
│                     KUBERNETES CLUSTER                       │
│                                                                │
│   ┌───────────────────────┐    ┌───────────────────────┐    │
│   │    CONTROL PLANE       │    │      WORKER NODES       │    │
│   │  (the "brain")          │    │  (the "muscle")          │    │
│   │  makes decisions        │    │  actually run your      │    │
│   │  about the cluster      │    │  application containers │    │
│   └───────────────────────┘    └───────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**Beginner mental model:** think of the control plane as an air traffic control tower, and worker nodes as the actual airplanes. The tower doesn't fly anywhere — it decides where every plane should go and watches that they get there. The planes (nodes) do the actual work of carrying passengers (your containers).

## Why this split exists

Separating "deciding" from "doing" means the cluster's actual application workloads (running on worker nodes) can keep running even if the control plane briefly has a problem — existing Pods don't just vanish the instant the control plane hiccups (a critical fact explored fully in the Failure Scenarios part). It also means you can scale the two independently: add more worker nodes for more application capacity without touching the control plane at all, or scale control-plane components (in HA setups) without adding a single worker.

---

# 4. CONTROL PLANE VS WORKER NODES

## Control Plane — what lives here

| Component | One-line role |
|---|---|
| **kube-apiserver** | The front door — every single interaction with the cluster goes through it |
| **etcd** | The cluster's entire memory — the one source of truth, stored as key-value data |
| **kube-scheduler** | Decides which node a new Pod should run on |
| **kube-controller-manager** | Runs the built-in reconciliation loops (Deployments, ReplicaSets, Nodes, etc.) |
| **cloud-controller-manager** | Talks to your cloud provider's APIs (load balancers, volumes, node lifecycle) |

**None of these run your application containers.** Their entire job is deciding, recording, and coordinating — never executing your workloads directly.

## Worker Nodes — what lives here

| Component | One-line role |
|---|---|
| **kubelet** | The node's agent — talks to the API server, makes containers actually run |
| **kube-proxy** | Implements Service networking rules on this node |
| **Container Runtime** (containerd/CRI-O) | Actually pulls images and starts/stops containers |

**This is where your Pods physically execute** — CPU cycles, memory, and network I/O for your application all happen here, never on the control plane.

## Beginner vs Advanced framing

- **Beginner framing:** "control plane thinks, worker nodes do."
- **Advanced framing:** the control plane holds and reconciles **desired state**; worker nodes are simply the last-mile executors that the control plane's decisions get delivered to via the kubelet — every worker-node component ultimately exists to answer one question asked by the control plane: *"is reality matching what was declared?"*

## Diagram: where each component physically runs

```
┌──────────────────────── CONTROL PLANE NODE(S) ─────────────────────────┐
│                                                                          │
│   kube-apiserver     etcd     kube-scheduler                           │
│   kube-controller-manager     cloud-controller-manager                 │
│                                                                          │
└──────────────────────────────────┬───────────────────────────────────┘
                                    │  (all communication is API-server-mediated —
                                    │   see Part 3 for exactly how)
        ┌───────────────────────────┼───────────────────────────┐
        ▼                            ▼                            ▼
┌───────────────┐          ┌───────────────┐          ┌───────────────┐
│  WORKER NODE 1 │          │  WORKER NODE 2 │          │  WORKER NODE 3 │
│                │          │                │          │                │
│  kubelet       │          │  kubelet       │          │  kubelet       │
│  kube-proxy    │          │  kube-proxy    │          │  kube-proxy    │
│  container     │          │  container     │          │  container     │
│  runtime       │          │  runtime       │          │  runtime       │
│                │          │                │          │                │
│  [Pod] [Pod]   │          │  [Pod] [Pod]   │          │  [Pod]         │
└───────────────┘          └───────────────┘          └───────────────┘
```

## Real-world example

A production EKS/GKE/AKS cluster: the cloud provider **fully manages the control plane** for you (you never SSH into it, never see its nodes directly — it's a managed service) and bills you for it separately, while you manage your own worker nodes (or let a managed node group/Fargate-style service handle even that). This managed-control-plane model is now the overwhelming majority of real-world production Kubernetes usage — very few companies run their own bare-metal control plane anymore outside of on-prem or specialized use cases.

---

# 5. COMPLETE KUBERNETES ARCHITECTURE (THE FULL PICTURE)

This is the diagram every later part of this masterclass will refer back to. Study it before continuing — everything else is detail layered on top of this skeleton.

```
                                    ┌─────────────┐
                                    │    USER      │
                                    │  (kubectl,   │
                                    │  CI/CD, etc) │
                                    └──────┬──────┘
                                           │ HTTPS (REST/JSON over TLS)
                                           ▼
 ┌─────────────────────────── CONTROL PLANE ─────────────────────────────┐
 │                                                                         │
 │                         ┌───────────────────┐                         │
 │            ┌───────────▶│   kube-apiserver   │◀───────────┐           │
 │            │            │  (the ONLY thing    │            │           │
 │            │            │  that talks to etcd) │            │           │
 │            │            └─────────┬──────────┘            │           │
 │            │                      │                        │           │
 │            │                      ▼                        │           │
 │            │            ┌───────────────────┐              │           │
 │            │            │       etcd         │              │           │
 │            │            │ (cluster's single  │              │           │
 │            │            │  source of truth)  │              │           │
 │            │            └───────────────────┘              │           │
 │            │                                                 │           │
 │   watches  │                                                 │  watches  │
 │            │                                                 │           │
 │  ┌─────────┴────────┐                              ┌────────┴─────────┐│
 │  │  kube-scheduler    │                              │ kube-controller- ││
 │  │  (assigns Pods      │                              │ manager          ││
 │  │  to nodes)          │                              │ (Deployment,     ││
 │  └────────────────────┘                              │ ReplicaSet, Node ││
 │                                                        │ controllers...) ││
 │                                                        └──────────────────┘│
 │                                                        ┌──────────────────┐│
 │                                                        │cloud-controller- ││
 │                                                        │manager           ││
 │                                                        │(LBs, volumes,    ││
 │                                                        │ node lifecycle)  ││
 │                                                        └──────────────────┘│
 └────────────────────────────────┬────────────────────────────────────────┘
                                   │ every worker node's kubelet ALSO
                                   │ talks directly to the API server
                                   ▼
 ┌────────────────────────── WORKER NODE ─────────────────────────────────┐
 │                                                                          │
 │   ┌─────────────┐        ┌──────────────┐        ┌───────────────┐    │
 │   │   kubelet    │───────▶│  Container    │───────▶│  Pod(s)        │    │
 │   │ (node agent) │  CRI   │  Runtime      │        │  (your app     │    │
 │   │              │        │ (containerd)  │        │   containers)  │    │
 │   └─────────────┘        └──────────────┘        └───────────────┘    │
 │                                                                          │
 │   ┌─────────────┐                                                       │
 │   │  kube-proxy  │  ← programs this node's Service routing rules        │
 │   └─────────────┘     (iptables/IPVS/eBPF — see Networking guide)       │
 │                                                                          │
 └──────────────────────────────────────────────────────────────────────┘

              (CoreDNS runs as regular Pods, scheduled onto worker
               nodes like any workload — it is NOT a control-plane
               component, despite feeling foundational; more in Part 2)
```

## The single most important architectural fact in all of Kubernetes

**No component ever talks directly to another component.** kube-scheduler never calls kubelet. kube-controller-manager never calls kube-scheduler. Every single interaction — without exception — flows *through the kube-apiserver*, which is the only component permitted to read or write etcd. This is not a minor implementation detail; it is the reason Kubernetes can be resilient, extensible, and pluggable at all. Part 3 unpacks exactly how and why.

```
WRONG mental model (this is NOT how Kubernetes works):
  scheduler ──────▶ kubelet          (direct call — DOES NOT HAPPEN)

CORRECT mental model:
  scheduler ──▶ apiserver ──▶ etcd ◀── apiserver ◀── kubelet (watching)
  (scheduler WRITES a decision; kubelet independently WATCHES and reacts)
```

## Beginner vs Intermediate vs Advanced summary of this diagram

- **Beginner:** there's a "brain" (control plane) and "hands" (worker nodes), and they talk through one central hub (the API server).
- **Intermediate:** every component is an independent process with one job, communicating exclusively by reading/writing objects through the API server — nothing is hardwired to anything else.
- **Advanced:** this is a **watch-based, eventually-consistent, controller-per-concern architecture** — each controller runs its own independent reconciliation loop, subscribing to only the object types it cares about via the API server's watch mechanism (fully detailed in Part 3), which is what allows Kubernetes to add entirely new controllers (including your own custom ones) without ever modifying existing components.

---

## Common beginner misunderstanding (called out early, more in Part 3)

**"The scheduler places the Pod onto the node."** Not quite — the scheduler only **writes a decision** (which node a Pod *should* run on) back to the API server/etcd. It never contacts the node, never starts anything, never confirms the Pod is actually running. The kubelet on that node is the one that notices (via its own independent watch) that a Pod has been assigned to it, and only *then* actually creates the container. This separation — "decide" vs "execute," done by two components that never talk to each other directly — trips up nearly everyone learning Kubernetes for the first time, and is worth re-reading until it feels obvious.

---

## What's coming in Part 2

Part 2 goes component-by-component through the full checklist for each of: **kube-apiserver, etcd, kube-scheduler, kube-controller-manager, cloud-controller-manager, kubelet, kube-proxy, Container Runtime, and CoreDNS** — for every one: what it is, why it exists, what it does, where it runs, how it communicates, what data it handles, what happens if it fails, and a real-world example.

Say **"continue"** whenever you're ready.
# Kubernetes Architecture — Complete Masterclass
## Part 2 of 5: Component Deep-Dives (kube-apiserver → CoreDNS)

*(Continuing from Part 1. Every component below is covered against the same eight-point checklist: what it is, why it exists, what it does, where it runs, how it communicates, what data it handles, what happens if it fails, and a real-world example — plus troubleshooting commands and a fun fact for each.)*

---

# 6. kube-apiserver

## What it is
A stateless REST API server — the **single front door** to the entire cluster. Every read and every write to Kubernetes, from every source (`kubectl`, controllers, kubelets, CI/CD pipelines, other API servers in HA setups), goes through it.

## Why it exists
Without a single, authoritative gatekeeper, every component would need its own logic for authentication, authorization, validation, and storage access — and directly reading/writing etcd from a dozen different components would be a data-corruption and security nightmare. Centralizing this into one component keeps every other piece of Kubernetes simple: they only ever need to know how to talk HTTP to one place.

## What it does
1. Accepts REST requests (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) against Kubernetes objects
2. Authenticates the caller (who are you?)
3. Authorizes the request (are you allowed to do this?)
4. Runs admission control (should this specific request be permitted/mutated?)
5. Validates the object against its schema
6. Persists the result to etcd (the *only* component allowed to do this)
7. Notifies all watchers of the change

## Where it runs
On control-plane nodes, as a static Pod (or a systemd-managed binary, depending on install method) — in HA clusters, multiple replicas run behind a load balancer.

## How it communicates
- **Inbound:** HTTPS, REST, JSON (or Protobuf internally for performance) — TLS client-cert or bearer-token authenticated
- **Outbound:** to etcd via gRPC; nothing else — it never initiates calls to schedulers, controllers, or kubelets. They call *it*.

## What data it handles
Every Kubernetes object definition (Pods, Deployments, Services, ConfigMaps, Secrets, Nodes, custom resources) passes through it — it is the sole read/write gateway to cluster state.

## What happens if it fails
- **No new changes can be made to the cluster** — no `kubectl apply`, no new Pods scheduled, no scaling events processed
- **Already-running Pods keep running** — kubelets continue managing containers they already know about locally, since they don't need the API server for basic container liveness
- Controllers and the scheduler stall (their watches disconnect) but resume automatically once the API server returns, with no data loss (etcd itself is unaffected)
- In HA setups, other API server replicas absorb the load transparently

## Real-world example
Managed Kubernetes (EKS/GKE/AKS) runs multiple API server replicas behind the cloud provider's own load balancer, invisible to you — you interact with a single DNS endpoint that's actually load-balanced across several redundant instances.

## Troubleshooting
```bash
kubectl get --raw='/healthz'          # is the API server itself healthy?
kubectl cluster-info
kubectl get componentstatuses          # deprecated but still informative on some clusters
journalctl -u kube-apiserver           # on a self-managed control-plane node
```

## Fun fact
The API server is **completely stateless** — it holds no cluster data in memory that survives a restart (aside from short-lived caches). You could kill and restart every API server replica simultaneously, and as long as etcd is intact, the cluster's actual state is never lost — this statelessness is precisely what makes horizontal API server scaling trivial.

---

# 7. etcd

## What it is
A distributed, strongly-consistent **key-value store**, originally built at CoreOS, that serves as Kubernetes' entire persistent memory.

## Why it exists
Kubernetes needs one authoritative, crash-consistent place to remember everything — and it needs multiple control-plane replicas to always agree on that state even if some of them are temporarily unreachable. etcd is built specifically to guarantee this using the **Raft consensus algorithm**.

## What it does
Stores every Kubernetes object as a key, structured hierarchically:
```
/registry/pods/default/my-pod
/registry/deployments/default/my-app
/registry/secrets/default/db-credentials
```
Every read from `kubectl get` and every write from `kubectl apply` ultimately becomes a read or write against one of these keys.

## Where it runs
On control-plane nodes, typically co-located with the API server on the same machines in small clusters, or on **dedicated etcd-only nodes** in larger/HA production deployments (etcd is disk-I/O and latency sensitive — noisy neighbors on the same box measurably hurt it).

## How it communicates
gRPC, exclusively with the kube-apiserver — no other component is permitted to talk to etcd directly. Between etcd's own replicas (in an HA cluster, always an **odd** number — 3 or 5), it uses the Raft protocol to replicate writes and elect a leader.

## What data it handles
The complete, authoritative state of the cluster — every object's full specification and status. If etcd's data is gone, the cluster's memory of everything is gone, unless a backup exists.

## What happens if it fails
- If **a minority** of etcd replicas fail (e.g., 1 out of 3), the cluster keeps functioning normally — Raft only needs a **quorum** (majority) to keep serving reads and writes
- If **quorum is lost** (e.g., 2 out of 3 down), the API server can no longer write *or* reliably read — the cluster effectively freezes for any state changes, though already-running Pods on worker nodes continue operating untouched
- This is why etcd cluster sizing is always odd-numbered — 3 tolerates 1 failure, 5 tolerates 2 — an even number (like 4) doesn't buy you anything extra over the odd number below it, since quorum math is about majority, not raw count

## Real-world example
Losing etcd data with no backup is one of the most catastrophic possible Kubernetes incidents — it means the cluster forgets every Deployment, Service, Secret, and RBAC rule ever created, even though the actual running containers on worker nodes might survive briefly. This is precisely why etcd backup (`etcdctl snapshot save`) is a non-negotiable line item in every production Kubernetes runbook.

## Troubleshooting
```bash
ETCDCTL_API=3 etcdctl endpoint health --cluster
ETCDCTL_API=3 etcdctl endpoint status --write-out=table --cluster
ETCDCTL_API=3 etcdctl snapshot save backup.db
```

## Fun fact
etcd's name literally comes from Unix's `/etc` directory (the traditional home of system configuration) plus "d" for "distributed" — "distributed /etc."

---

# 8. kube-scheduler

## What it is
A control-plane component whose entire job is answering exactly one question, repeatedly, for every unscheduled Pod: **"which node should this run on?"**

## Why it exists
Without a scheduler, you'd have to manually decide (and keep re-deciding, as capacity shifts) which of potentially thousands of nodes should run each new Pod — an intractable, constantly-changing bin-packing problem that needs to be automated and fast.

## What it does (two phases)
1. **Filtering** — eliminate every node that *cannot* run this Pod (insufficient resources, taints without matching tolerations, node selectors/affinity not satisfied, port conflicts, etc.)
2. **Scoring** — rank every remaining candidate node using a set of scoring plugins (least-requested resources, spread across zones, node affinity preferences, etc.), then pick the highest-scoring node

```
Unscheduled Pod
      │
      ▼
┌─────────────┐     ┌─────────────┐
│  Filtering    │────▶│  Scoring     │────▶ winning node chosen
│ (hard rules)  │     │ (soft prefs) │
└─────────────┘     └─────────────┘
```

## Where it runs
Control-plane nodes, as a static Pod/binary — stateless like the API server, so multiple replicas can run in HA setups with leader election (only one is ever *active* at a time; others stand by).

## How it communicates
Exclusively through the API server: **watches** for Pods with an empty `.spec.nodeName`, and **writes** its decision back as a `Binding` — it never contacts a node or a kubelet directly.

## What data it handles
Pod specs (resource requests, affinity/anti-affinity, tolerations), and Node objects (capacity, allocatable resources, labels, taints) — pure decision-making, no execution.

## What happens if it fails
- Already-scheduled and already-running Pods are completely unaffected
- **New** Pods simply pile up in `Pending` state, unscheduled, until the scheduler recovers
- In HA clusters, the standby replica wins a new leader election almost immediately, so real-world impact is usually seconds, not minutes

## Real-world example
A common production customization: running a **second, specialized scheduler** (using `schedulerName` in the Pod spec) for workloads with unusual placement needs — e.g., batch/ML jobs using a custom scheduler that understands GPU topology far more precisely than the default scheduler's generic scoring plugins.

## Troubleshooting
```bash
kubectl get pods --field-selector=status.phase=Pending
kubectl describe pod <pod>          # Events show exactly why scheduling failed/succeeded
kubectl -n kube-system logs <scheduler-pod-name>
```

## Fun fact
The scheduler is fully **pluggable** since the "Scheduling Framework" was introduced — you can inject your own filtering/scoring logic as plugins without forking or replacing the whole component, which is how tools like the Kubernetes-native GPU/topology-aware schedulers are built today.

---

# 9. kube-controller-manager

## What it is
Not one controller, but a **single binary bundling dozens of independent control loops** — each one watches specific object types and works to reconcile actual state toward desired state.

## Why it exists
Bundling these into one process (rather than dozens of separate binaries) simplifies operations and deployment — though each internal controller still operates completely independently in logic, just sharing a process and leader-election mechanism for HA.

## What it does — key built-in controllers

| Controller | Watches | Reconciles |
|---|---|---|
| **Deployment controller** | Deployment objects | Creates/updates ReplicaSets to match desired rollout state |
| **ReplicaSet controller** | ReplicaSet objects | Creates/deletes Pods to match desired replica count |
| **Node controller** | Node heartbeats | Marks nodes NotReady/evicts Pods after a timeout when a node stops reporting |
| **Job controller** | Job objects | Ensures the right number of Pods run to completion |
| **Namespace controller** | Namespace deletions | Cleans up all objects within a deleted namespace |
| **ServiceAccount controller** | ServiceAccount objects | Auto-provisions default ServiceAccounts and tokens |

Each follows the identical reconciliation pattern (fully detailed in Part 3): **watch → compare desired vs actual → act → repeat, forever.**

## Where it runs
Control-plane nodes, static Pod/binary, with leader election across replicas in HA setups (only one active leader runs the actual reconciliation logic at a time, to avoid every replica fighting over the same objects).

## How it communicates
Exclusively via the API server — watches object types it cares about, writes changes (e.g., creating a Pod object) back through it.

## What data it handles
Every built-in workload/cluster-management object type: Deployments, ReplicaSets, Nodes, Jobs, Namespaces, ServiceAccounts, and more.

## What happens if it fails
- Existing Pods keep running untouched
- **No new reconciliation happens** — a Deployment scaled up won't actually get new Pods created, a crashed Pod under a ReplicaSet won't be replaced, a truly dead node won't get its Pods evicted and rescheduled elsewhere
- HA leader election typically fails over within seconds

## Real-world example
This is exactly the component responsible for the core promise "if I say 3 replicas, I always have 3 replicas" — the ReplicaSet controller inside kube-controller-manager is silently comparing "desired: 3" against "actual: 2" every reconciliation cycle and creating a replacement Pod the moment a discrepancy appears.

## Troubleshooting
```bash
kubectl -n kube-system get leases    # see which replica currently holds leadership
kubectl -n kube-system logs <controller-manager-pod>
kubectl get deployment <name> -o yaml   # compare .spec (desired) vs .status (actual)
```

## Fun fact
Custom controllers you write yourself (the entire premise of the "Operator pattern") follow the **exact same watch-reconcile loop** as these built-in ones — there is no architectural difference between a controller Kubernetes ships and one a third party writes; they're peers using the same public API.

---

# 10. cloud-controller-manager

## What it is
The component that isolates all **cloud-provider-specific** logic out of the core Kubernetes codebase into a separate, pluggable binary.

## Why it exists
Early Kubernetes had cloud-provider code baked directly into kube-controller-manager, which meant every cloud integration change required touching and re-releasing core Kubernetes. Splitting it out lets cloud providers (AWS, GCP, Azure, and others) ship and version their own integration independently, and lets on-prem/bare-metal clusters simply omit this component entirely.

## What it does — its own set of controllers

| Controller | Job |
|---|---|
| **Node controller** | Checks with the cloud API whether a node that stopped responding was actually deleted/terminated, not just network-partitioned |
| **Route controller** | Configures cloud-network routes so Pod traffic can flow between nodes, where required by the cloud's networking model |
| **Service controller** | Provisions/de-provisions actual cloud load balancers when you create a `type: LoadBalancer` Service |

## Where it runs
Control-plane nodes — in managed Kubernetes (EKS/GKE/AKS), this is entirely handled for you and invisible.

## How it communicates
To the API server (like every other controller) **and** outward to the cloud provider's own API (AWS API, GCP API, etc.) — the only control-plane component with a legitimate reason to make external API calls beyond the cluster itself.

## What data it handles
Node objects (for lifecycle correlation with actual cloud instances) and Service objects of type `LoadBalancer` (to provision the corresponding cloud resource).

## What happens if it fails
- Existing cloud load balancers keep working
- **New** `LoadBalancer` Services stay stuck with `EXTERNAL-IP: <pending>` indefinitely
- Nodes that actually terminated in the cloud won't be promptly cleaned up from the cluster's Node list

## Real-world example
This is the exact component responsible for the moment, in the Ingress guide, where a `type: LoadBalancer` Service's `EXTERNAL-IP` flips from `<pending>` to a real IP — that transition is the cloud-controller-manager's Service controller finishing its cloud API call to provision the actual load balancer.

## Fun fact
On bare-metal/on-prem clusters, this entire component is often simply **absent** — projects like MetalLB exist specifically to implement `LoadBalancer` Service support in environments with no cloud-controller-manager to provision one.

---

# 11. kubelet

## What it is
The **node agent** — a process running on every worker node (and control-plane nodes too, for their own static Pods) that is the sole bridge between "what the API server says should run here" and "what's actually running on this machine."

## Why it exists
Every worker node needs *something* locally responsible for actually pulling images, starting containers, watching their health, and reporting status back — without that, the control plane's decisions would just be abstract records with nothing to act on them.

## What it does
1. **Watches** the API server for Pods assigned to its own node (`spec.nodeName == this-node`)
2. Instructs the container runtime (via CRI, the Container Runtime Interface) to pull images and start containers
3. Continuously runs configured **liveness, readiness, and startup probes**
4. Reports Pod and node status back to the API server (this is what populates `kubectl get pods`)
5. Mounts volumes, injects ConfigMaps/Secrets as files/env vars into containers
6. Sends periodic **node heartbeats**, so the control plane knows this node is alive

## Where it runs
Directly on every node — as a systemd-managed binary (not a Pod itself; it's the thing that creates Pods).

## How it communicates
HTTPS to the API server (watch + report) and via the **CRI (Container Runtime Interface)**, a gRPC API, to whatever container runtime is installed (containerd, CRI-O). It never talks to the scheduler, other kubelets, or the controller-manager directly.

## What data it handles
Pod specs assigned to its node, container images, mounted volumes, probe results, and resource usage metrics for its node and Pods.

## What happens if it fails
- The kubelet crashing means: no new Pods can start on this node, no health checks are evaluated, and status stops being reported
- **Already-running containers keep running** — they're managed by the container runtime independently, which doesn't stop just because the kubelet supervising it briefly disappeared
- After a configurable grace period with no heartbeat, the node controller (in kube-controller-manager) marks the node `NotReady` and, after a further timeout, begins evicting/rescheduling its Pods elsewhere

## Real-world example
This is exactly the mechanism behind a Pod restarting after a failed liveness probe — the kubelet itself runs the probe locally, decides the container is unhealthy, and restarts it directly, with **no round-trip to the control plane needed** for that specific decision — a deliberate design choice that keeps basic self-healing working even during API server outages.

## Troubleshooting
```bash
systemctl status kubelet
journalctl -u kubelet -f
kubectl describe node <node>       # Conditions section shows kubelet-reported health
kubectl get pod <pod> -o wide      # confirms which node/kubelet owns this Pod
```

## Fun fact
The kubelet predates the name "Kubernetes" in a sense — its CRI abstraction is exactly why Kubernetes was able to drop direct Docker-specific integration (`dockershim`) in 1.24+ without breaking workloads: any runtime speaking CRI (containerd, CRI-O) is interchangeable from the kubelet's point of view.

---

# 12. kube-proxy

## What it is
A per-node network agent responsible for implementing **Service** routing rules on every node — covered in full mechanical depth in the Kubernetes Services and Networking guides; here's its architectural role specifically.

## Why it exists
Services need their virtual IP → real Pod IP translation enforced *somewhere* on every node, continuously, as Pods come and go — kube-proxy is the component that watches Services/EndpointSlices and keeps each node's local packet-forwarding rules (iptables/IPVS) in sync with reality.

## What it does
Watches Service and EndpointSlice objects via the API server, and programs the node's kernel packet-forwarding rules (iptables, IPVS, or is entirely replaced by eBPF in Cilium-style setups) accordingly.

## Where it runs
Every node, as a DaemonSet (a Pod itself — unusually, unlike kubelet, which is not a Pod).

## How it communicates
Watches the API server for Services/EndpointSlices; talks to the local kernel (iptables/IPVS netlink calls) to install rules — never contacts other nodes or components directly.

## What data it handles
Service definitions and EndpointSlices — never touches actual application traffic content, only routing rule metadata.

## What happens if it fails
Existing iptables/IPVS rules **remain in place and keep working** (the kernel enforces them independently of kube-proxy's process being alive) — but **new** Services or Pod IP changes stop being reflected, so Service routing gradually goes stale as Pods churn.

## Real-world example
Full mechanical detail already covered in the Services guide's Part 5 — this entry exists here purely to place kube-proxy correctly in the *architectural* picture: it's a worker-node DaemonSet, not a control-plane component, despite governing cluster-wide-feeling behavior.

## Troubleshooting
```bash
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
kubectl -n kube-system logs <kube-proxy-pod>
```

## Fun fact
kube-proxy can be entirely **disabled and replaced** (Cilium's "kube-proxy replacement mode") — it's optional infrastructure, not a hardcoded core requirement, which is only possible *because* of Kubernetes' pluggable, watch-based architecture.

---

# 13. Container Runtime

## What it is
The actual software that pulls container images and creates/manages the Linux primitives (namespaces, cgroups) that make a "container" real — containerd and CRI-O are the two dominant choices today.

## Why it exists
Kubernetes itself never runs containers — it delegates that entirely, through the CRI standard, so any CRI-compliant runtime is interchangeable.

## What it does
Pulls images from registries, unpacks image layers, creates the container's namespaces/cgroups, starts/stops the container process, and reports status back to the kubelet via CRI.

## Where it runs
On every node, as a systemd-managed daemon (not itself a Pod), directly beneath the kubelet.

## How it communicates
gRPC via CRI, exclusively with the local kubelet — no direct API server interaction, no cluster-wide awareness at all; entirely node-local in scope.

## What data it handles
Container images, running container processes, and the low-level cgroup/namespace configuration each container needs.

## What happens if it fails
No containers on that node can be started, stopped, or have their status queried — existing containers may keep running at the kernel level briefly (they're still real Linux processes), but the kubelet loses its ability to manage or observe them, which typically surfaces quickly as node/Pod status going stale or `Unknown`.

## Real-world example
The dockershim removal (Kubernetes 1.24) was purely a Container-Runtime-layer change — Docker Engine itself doesn't speak CRI natively, so a translation shim was needed until it was dropped in favor of runtimes (containerd, CRI-O) that speak CRI directly; this changed nothing about how Pods/Deployments/Services behaved, since that entire layer sits below anything application-facing.

## Troubleshooting
```bash
crictl ps                  # list containers, CRI-native, runtime-agnostic
crictl images
systemctl status containerd
```

## Fun fact
containerd itself was **originally part of Docker**, later donated to the CNCF and split out as a standalone project — it's the same container-running engine underneath both plain Docker and most modern Kubernetes clusters, just accessed via different APIs (Docker Engine API vs CRI).

---

# 14. CoreDNS

## What it is
The default cluster DNS server — resolving Service and Pod names to IPs cluster-wide (full mechanics already covered in the Networking and Services guides).

## Why it exists (architectural angle)
Placed here deliberately to correct a common misconception: **CoreDNS is not a control-plane component.** It runs as an ordinary Deployment, made of ordinary Pods, scheduled onto ordinary worker nodes by the ordinary scheduler — it just *feels* foundational because so much depends on DNS working.

## What it does
Watches Services (and optionally Endpoints/EndpointSlices) via the API server and serves DNS responses for `*.svc.cluster.local` queries from any Pod in the cluster.

## Where it runs
Worker nodes, as a regular Deployment in the `kube-system` namespace — typically 2 replicas by default for redundancy.

## How it communicates
Watches the API server (control-plane direction) and serves plain DNS (port 53, UDP/TCP) to any Pod that queries it (data-plane direction) — it straddles both worlds architecturally, which is exactly why it's listed last in this section as a bridge into the next.

## What data it handles
Service/Endpoint metadata only — hostnames and IPs, never application data.

## What happens if it fails
Existing established connections (already resolved to an IP) keep working. **Any new DNS lookup fails** — a very disruptive, cluster-wide issue in practice, since virtually every application does DNS lookups continuously (connection pools expiring, retries, new connections) — this is why CoreDNS running with too few replicas or too little CPU/memory is a classic, high-blast-radius production incident.

## Real-world example
Already covered in depth in the Networking guide's troubleshooting section — flagged here specifically to reinforce: **CoreDNS is a workload, not infrastructure**, from the scheduler's point of view; you can `kubectl scale` it, apply a PodDisruptionBudget to it, and troubleshoot it exactly like any other Deployment.

## Fun fact
CoreDNS's configuration (the "Corefile") uses a plugin-chain architecture directly inspired by Caddy's web server config — CoreDNS's creators also created Caddy, and CoreDNS itself started as a general-purpose DNS server before becoming Kubernetes' default.

---

## Component summary table (architectural placement)

| Component | Control plane or worker? | Runs as | Talks to |
|---|---|---|---|
| kube-apiserver | Control plane | Static Pod/binary | etcd (only), everything else calls it |
| etcd | Control plane | Static Pod/binary | kube-apiserver only |
| kube-scheduler | Control plane | Static Pod/binary | API server only |
| kube-controller-manager | Control plane | Static Pod/binary | API server only |
| cloud-controller-manager | Control plane | Static Pod/binary | API server + cloud provider API |
| kubelet | Worker node | systemd daemon (not a Pod) | API server + local container runtime (CRI) |
| kube-proxy | Worker node | DaemonSet (is a Pod) | API server + local kernel (iptables/IPVS) |
| Container Runtime | Worker node | systemd daemon (not a Pod) | Local kubelet only (CRI) |
| CoreDNS | Worker node | Deployment (is a Pod) | API server + any Pod doing DNS lookups |

---

## What's coming in Part 3

Part 3 covers topics 15–24: **Controllers, Pods/workloads as concepts, how all components communicate, the Kubernetes API, the declarative model, desired vs current state, reconciliation loops, watches, informers, and the full authentication/authorization/admission flow.** This is where the "no component talks directly to another" claim from Part 1 gets fully unpacked mechanically.

Say **"continue"** whenever you're ready.
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

# Kubernetes # Kubernetes Pod Troubleshooting — Complete Masterclass
## Part 1 of 4: CrashLoopBackOff, ImagePullBackOff, ErrImagePull, Pending, OOMKilled

*(This masterclass covers 15 failure modes at full depth — delivered in installments so each gets the complete 12-point treatment rather than being compressed. Part 1 covers the five most frequently encountered failures. Say "continue" for Part 2: CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted.)*

---

# THE UNIVERSAL TROUBLESHOOTING ENTRY POINT

Before diving into individual failures, every single one of them starts the same way:
```bash
kubectl get pods                    # spot the abnormal STATUS column
kubectl describe pod <pod>           # Events section — almost always names the exact problem
kubectl logs <pod>                    # what the CONTAINER itself said before dying
kubectl logs <pod> --previous          # logs from the PREVIOUS crashed instance (crucial for CrashLoopBackOff)
```
Every section below assumes you've already run these four commands — they're the universal first move, not repeated as "step 1" for every failure type below.

---

# 1. CrashLoopBackOff

## What it means
The container **starts, then exits** (crashes, or even exits cleanly with code 0 when it shouldn't), repeatedly — and Kubernetes is now waiting increasingly long **backoff** periods between each restart attempt rather than restarting instantly forever.

## What Kubernetes is doing internally
This is a **kubelet-local** behavior (Pods masterclass, Diagram D1) — no scheduler, no controller-manager involvement. The kubelet observes the container exit, and per `restartPolicy: Always` (the Deployment/ReplicaSet default), restarts it — but with **exponential backoff**: 10s, 20s, 40s, 80s... capped at 5 minutes between attempts, specifically to avoid hammering a fundamentally broken container in a tight, wasteful restart loop.
```
Container exits → kubelet waits (backoff) → restarts → exits again → wait LONGER → restart → ...
```

## Common causes
- Application crashes immediately on startup (missing config, unhandled exception, failed dependency connection)
- Wrong `command`/`args` — e.g., a command that runs once and exits, used on a Pod expecting a long-running process
- A missing environment variable or file the app requires to even initialize
- The main process legitimately finishing (exit 0) in a container Kubernetes expects to run forever

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     CrashLoopBackOff   7          12m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl logs my-app                 # current attempt (often empty if it crashed instantly)
kubectl logs my-app --previous       # THE crash's actual output — usually the real answer
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      ...
  Finished:     ...
Restart Count:  7
```
**Exit code matters a lot here:**
- `0` — the process exited cleanly, meaning your restartPolicy expectations and the app's actual behavior disagree (app thinks it's done; Kubernetes expects it to run forever)
- `1` (or other nonzero) — generic application error; the actual reason is almost always in `logs --previous`
- `137` — this is actually `OOMKilled` wearing a CrashLoopBackOff's clothes (§5) — always check `Reason` explicitly, not just the exit code

## Root-cause investigation
```bash
kubectl logs my-app --previous --timestamps
kubectl describe pod my-app | grep -A10 "Last State"
# reproduce locally if possible:
docker run --rm <same-image> <same-command>
```
The single highest-leverage move: **read `logs --previous` before doing anything else** — the overwhelming majority of CrashLoopBackOff cases are fully explained by the application's own final log lines before it died.

## Fixes
- Fix the actual application bug/misconfiguration revealed by the logs
- If it's a legitimately run-to-completion process, use a **Job** (Jobs guide), not a Deployment
- Add a `startupProbe` (§15) if the app just needs more time before health checks should even begin evaluating it

## Verification
```bash
kubectl get pod my-app -w
# RESTARTS stops climbing, STATUS settles to Running, READY reaches 1/1
```

## Prevention
- Fail fast and log clearly on startup — a silent crash with no log output is far harder to diagnose than one that prints exactly what config was missing
- Use `startupProbe` for slow-initializing apps so the kubelet doesn't prematurely judge them unhealthy during legitimate startup time

## Production example
A Node.js app crash-looped in production because a required environment variable (`DATABASE_URL`) was renamed in a ConfigMap update but the Deployment's `env` reference still pointed at the old key name — `logs --previous` showed `TypeError: Cannot read property 'connect' of undefined` within milliseconds of each restart, immediately pointing at the missing config rather than a code bug.

## Interview question
**Q: A Pod is in CrashLoopBackOff with exit code 0. Why is that unusual, and what does it suggest?**
A: Exit code 0 means the process believed it finished successfully — but `restartPolicy: Always` (typical under a Deployment) treats *any* exit as something to restart. This mismatch suggests either the workload should actually be a Job (finite, run-to-completion) rather than a Deployment, or the application has a bug causing it to terminate early when it should keep running.

---

# 2–3. ImagePullBackOff and ErrImagePull

## What they mean
`ErrImagePull` is the **first** failed attempt to pull a container image. `ImagePullBackOff` is what you see on **subsequent** attempts, once the kubelet starts backing off between retries — the same exponential-backoff relationship as CrashLoopBackOff, just applied to image pulls instead of container starts.

## What Kubernetes is doing internally
```
kubelet sees a Pod assigned to its node
    → calls the container runtime (via CRI) to pull the specified image
    → runtime attempts to contact the registry, authenticate, download layers
    → FAILS → kubelet reports ErrImagePull, schedules a retry with backoff
    → next failure → reports ImagePullBackOff, backoff continues to grow
```

## Common causes
- Typo in the image name or tag (`myapp:1.O` instead of `1.0` — a classic letter-O-vs-zero mistake)
- Image genuinely doesn't exist in the registry, or was deleted
- Private registry requiring authentication, with no (or wrong) `imagePullSecrets` configured
- Rate limiting from the registry (Docker Hub's anonymous pull limits are a very common real-world trigger)
- Network policy or firewall blocking egress from nodes to the registry

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     ImagePullBackOff   0          2m
```

## Commands to run
```bash
kubectl describe pod my-app
```

## How to interpret the output
```
Events:
  Warning  Failed     kubelet  Failed to pull image "myapp:1.O":
                                rpc error: code = NotFound desc = failed to pull and
                                unpack image "docker.io/library/myapp:1.O":
                                failed to resolve reference: myapp:1.O: not found
  Warning  Failed     kubelet  Error: ErrImagePull
  Normal   BackOff    kubelet  Back-off pulling image "myapp:1.O"
  Warning  Failed     kubelet  Error: ImagePullBackOff
```
The **exact error text after "Failed to pull image"** tells you precisely which failure mode you're in — `not found` (bad name/tag), `unauthorized`/`403` (auth problem), `connection refused`/`timeout` (network problem) all point in genuinely different directions.

## Root-cause investigation
```bash
# verify the image actually exists and is spelled correctly:
docker pull myapp:1.0          # try it manually, from a machine with equivalent network access
kubectl get pod my-app -o jsonpath='{.spec.containers[0].image}'   # confirm exactly what was requested
kubectl get pod my-app -o jsonpath='{.spec.imagePullSecrets}'       # confirm a pull secret is even attached
```

## Fixes
```bash
# for private registries:
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io --docker-username=me --docker-password=secret
```
```yaml
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: myregistry.io/myapp:1.0    # fix any typo here
```

## Verification
```bash
kubectl delete pod my-app        # if managed by a controller, a fresh Pod will retry immediately
kubectl get pod -w
```

## Prevention
- Pin exact image tags (never `:latest` in production — Pods masterclass-adjacent best practice) and verify them in CI before deployment
- Test `imagePullSecrets` in a staging namespace before relying on them in production
- Consider a pull-through cache/mirror to avoid public registry rate limits entirely

## Production example
A cluster's nodes suddenly couldn't pull any Docker Hub images after Docker Hub introduced anonymous pull rate limits — every new Pod scheduled that day hit `ErrImagePull` with a `429 Too Many Requests` message buried in the Events, resolved by adding authenticated `imagePullSecrets` (which have a much higher rate limit) cluster-wide via a default ServiceAccount patch.

## Interview question
**Q: What's the actual difference between ErrImagePull and ImagePullBackOff?**
A: They're the same underlying failure — ErrImagePull is the immediate, first-attempt failure state; ImagePullBackOff is what's reported once the kubelet starts throttling retry attempts with exponential backoff after repeated failures. Neither indicates a fundamentally different problem — they're two points on the same timeline.

---

# 4. Pending

## What it means
The Pod object exists in etcd, but **has not been scheduled to a node at all** (or, less commonly, is scheduled but can't start for some pre-container reason) — this is the Pod phase from the Pods masterclass §6–9, at its most literal.

## What Kubernetes is doing internally
```
Pod created → scheduler watches for Pods with empty .spec.nodeName
            → filtering phase eliminates unsuitable nodes
            → IF NO NODE SURVIVES FILTERING: Pod stays Pending, scheduler
              retries on backoff and whenever cluster state changes
              (Architecture masterclass, Part 4 §28)
```

## Common causes
- **Insufficient resources** cluster-wide for the Pod's requests (Resource Requests guide)
- **Taints without matching tolerations** — every node the Pod could otherwise fit on is tainted against it
- **Node selector/affinity rules** that no current node satisfies
- **No available PersistentVolume** matching a required PVC (Storage guide) — scheduling waits on binding
- Zero worker nodes exist yet (a brand-new cluster, or all nodes cordoned/draining)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS    RESTARTS   AGE
# my-app    0/1     Pending   0          5m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl get nodes
kubectl describe nodes | grep -A5 "Allocated resources"
```

## How to interpret the output
```
Events:
  Warning  FailedScheduling  0/5 nodes are available: 3 Insufficient cpu,
           2 node(s) had taint {dedicated: gpu}, that the pod didn't tolerate.
```
This message is **exhaustive and literal** — it tells you exactly how many nodes were rejected and for which specific reason, across every node in the cluster. Read it fully before guessing.

## Root-cause investigation
```bash
kubectl get pod my-app -o jsonpath='{.spec.tolerations}'
kubectl get pod my-app -o jsonpath='{.spec.nodeSelector}'
kubectl get pod my-app -o jsonpath='{.spec.containers[*].resources}'
kubectl describe nodes | grep -B2 Taints
```

## Fixes
- Reduce over-inflated resource requests, or add cluster capacity (more nodes, or trigger a cluster autoscaler)
- Add the required `tolerations` if the workload genuinely belongs on a tainted node pool
- Correct a `nodeSelector`/`affinity` rule that's impossible to satisfy (e.g., referencing a label that no node actually has, often a typo)

## Verification
```bash
kubectl get pod my-app -w
# STATUS moves from Pending to ContainerCreating to Running
```

## Prevention
- Right-size resource requests based on actual observed usage (Resource Requests guide) rather than guessing high "to be safe," which wastes schedulable capacity across the whole cluster
- Keep taints/tolerations and nodeSelectors documented and reviewed — these are easy to silently break during node pool migrations

## Production example
A team migrated from one node pool to another with different labels; their Deployment's `nodeSelector` still referenced the old pool's label, and every new Pod sat `Pending` indefinitely after the old pool was fully drained — `FailedScheduling`'s message explicitly listed "0/12 nodes match node selector," which immediately pointed at the stale selector.

## Interview question
**Q: Is a Pending Pod always a resource-capacity problem?**
A: No — resource insufficiency is one cause among several (taints/tolerations mismatches, node selector/affinity that no node satisfies, an unbound PVC, or literally zero nodes existing). The `FailedScheduling` event message always states the specific reason(s); assuming it's always "not enough CPU/memory" without reading the actual message is a common, avoidable diagnostic mistake.

---

# 5. OOMKilled

## What it means
*(Full internal mechanics already covered exhaustively in the Resource Limits guide, Part 3 — this section is the troubleshooting-flow-focused summary.)* A container exceeded its memory limit and was forcibly killed by the kernel's OOM killer, scoped to that container's own cgroup.

## What Kubernetes is doing internally
```
Container's memory usage → approaches cgroup memory.max
    → kernel reclaims what it can (silent, no kill yet)
    → still over → kernel OOM killer sends SIGKILL to the container's process
    → container exits with code 137 (128 + SIGKILL's signal number 9)
    → kubelet observes the exit, reports Reason: OOMKilled
    → restartPolicy governs what happens next (same restart mechanics as
      any container exit, per Pods masterclass Diagram D1)
```

## Common causes
- Memory limit set genuinely too low for real peak usage
- A memory leak — usage climbs steadily over the container's lifetime rather than stabilizing
- A sudden legitimate spike (large request payload, batch processing burst) exceeding a limit sized for steady-state only
- No limit set at all, but the **node itself** ran out of memory (node-level OOM, per the Resource Limits guide — more disruptive, can affect unrelated Pods)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS      RESTARTS   AGE
# my-app    0/1     OOMKilled   3          8m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl top pod my-app --containers      # if it's currently running, see live usage
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Started:      ...
  Finished:     ...
```
**Exit code 137 plus `Reason: OOMKilled` together are unambiguous** — this is never a coincidence or a generic crash; it's specifically the kernel's memory enforcement, distinct from every other failure in this guide.

## Root-cause investigation
```bash
kubectl describe pod my-app | grep -A3 "Limits\|Requests"
kubectl top pod my-app --containers        # compare against the limit, if still running long enough
# check historical usage if you have metrics retention (Prometheus, etc.) —
# was this a SLOW climb (leak) or a SUDDEN spike (load-driven)?
```

## Fixes
- Raise the memory limit to a realistic ceiling based on observed peak usage, with headroom
- Fix an actual memory leak if usage climbs without bound over time rather than stabilizing
- For genuinely bursty workloads, consider whether the request should be raised too (affects scheduling and QoS, per the Resource Requests/Limits guides), not just the limit

## Verification
```bash
kubectl get pod my-app -w
kubectl top pod my-app --containers -w
# usage should stabilize comfortably under the new limit, RESTARTS stops climbing
```

## Prevention
- Load-test with realistic peak traffic before setting production memory limits
- Alert on Pods approaching their memory limit (e.g., >80% sustained) **before** they get OOMKilled, not just after
- Use Guaranteed QoS (Resource Limits guide) for workloads where an OOMKill is especially costly/disruptive

## Production example
A Java application was OOMKilled repeatedly after a Kubernetes migration because its JVM heap flags were set assuming the *node's* total memory, not the container's cgroup limit — the JVM allocated far more heap than the container was actually permitted, guaranteeing an eventual OOMKill under any real load; the fix was explicit `-Xmx` flags sized against the container's own memory limit, not the host's.

## Interview question
**Q: A container is OOMKilled even though `kubectl top pod` showed usage well under its limit most of the time. What's the likely explanation?**
A: `kubectl top` reports periodic snapshots, not continuous monitoring — a brief, sharp memory spike between polling intervals can cross the limit and trigger an instant SIGKILL without ever being visible in average or snapshot-based metrics. This is exactly why memory limit violations have zero grace period, unlike CPU throttling: there's no "average was fine" defense once a single instant crosses the cgroup ceiling.

---

## What's coming in Part 2

Part 2 covers: **CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted** — each with the full 12-point treatment, realistic error messages, and production examples.

Say **"continue"** whenever you're ready. — Complete Masterclass
## Part 1 of 4: CrashLoopBackOff, ImagePullBackOff, ErrImagePull, Pending, OOMKilled

*(This masterclass covers 15 failure modes at full depth — delivered in installments so each gets the complete 12-point treatment rather than being compressed. Part 1 covers the five most frequently encountered failures. Say "continue" for Part 2: CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted.)*

---

# THE UNIVERSAL TROUBLESHOOTING ENTRY POINT

Before diving into individual failures, every single one of them starts the same way:
```bash
kubectl get pods                    # spot the abnormal STATUS column
kubectl describe pod <pod>           # Events section — almost always names the exact problem
kubectl logs <pod>                    # what the CONTAINER itself said before dying
kubectl logs <pod> --previous          # logs from the PREVIOUS crashed instance (crucial for CrashLoopBackOff)
```
Every section below assumes you've already run these four commands — they're the universal first move, not repeated as "step 1" for every failure type below.

---

# 1. CrashLoopBackOff

## What it means
The container **starts, then exits** (crashes, or even exits cleanly with code 0 when it shouldn't), repeatedly — and Kubernetes is now waiting increasingly long **backoff** periods between each restart attempt rather than restarting instantly forever.

## What Kubernetes is doing internally
This is a **kubelet-local** behavior (Pods masterclass, Diagram D1) — no scheduler, no controller-manager involvement. The kubelet observes the container exit, and per `restartPolicy: Always` (the Deployment/ReplicaSet default), restarts it — but with **exponential backoff**: 10s, 20s, 40s, 80s... capped at 5 minutes between attempts, specifically to avoid hammering a fundamentally broken container in a tight, wasteful restart loop.
```
Container exits → kubelet waits (backoff) → restarts → exits again → wait LONGER → restart → ...
```

## Common causes
- Application crashes immediately on startup (missing config, unhandled exception, failed dependency connection)
- Wrong `command`/`args` — e.g., a command that runs once and exits, used on a Pod expecting a long-running process
- A missing environment variable or file the app requires to even initialize
- The main process legitimately finishing (exit 0) in a container Kubernetes expects to run forever

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     CrashLoopBackOff   7          12m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl logs my-app                 # current attempt (often empty if it crashed instantly)
kubectl logs my-app --previous       # THE crash's actual output — usually the real answer
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      ...
  Finished:     ...
Restart Count:  7
```
**Exit code matters a lot here:**
- `0` — the process exited cleanly, meaning your restartPolicy expectations and the app's actual behavior disagree (app thinks it's done; Kubernetes expects it to run forever)
- `1` (or other nonzero) — generic application error; the actual reason is almost always in `logs --previous`
- `137` — this is actually `OOMKilled` wearing a CrashLoopBackOff's clothes (§5) — always check `Reason` explicitly, not just the exit code

## Root-cause investigation
```bash
kubectl logs my-app --previous --timestamps
kubectl describe pod my-app | grep -A10 "Last State"
# reproduce locally if possible:
docker run --rm <same-image> <same-command>
```
The single highest-leverage move: **read `logs --previous` before doing anything else** — the overwhelming majority of CrashLoopBackOff cases are fully explained by the application's own final log lines before it died.

## Fixes
- Fix the actual application bug/misconfiguration revealed by the logs
- If it's a legitimately run-to-completion process, use a **Job** (Jobs guide), not a Deployment
- Add a `startupProbe` (§15) if the app just needs more time before health checks should even begin evaluating it

## Verification
```bash
kubectl get pod my-app -w
# RESTARTS stops climbing, STATUS settles to Running, READY reaches 1/1
```

## Prevention
- Fail fast and log clearly on startup — a silent crash with no log output is far harder to diagnose than one that prints exactly what config was missing
- Use `startupProbe` for slow-initializing apps so the kubelet doesn't prematurely judge them unhealthy during legitimate startup time

## Production example
A Node.js app crash-looped in production because a required environment variable (`DATABASE_URL`) was renamed in a ConfigMap update but the Deployment's `env` reference still pointed at the old key name — `logs --previous` showed `TypeError: Cannot read property 'connect' of undefined` within milliseconds of each restart, immediately pointing at the missing config rather than a code bug.

## Interview question
**Q: A Pod is in CrashLoopBackOff with exit code 0. Why is that unusual, and what does it suggest?**
A: Exit code 0 means the process believed it finished successfully — but `restartPolicy: Always` (typical under a Deployment) treats *any* exit as something to restart. This mismatch suggests either the workload should actually be a Job (finite, run-to-completion) rather than a Deployment, or the application has a bug causing it to terminate early when it should keep running.

---

# 2–3. ImagePullBackOff and ErrImagePull

## What they mean
`ErrImagePull` is the **first** failed attempt to pull a container image. `ImagePullBackOff` is what you see on **subsequent** attempts, once the kubelet starts backing off between retries — the same exponential-backoff relationship as CrashLoopBackOff, just applied to image pulls instead of container starts.

## What Kubernetes is doing internally
```
kubelet sees a Pod assigned to its node
    → calls the container runtime (via CRI) to pull the specified image
    → runtime attempts to contact the registry, authenticate, download layers
    → FAILS → kubelet reports ErrImagePull, schedules a retry with backoff
    → next failure → reports ImagePullBackOff, backoff continues to grow
```

## Common causes
- Typo in the image name or tag (`myapp:1.O` instead of `1.0` — a classic letter-O-vs-zero mistake)
- Image genuinely doesn't exist in the registry, or was deleted
- Private registry requiring authentication, with no (or wrong) `imagePullSecrets` configured
- Rate limiting from the registry (Docker Hub's anonymous pull limits are a very common real-world trigger)
- Network policy or firewall blocking egress from nodes to the registry

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     ImagePullBackOff   0          2m
```

## Commands to run
```bash
kubectl describe pod my-app
```

## How to interpret the output
```
Events:
  Warning  Failed     kubelet  Failed to pull image "myapp:1.O":
                                rpc error: code = NotFound desc = failed to pull and
                                unpack image "docker.io/library/myapp:1.O":
                                failed to resolve reference: myapp:1.O: not found
  Warning  Failed     kubelet  Error: ErrImagePull
  Normal   BackOff    kubelet  Back-off pulling image "myapp:1.O"
  Warning  Failed     kubelet  Error: ImagePullBackOff
```
The **exact error text after "Failed to pull image"** tells you precisely which failure mode you're in — `not found` (bad name/tag), `unauthorized`/`403` (auth problem), `connection refused`/`timeout` (network problem) all point in genuinely different directions.

## Root-cause investigation
```bash
# verify the image actually exists and is spelled correctly:
docker pull myapp:1.0          # try it manually, from a machine with equivalent network access
kubectl get pod my-app -o jsonpath='{.spec.containers[0].image}'   # confirm exactly what was requested
kubectl get pod my-app -o jsonpath='{.spec.imagePullSecrets}'       # confirm a pull secret is even attached
```

## Fixes
```bash
# for private registries:
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io --docker-username=me --docker-password=secret
```
```yaml
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: myregistry.io/myapp:1.0    # fix any typo here
```

## Verification
```bash
kubectl delete pod my-app        # if managed by a controller, a fresh Pod will retry immediately
kubectl get pod -w
```

## Prevention
- Pin exact image tags (never `:latest` in production — Pods masterclass-adjacent best practice) and verify them in CI before deployment
- Test `imagePullSecrets` in a staging namespace before relying on them in production
- Consider a pull-through cache/mirror to avoid public registry rate limits entirely

## Production example
A cluster's nodes suddenly couldn't pull any Docker Hub images after Docker Hub introduced anonymous pull rate limits — every new Pod scheduled that day hit `ErrImagePull` with a `429 Too Many Requests` message buried in the Events, resolved by adding authenticated `imagePullSecrets` (which have a much higher rate limit) cluster-wide via a default ServiceAccount patch.

## Interview question
**Q: What's the actual difference between ErrImagePull and ImagePullBackOff?**
A: They're the same underlying failure — ErrImagePull is the immediate, first-attempt failure state; ImagePullBackOff is what's reported once the kubelet starts throttling retry attempts with exponential backoff after repeated failures. Neither indicates a fundamentally different problem — they're two points on the same timeline.

---

# 4. Pending

## What it means
The Pod object exists in etcd, but **has not been scheduled to a node at all** (or, less commonly, is scheduled but can't start for some pre-container reason) — this is the Pod phase from the Pods masterclass §6–9, at its most literal.

## What Kubernetes is doing internally
```
Pod created → scheduler watches for Pods with empty .spec.nodeName
            → filtering phase eliminates unsuitable nodes
            → IF NO NODE SURVIVES FILTERING: Pod stays Pending, scheduler
              retries on backoff and whenever cluster state changes
              (Architecture masterclass, Part 4 §28)
```

## Common causes
- **Insufficient resources** cluster-wide for the Pod's requests (Resource Requests guide)
- **Taints without matching tolerations** — every node the Pod could otherwise fit on is tainted against it
- **Node selector/affinity rules** that no current node satisfies
- **No available PersistentVolume** matching a required PVC (Storage guide) — scheduling waits on binding
- Zero worker nodes exist yet (a brand-new cluster, or all nodes cordoned/draining)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS    RESTARTS   AGE
# my-app    0/1     Pending   0          5m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl get nodes
kubectl describe nodes | grep -A5 "Allocated resources"
```

## How to interpret the output
```
Events:
  Warning  FailedScheduling  0/5 nodes are available: 3 Insufficient cpu,
           2 node(s) had taint {dedicated: gpu}, that the pod didn't tolerate.
```
This message is **exhaustive and literal** — it tells you exactly how many nodes were rejected and for which specific reason, across every node in the cluster. Read it fully before guessing.

## Root-cause investigation
```bash
kubectl get pod my-app -o jsonpath='{.spec.tolerations}'
kubectl get pod my-app -o jsonpath='{.spec.nodeSelector}'
kubectl get pod my-app -o jsonpath='{.spec.containers[*].resources}'
kubectl describe nodes | grep -B2 Taints
```

## Fixes
- Reduce over-inflated resource requests, or add cluster capacity (more nodes, or trigger a cluster autoscaler)
- Add the required `tolerations` if the workload genuinely belongs on a tainted node pool
- Correct a `nodeSelector`/`affinity` rule that's impossible to satisfy (e.g., referencing a label that no node actually has, often a typo)

## Verification
```bash
kubectl get pod my-app -w
# STATUS moves from Pending to ContainerCreating to Running
```

## Prevention
- Right-size resource requests based on actual observed usage (Resource Requests guide) rather than guessing high "to be safe," which wastes schedulable capacity across the whole cluster
- Keep taints/tolerations and nodeSelectors documented and reviewed — these are easy to silently break during node pool migrations

## Production example
A team migrated from one node pool to another with different labels; their Deployment's `nodeSelector` still referenced the old pool's label, and every new Pod sat `Pending` indefinitely after the old pool was fully drained — `FailedScheduling`'s message explicitly listed "0/12 nodes match node selector," which immediately pointed at the stale selector.

## Interview question
**Q: Is a Pending Pod always a resource-capacity problem?**
A: No — resource insufficiency is one cause among several (taints/tolerations mismatches, node selector/affinity that no node satisfies, an unbound PVC, or literally zero nodes existing). The `FailedScheduling` event message always states the specific reason(s); assuming it's always "not enough CPU/memory" without reading the actual message is a common, avoidable diagnostic mistake.

---

# 5. OOMKilled

## What it means
*(Full internal mechanics already covered exhaustively in the Resource Limits guide, Part 3 — this section is the troubleshooting-flow-focused summary.)* A container exceeded its memory limit and was forcibly killed by the kernel's OOM killer, scoped to that container's own cgroup.

## What Kubernetes is doing internally
```
Container's memory usage → approaches cgroup memory.max
    → kernel reclaims what it can (silent, no kill yet)
    → still over → kernel OOM killer sends SIGKILL to the container's process
    → container exits with code 137 (128 + SIGKILL's signal number 9)
    → kubelet observes the exit, reports Reason: OOMKilled
    → restartPolicy governs what happens next (same restart mechanics as
      any container exit, per Pods masterclass Diagram D1)
```

## Common causes
- Memory limit set genuinely too low for real peak usage
- A memory leak — usage climbs steadily over the container's lifetime rather than stabilizing
- A sudden legitimate spike (large request payload, batch processing burst) exceeding a limit sized for steady-state only
- No limit set at all, but the **node itself** ran out of memory (node-level OOM, per the Resource Limits guide — more disruptive, can affect unrelated Pods)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS      RESTARTS   AGE
# my-app    0/1     OOMKilled   3          8m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl top pod my-app --containers      # if it's currently running, see live usage
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Started:      ...
  Finished:     ...
```
**Exit code 137 plus `Reason: OOMKilled` together are unambiguous** — this is never a coincidence or a generic crash; it's specifically the kernel's memory enforcement, distinct from every other failure in this guide.

## Root-cause investigation
```bash
kubectl describe pod my-app | grep -A3 "Limits\|Requests"
kubectl top pod my-app --containers        # compare against the limit, if still running long enough
# check historical usage if you have metrics retention (Prometheus, etc.) —
# was this a SLOW climb (leak) or a SUDDEN spike (load-driven)?
```

## Fixes
- Raise the memory limit to a realistic ceiling based on observed peak usage, with headroom
- Fix an actual memory leak if usage climbs without bound over time rather than stabilizing
- For genuinely bursty workloads, consider whether the request should be raised too (affects scheduling and QoS, per the Resource Requests/Limits guides), not just the limit

## Verification
```bash
kubectl get pod my-app -w
kubectl top pod my-app --containers -w
# usage should stabilize comfortably under the new limit, RESTARTS stops climbing
```

## Prevention
- Load-test with realistic peak traffic before setting production memory limits
- Alert on Pods approaching their memory limit (e.g., >80% sustained) **before** they get OOMKilled, not just after
- Use Guaranteed QoS (Resource Limits guide) for workloads where an OOMKill is especially costly/disruptive

## Production example
A Java application was OOMKilled repeatedly after a Kubernetes migration because its JVM heap flags were set assuming the *node's* total memory, not the container's cgroup limit — the JVM allocated far more heap than the container was actually permitted, guaranteeing an eventual OOMKill under any real load; the fix was explicit `-Xmx` flags sized against the container's own memory limit, not the host's.

## Interview question
**Q: A container is OOMKilled even though `kubectl top pod` showed usage well under its limit most of the time. What's the likely explanation?**
A: `kubectl top` reports periodic snapshots, not continuous monitoring — a brief, sharp memory spike between polling intervals can cross the limit and trigger an instant SIGKILL without ever being visible in average or snapshot-based metrics. This is exactly why memory limit violations have zero grace period, unlike CPU throttling: there's no "average was fine" defense once a single instant crosses the cgroup ceiling.

---

## What's coming in Part 2

Part 2 covers: **CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted** — each with the full 12-point treatment, realistic error messages, and production examples.

Say **"continue"** whenever you're ready.


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


# Kubernetes ReplicaSets — Complete Mastery Guide
### From Absolute Beginner to Production-Grade Reconciliation Internals

---

# PART 1 — WHAT AND WHY

## 1. What is a ReplicaSet?

A ReplicaSet is a Kubernetes controller object whose entire job is: **ensure that exactly N Pods matching a given label selector are running, at all times.** Nothing more. It doesn't know about rollouts, versions, or history — just a number, a selector, and a template.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: myapp:1.0
```

## 2. Why ReplicaSets exist

A bare Pod has **zero self-healing** — if it crashes past what the kubelet can restart locally, or the node it's on dies, nothing brings it back (Architecture masterclass, Part 5, Diagram E). ReplicaSets exist to solve exactly one problem: **maintain a stable count of identical, interchangeable Pods**, continuously, without a human re-running `kubectl create` every time something goes wrong.

## 3. ReplicaSet vs Pod

| | Pod | ReplicaSet |
|---|---|---|
| Self-healing? | No | Yes — replaces failed/deleted Pods automatically |
| Represents | One running instance | A desired *count* of interchangeable instances |
| You typically create it directly? | Rarely in production | Rarely directly either — see §20/production guidance |

## 4. ReplicaSet vs Deployment

| | ReplicaSet | Deployment |
|---|---|---|
| Manages Pods directly? | Yes | No — manages ReplicaSets, which manage Pods |
| Rolling updates? | No native support | Yes — creates new ReplicaSets, scales old ones down |
| Rollback history? | No | Yes — keeps old ReplicaSets around, scaled to 0 |
| What you edit in practice | Almost never, directly | Almost always |

**The relationship is strictly layered:**
```
Deployment ──creates/manages──▶ ReplicaSet ──creates/manages──▶ Pod
```
A Deployment adds exactly one thing on top of a ReplicaSet: **the ability to change the Pod template over time, safely, with history.** A ReplicaSet alone has no concept of "update" at all — if you change its `.spec.template`, it does **not** touch any existing Pods; it only affects Pods created from that point forward (see §20).

---

# PART 2 — ARCHITECTURE

## 5. ReplicaSet architecture

A ReplicaSet is just another controller in the `kube-controller-manager` binary (Architecture masterclass, Part 2 §9), following the exact same universal pattern: watch → compare desired vs actual → act.

```
┌─────────────────────────────────────────────┐
│           ReplicaSet controller                │
│                                                 │
│   watches: ReplicaSet objects, Pod objects       │
│                                                 │
│   for each ReplicaSet:                            │
│     desired = .spec.replicas                       │
│     actual  = count of Pods matching .spec.selector  │
│               that are not terminating                │
│     if actual < desired: CREATE (desired - actual) Pods│
│     if actual > desired: DELETE (actual - desired) Pods│
└─────────────────────────────────────────────┘
```

## 6–8. Desired replicas, current replicas, ready replicas

Three **separate** numbers, all reported in `.status` — commonly confused:

```yaml
spec:
  replicas: 3          # DESIRED — what you asked for
status:
  replicas: 3           # CURRENT — how many Pod objects exist right now (any phase)
  readyReplicas: 2       # READY — how many of those are passing readiness probes (§Pod guide)
  availableReplicas: 2    # AVAILABLE — ready AND stable for minReadySeconds (Deployment-level concept, inherited here)
```

```bash
kubectl get rs my-app-rs
# NAME         DESIRED   CURRENT   READY   AGE
# my-app-rs    3         3         2       5m
```
**`CURRENT` reaching `DESIRED` does not mean the app is actually serving traffic** — a Pod can exist (counted in `CURRENT`) while still failing its readiness probe (not counted in `READY`). This is the ReplicaSet-level echo of the Pod guide's `Ready` condition discussion — same underlying mechanism, different vantage point.

## 9–10. Selectors and labels

The **only** mechanism connecting a ReplicaSet to its Pods is a label selector — there is no other pointer, no direct reference by name or UID for "which Pods belong to me."

```yaml
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app        # MUST match the selector above, or the API server REJECTS the object
```
**Validation rule worth knowing:** the API server enforces that `.spec.template.metadata.labels` is a **superset** of `.spec.selector.matchLabels` at creation time — you cannot create a ReplicaSet whose own Pod template wouldn't match its own selector; that would be a self-contradicting, permanently-Pod-creating-but-never-satisfied object, so Kubernetes rejects it outright.

```bash
kubectl get pods -l app=my-app --show-labels
```

---

# PART 3 — OWNERSHIP AND RECONCILIATION

## 11–12. Pod ownership and OwnerReferences

Every Pod created by a ReplicaSet carries an `ownerReferences` field pointing back to the ReplicaSet that created it — this is the actual, structural link (distinct from the selector, which is how the ReplicaSet *finds* Pods; `ownerReferences` is how Kubernetes' garbage collector knows a Pod *belongs* to something and should be cleaned up when that owner is deleted).

```yaml
# on the Pod object itself:
metadata:
  ownerReferences:
    - apiVersion: apps/v1
      kind: ReplicaSet
      name: my-app-rs
      uid: 3f29a1b2-...
      controller: true          # marks this as the MANAGING controller (vs. just a reference)
      blockOwnerDeletion: true   # prevents deleting the RS while this Pod still exists, if set
```
```bash
kubectl get pod <pod-name> -o jsonpath='{.metadata.ownerReferences}'
```
**Why this matters practically:** deleting a ReplicaSet with `kubectl delete rs my-app-rs` cascades to delete its owned Pods by default (via Kubernetes' garbage collector following `ownerReferences`) — `--cascade=orphan` explicitly detaches the Pods instead, leaving them running, ownerless, invisible to any reconciliation loop from that point on.

```bash
kubectl delete rs my-app-rs --cascade=orphan     # Pods survive, now unmanaged
kubectl delete rs my-app-rs                       # default — Pods are deleted too
```

## 13. Reconciliation — the complete internal process

```
1. ReplicaSet object changes (created, .spec.replicas edited, etc.)
   OR a Pod matching its selector changes (created, deleted, label changed off/onto match)
        │
        ▼
2. ReplicaSet controller's Informer cache is updated (Architecture masterclass, Part 3 §23)
        │
        ▼
3. Reconcile function runs for this specific ReplicaSet:
     a. List all Pods (from the LOCAL cache, not a live API call) matching .spec.selector
     b. Filter out Pods that are terminating (being deleted) — they don't count as "current"
     c. actual = count of remaining matching Pods
     d. desired = .spec.replicas
     e. if actual < desired: create (desired - actual) new Pods from .spec.template
        if actual > desired: delete (actual - desired) Pods (see deletion ordering below)
     f. write updated .status (replicas, readyReplicas, availableReplicas) back through
        the API server
        │
        ▼
4. Loop repeats — triggered again by the NEXT relevant watch event, not on a fixed timer
   (though a periodic full resync also happens as a safety net, independent of watches)
```

## Deletion ordering (scale-down specifics)
When `actual > desired`, the ReplicaSet controller doesn't delete arbitrarily — it uses a defined priority order, roughly: not-yet-scheduled Pods first, then Pods in earlier lifecycle phases, then (among equally-ready Pods) Pods that have restarted more, then younger Pods before older ones — the intent is to preserve the most stable, longest-running, healthiest Pods when scaling down, and sacrifice the least-established ones first.

---

# PART 4 — SCALING AND SELF-HEALING

## 14. Scaling

```bash
kubectl scale rs my-app-rs --replicas=5
```
Internally, this is nothing more than a `PATCH` to `.spec.replicas` — there's no separate "scaling" API or mechanism; it's the exact same reconciliation loop from §13, just triggered by the desired number changing instead of the actual count changing. Scaling up and self-healing (§15) are **the literal same code path**, as established in the Architecture masterclass's controller-reconciliation walkthrough.

## 15–18. Self-healing behaviors, explicitly, per cause

## 16. When a Pod dies (crash, OOMKilled)
```
kubelet reports Pod phase → Failed (or container restarts exhaust,
depending on restartPolicy)
    → ReplicaSet controller's watch fires
    → actual (2) < desired (3)
    → creates 1 new Pod
```
Typically resolved in single-digit seconds (Architecture masterclass, Part 4 §29's exact walkthrough applies unchanged here).

## 17. When a Pod is manually deleted
```bash
kubectl delete pod my-app-rs-x7z2p
```
**Identical reaction to §16** — the ReplicaSet controller has no concept of "who deleted it or why." It only ever sees "actual dropped below desired" and reacts identically regardless of cause (this is the level-triggered reconciliation principle from the Architecture masterclass, Part 3 §21, applied concretely). This is exactly why deleting one Pod under a ReplicaSet never reduces your replica count — a replacement appears almost immediately.

## 18. When a node fails
Follows the full node-failure timeline from the Architecture masterclass (Part 5, Diagram E): heartbeat lost → 40s to `NotReady`/`Unknown` → 5 minutes (default) to eviction → **only then** does the ReplicaSet controller see `actual < desired` and create replacements, scheduled onto healthy nodes. The ReplicaSet controller has zero special node-failure-specific logic — it's the **Node controller's** eviction that eventually produces the same "Pod count dropped" signal the ReplicaSet controller always reacts to identically.

## 19. ReplicaSet adoption of existing Pods

A ReplicaSet doesn't only create Pods — on every reconcile, it also **adopts** any existing, currently-unowned Pod that happens to match its selector (and releases Pods that no longer match, e.g., if you manually edit a Pod's labels off the match).

```bash
# create a bare Pod with a matching label BEFORE the ReplicaSet exists
kubectl run stray-pod --image=nginx --labels="app=my-app"
# now create the ReplicaSet with selector app=my-app, replicas: 3
kubectl apply -f my-app-rs.yaml
kubectl get pods -l app=my-app
# → the ReplicaSet ADOPTS "stray-pod" (sets ownerReferences on it) and
#   creates only 2 MORE Pods, not 3, since it counts the adopted one
#   toward its desired total
```
**This is a frequently surprising, real-world-relevant behavior:** if you ever manually create a Pod whose labels happen to match an existing ReplicaSet's selector, that ReplicaSet will silently claim it — this is a common cause of "why did my hand-created debug Pod suddenly get deleted when I scaled down" incidents.

## 20. ReplicaSet limitations (and why Deployments exist)

- **No update mechanism.** Editing `.spec.template` on a live ReplicaSet does **not** touch existing Pods — it only affects Pods created after the edit (e.g., ones created later due to scaling up, or replacing a crashed one). This means changing the image on a ReplicaSet directly produces an inconsistent mix of old and new Pods with no coordinated rollout at all.
- **No rollback / history.** There's no record of "what did this look like before" — once you've overwritten `.spec.template`, the previous version is simply gone.
- **No coordinated rollout strategy.** No `maxSurge`/`maxUnavailable`-style controlled transition (Architecture masterclass, Part 5, Diagram F) — that entire mechanism lives one layer up, in the Deployment controller, specifically because it needs to orchestrate **two ReplicaSets at once**, which a ReplicaSet itself has no concept of.

```
Directly editing a ReplicaSet's image:

  Before: 3 Pods, all image v1.0
  You edit .spec.template.spec.containers[0].image = v2.0
  Nothing happens to the 3 existing Pods — they keep running v1.0
  Only if one crashes (or you scale up) does a NEW Pod get created with v2.0
  Result: potentially a silent, uncoordinated mix of v1.0 and v2.0 Pods
          with zero rollout control — this is exactly the failure mode
          Deployments exist to prevent
```

## Why you (almost) always create a Deployment, never a ReplicaSet directly
Given the above, a bare ReplicaSet is essentially "the fixed-replica-count part of a Deployment, with none of the update safety." Unless you have a very specific reason to want a static, never-updated set of Pods with no rollout story at all, a Deployment gives you everything a ReplicaSet gives you **plus** safe updates and rollback, at zero additional cost — there's essentially no practical scenario in modern Kubernetes usage where hand-authoring a ReplicaSet is the better choice over a Deployment.

---

# PART 5 — DIAGRAMS

## ReplicaSet → Pods

```
┌─────────────────────── ReplicaSet "my-app-rs" ───────────────────────┐
│  spec.replicas: 3                                                       │
│  spec.selector: app=my-app                                               │
└──────────────────────────────┬───────────────────────────────────────┘
                                 │ creates/owns (ownerReferences)
        ┌────────────────────────┼────────────────────────┐
        ▼                         ▼                         ▼
┌───────────────┐        ┌───────────────┐        ┌───────────────┐
│  Pod (app=my-  │        │  Pod (app=my-  │        │  Pod (app=my-  │
│  app), v1.0     │        │  app), v1.0     │        │  app), v1.0     │
└───────────────┘        └───────────────┘        └───────────────┘
```

## Deployment → ReplicaSet → Pods

```
┌───────────────────── Deployment "my-app" ─────────────────────┐
│  spec.template: image v2.0 (latest edit)                          │
└───────────────────────────┬───────────────────────────────┬────┘
                              │ manages                       │ manages
                              ▼                                 ▼
              ┌───────────────────────┐          ┌───────────────────────┐
              │  ReplicaSet "my-app-    │          │  ReplicaSet "my-app-    │
              │  7d4f9c" (v1.0, OLD)     │          │  8b2e1a" (v2.0, NEW)     │
              │  replicas: 0              │          │  replicas: 3              │
              │  (kept for rollback        │          │  (currently active)         │
              │   history, per Architecture│          └───────────┬───────────┘
              │   masterclass Part 5 Diag F)│                       │ creates/owns
              └───────────────────────┘             ┌────────────┼────────────┐
                                                       ▼             ▼             ▼
                                                   Pod v2.0      Pod v2.0      Pod v2.0
```

---

# PART 6 — kubectl DEMONSTRATIONS

```bash
# View ReplicaSets
kubectl get rs
# NAME                DESIRED   CURRENT   READY   AGE
# my-app-rs            3         3         3       2m

# Inspect one in detail — see Events for creation/scaling history
kubectl describe rs my-app-rs
# Events show: "Created pod: my-app-rs-x7z2p" etc., one per Pod ever created by it

# Scale
kubectl scale rs my-app-rs --replicas=5
kubectl get rs my-app-rs -w    # watch DESIRED/CURRENT/READY converge live

# Delete a Pod and watch it get replaced
kubectl delete pod my-app-rs-x7z2p
kubectl get pods -o wide -w
# a BRAND NEW Pod name appears (per the Pod-restart-vs-replacement
# distinction from the Pods masterclass) — not the same Pod restarting

# Confirm ownership
kubectl get pod <any-pod-from-this-rs> -o jsonpath='{.metadata.ownerReferences[0].name}'
```

### Expected output walkthrough
After `kubectl delete pod my-app-rs-x7z2p`, within a couple of seconds `kubectl get pods -o wide` should show the deleted Pod gone and a new one (different name suffix, `AGE: 0s`-ish, fresh IP) present — this is the concrete, hands-on proof of the reconciliation loop from §13 running in real time.

---

# PART 7 — HANDS-ON EXERCISES (MINIKUBE)

### Exercise 1: Watch reconciliation happen live
```bash
minikube start
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: demo-rs
spec:
  replicas: 3
  selector:
    matchLabels: { app: demo }
  template:
    metadata: { labels: { app: demo } }
    spec:
      containers:
        - name: web
          image: nginx
EOF
kubectl get pods -l app=demo -w &
kubectl delete pod -l app=demo --field-selector=status.phase=Running -w 2>/dev/null | head -1
# watch a replacement appear within seconds
```

### Exercise 2: Prove editing the template doesn't touch existing Pods
```bash
kubectl patch rs demo-rs --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"nginx:1.25"}]'
kubectl get pods -l app=demo -o jsonpath='{.items[*].spec.containers[0].image}'
# → still shows the OLD image for every existing Pod — exactly per §20
kubectl scale rs demo-rs --replicas=4
kubectl get pods -l app=demo -o jsonpath='{.items[*].spec.containers[0].image}'
# → the ONE new Pod created by this scale-up uses the NEW image; the
#   original 3 are still on the old one — a live, hands-on demonstration
#   of the exact inconsistency Deployments exist to prevent
```

### Exercise 3: Adoption
```bash
kubectl run stray --image=nginx --labels="app=demo2"
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata: { name: demo2-rs }
spec:
  replicas: 3
  selector: { matchLabels: { app: demo2 } }
  template:
    metadata: { labels: { app: demo2 } }
    spec: { containers: [{ name: web, image: nginx }] }
EOF
kubectl get pods -l app=demo2
# → exactly 3 Pods total, "stray" among them, adopted rather than
#   duplicated — confirm with:
kubectl get pod stray -o jsonpath='{.metadata.ownerReferences}'
```

---

# PART 8 — TROUBLESHOOTING

**ReplicaSet shows `DESIRED: 3, CURRENT: 3, READY: 0`**
Pods exist but none are passing readiness — this is a Pod-level problem (probe misconfiguration, app not actually up yet), not a ReplicaSet problem; `kubectl describe pod` on one of them per the Pods masterclass' probe troubleshooting.

**Scaling up does nothing — CURRENT stays below DESIRED**
Check for Pods stuck `Pending` (`kubectl get pods -l <selector>`) — this is almost always a scheduling problem (insufficient node resources, per the Resource Requests guide), not a ReplicaSet malfunction; the ReplicaSet has already done its job (created the Pod objects) — the scheduler is what's stuck.

**A Pod I created manually keeps disappearing**
Check whether its labels accidentally match an existing ReplicaSet's selector — per §19, it may have been adopted and then deleted during a later scale-down/reconcile, exactly as if it were one of the ReplicaSet's "own" Pods.

**Deleting the ReplicaSet didn't delete its Pods**
Check whether `--cascade=orphan` was used (intentionally or by an automation script/tool default) — this is the expected, documented behavior of that flag, not a bug.

**Two ReplicaSets appear to be fighting over the same Pods**
Check for overlapping selectors — if two ReplicaSets' selectors both match the same Pods, they will genuinely compete, each trying to reconcile toward its own `replicas` count using a shared, overlapping Pod pool — this is a real, hand-authored-manifest misconfiguration (`kubectl describe rs` on both, compare `.spec.selector`), not something Kubernetes prevents automatically.

---

# PART 9 — PRODUCTION CONSIDERATIONS

1. **Never hand-author bare ReplicaSets for application workloads** — use Deployments; per §20, there's essentially no upside and a real, concrete downside (no safe update path).
2. **Never manually create Pods with labels that could collide with an existing ReplicaSet's selector** — per §19, this leads to silent adoption and unexpected deletion during routine scaling.
3. **Use precise, unique label selectors** — avoid generic labels like `app: web` shared loosely across unrelated ReplicaSets/Deployments in the same namespace; overlapping selectors (per the troubleshooting section) are a real, avoidable production incident class.
4. **Understand that `kubectl delete pod` under a ReplicaSet is not a safe way to "restart" an app** — it just churns one instance; for a coordinated restart of every Pod, use `kubectl rollout restart deployment/<name>` at the Deployment layer instead, which properly cycles through with readiness gating.
5. **Watch `readyReplicas` vs `replicas` in monitoring/alerting**, not just Pod count — a ReplicaSet fully "at desired count" can still be serving zero real traffic if readiness is failing cluster-wide.
6. **Set PodDisruptionBudgets at the workload level** (not on the ReplicaSet directly, though they do apply to its Pods) to prevent voluntary disruptions (node drains, cluster upgrades) from taking down too many replicas simultaneously.

---

# PART 10 — 20 INTERVIEW QUESTIONS

**Fundamentals**
1. What is the single responsibility of a ReplicaSet?
2. What's the architectural relationship between Deployment, ReplicaSet, and Pod?
3. Why does a bare Pod have no self-healing, while a ReplicaSet-managed one does?

**Mechanics**
4. What connects a ReplicaSet to its Pods — is it a direct reference or something else?
5. What is an `ownerReference`, and what's it used for?
6. What validation does the API server enforce between a ReplicaSet's selector and its Pod template labels, and why?
7. Explain the ReplicaSet's reconciliation loop, step by step.
8. What determines the order in which Pods are deleted during a scale-down?

**Behavior**
9. What happens when a Pod managed by a ReplicaSet is manually deleted?
10. Does the ReplicaSet controller behave differently for a crashed Pod versus a manually deleted one? Why or why not?
11. What is Pod adoption, and when does it happen?
12. What happens if you edit a running ReplicaSet's `.spec.template.spec.containers[0].image`?
13. Why does that edit not immediately affect existing Pods?

**Deployment relationship**
14. What does a Deployment add on top of a ReplicaSet?
15. Why does the Deployment controller need to manage two ReplicaSets simultaneously during a rollout, but a ReplicaSet itself never needs to manage two Pod generations?
16. Why are old ReplicaSets kept around (scaled to 0) instead of being deleted after a rollout?

**Production / scenario**
17. You notice `DESIRED: 5, CURRENT: 5, READY: 1` — what's your diagnostic path?
18. A manually created debug Pod keeps disappearing — what's the likely cause?
19. Why would you almost never create a bare ReplicaSet directly in a production manifest?
20. Two ReplicaSets seem to be repeatedly creating and deleting each other's Pods — what misconfiguration would cause this?

---

*This guide covers Kubernetes ReplicaSets from the desired-vs-actual reconciliation loop through ownership, adoption, scaling, and the precise reasons Deployments exist on top of them — the complete arc from beginner to core-contributor-level understanding of Kubernetes' most foundational self-healing primitive.*

# Kubernetes StatefulSets — Complete Masterclass (Beginner → Production)

---

## PART 1 — CONCEPTS

### 1. What a StatefulSet is

A **StatefulSet** is a Kubernetes controller for managing Pods that need a **stable, unique identity** — persistent name, persistent network address, and persistent storage — across restarts, rescheduling, and scaling. It's the workload API for anything where "which instance this is" matters, not just "how many instances exist."

```
Deployment mindset:  "I need 3 interchangeable copies of my app"
StatefulSet mindset: "I need pod-0, pod-1, pod-2 — each with its own
                       identity, its own disk, and each one matters individually"
```

### 2. Why StatefulSets exist

Deployments + ReplicaSets treat Pods as **cattle** — interchangeable, disposable, replaced with a fresh random name and IP whenever anything goes wrong. That works great for stateless web servers. It breaks for:

- **Databases** — a replica needs to know "am I the primary or replica-2?" and reconnect to *its own* disk, not a random one
- **Clustered systems** (Kafka, Zookeeper, Elasticsearch, Cassandra) — nodes reference each other by stable identity (`kafka-0`, `kafka-1`...) for quorum/consensus
- **Ordered bootstrap** — some clusters must start node 0 before node 1 (e.g., initializing a replica set primary)

StatefulSets provide the primitives these systems need: stable naming, stable DNS, stable per-Pod storage, and ordered rollout — without you having to hand-roll that logic in a controller.

### 3. Stateless vs Stateful Applications

| | Stateless | Stateful |
|---|---|---|
| Any replica can serve any request | ✅ | ❌ (often must hit specific node) |
| Losing a replica = no data loss | ✅ | ❌ (each Pod may own unique data) |
| Replicas are interchangeable | ✅ | ❌ (each has an identity/role) |
| Example | nginx, stateless API | MySQL, Kafka, Zookeeper, MongoDB |

### 4. Deployment vs StatefulSet

| | Deployment | StatefulSet |
|---|---|---|
| Pod naming | Random suffix (`myapp-7d9f8-x2k9p`) | Ordinal, stable (`myapp-0`, `myapp-1`) |
| Pod identity across restarts | New identity each time | **Same name** reused |
| Storage | Shared or none; PVCs not identity-bound | Each Pod gets **its own** PVC, reused on restart |
| Network identity | Pod IP changes, no stable DNS per-Pod | Stable DNS per Pod via headless Service |
| Creation order | Parallel, no order | Sequential (0, 1, 2, ... by default) |
| Deletion order | Parallel, no order | Reverse sequential (N, N-1, ..., 0) |
| Scaling | Any Pod, any order | Ordered, one at a time by default |
| Use case | Stateless web/API servers | Databases, message queues, clustered stores |

---

## PART 2 — CORE GUARANTEES

### 5-7. Stable Pod Identity, Stable Network Identity, Stable Storage — the Three Pillars

```
┌─────────────────────── StatefulSet "db" ───────────────────────┐
│                                                                    │
│   pod-0            pod-1            pod-2                        │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐                  │
│   │ Name:    │      │ Name:    │      │ Name:    │                 │
│   │  db-0    │      │  db-1    │      │  db-2    │                 │
│   │ DNS:     │      │ DNS:     │      │ DNS:     │                 │
│   │db-0.svc  │      │db-1.svc  │      │db-2.svc  │                 │
│   │ PVC:     │      │ PVC:     │      │ PVC:     │                 │
│   │data-db-0 │      │data-db-1 │      │data-db-2 │                 │
│   └─────────┘      └─────────┘      └─────────┘                  │
└────────────────────────────────────────────────────────────────┘
```

**Stable identity** means: if `db-1` dies (crash, node failure, eviction), Kubernetes recreates a Pod **named `db-1` again**, reattaches **the same PVC** (`data-db-1`), and it gets **the same DNS name** (`db-1.svc...`). Nothing about "who am I" changes — only the underlying physical placement might.

This is fundamentally different from a Deployment, where a dead Pod is gone forever and a totally new Pod (new name, new IP, and — unless using a shared/external volume — new/empty storage) takes its place.

### 8-9. Ordered Pod Creation / Deletion

**Default (`OrderedReady` pod management policy):**

```
Scale up 0→3:
  Create db-0 → wait until Running & Ready → Create db-1 → wait → Create db-2

Scale down 3→0 (or delete):
  Terminate db-2 → wait until fully gone → Terminate db-1 → wait → Terminate db-0
```

This matters for clustered systems that bootstrap in order (e.g., "node 0 initializes the replica set, others join it").

### 10. Pod Naming

Pattern: `<statefulset-name>-<ordinal>`, ordinals start at 0.

```
db-0, db-1, db-2, ...
```

Ordinal is baked into the Pod name **permanently** for that slot — scaling down removes the highest ordinal first, scaling back up recreates it with the same name and reattaches the same PVC if it still exists.

---

## PART 3 — NETWORKING

### 11. Headless Services

A **headless Service** (`clusterIP: None`) doesn't get a single virtual IP or load-balance traffic. Instead, DNS lookups for the Service name return the **individual Pod IPs directly** — and for StatefulSet Pods, each also gets its **own DNS A record**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db                     # this name becomes part of every Pod's DNS name
spec:
  clusterIP: None               # HEADLESS — no virtual IP, no load balancing
  selector:
    app: db
  ports:
  - port: 5432
```

**Why headless, not normal ClusterIP:** a normal Service load-balances across all matching Pods — great for stateless replicas, useless (even harmful) for a database where clients need to reach a **specific** replica (the primary, or a specific shard). Headless + StatefulSet gives you direct, individually addressable Pods via DNS.

```
Normal Service:  db.namespace.svc.cluster.local → 1 VIP → round-robin to any Pod
Headless Service: db.namespace.svc.cluster.local → returns ALL Pod IPs directly
                  db-0.db.namespace.svc.cluster.local → always db-0's IP specifically
```

### 12. DNS

For a StatefulSet named `db` with a headless Service also named `db` in namespace `prod`:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local

db-0.db.prod.svc.cluster.local   → resolves to db-0's current Pod IP
db-1.db.prod.svc.cluster.local   → resolves to db-1's current Pod IP
db.prod.svc.cluster.local        → resolves to ALL Pod IPs (headless "list" record)
```

**Internals:** the **CoreDNS** cluster DNS plugin generates these records dynamically from the **Endpoints/EndpointSlice** object associated with the headless Service, which is kept in sync with actual running Pods matching the selector, tagged with `hostname` = Pod name. This is what makes `db-0.db...` resolve correctly even though DNS records aren't manually created — it's derived live from cluster state.

**Why this matters:** application config for Kafka/MongoDB/etc. can hardcode `db-0.db.prod.svc.cluster.local:5432` as a **seed peer address**, and it keeps resolving correctly across Pod restarts/reschedules, because the *name* is stable even when the underlying IP changes.

---

## PART 4 — STORAGE

### 13. volumeClaimTemplates

The mechanism that gives each Pod its **own** PVC automatically:

```yaml
spec:
  volumeClaimTemplates:
  - metadata:
      name: data                          # becomes the mount reference name
    spec:
      accessModes: ["ReadWriteOnce"]        # one node can mount it read-write at a time
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
```

For a StatefulSet named `db` with 3 replicas, Kubernetes auto-creates:
```
data-db-0   (10Gi PVC, bound to some PV)
data-db-1   (10Gi PVC, bound to some PV)
data-db-2   (10Gi PVC, bound to some PV)
```

### 14. Persistent Storage — the identity binding

```
db-0  ──always mounts──►  data-db-0  ──bound to──►  PV (actual disk)
db-1  ──always mounts──►  data-db-1  ──bound to──►  PV (actual disk)
db-2  ──always mounts──►  data-db-2  ──bound to──►  PV (actual disk)
```

If `db-1`'s Pod is deleted (crash, eviction, node loss) and recreated, the **new** `db-1` Pod mounts the **same** `data-db-1` PVC — same data, no rebuild/resync needed for a simple restart.

⚠️ **Critical gotcha:** deleting a StatefulSet does **NOT** delete its PVCs by default (safety feature — avoids accidental data loss). You must manually delete `data-db-*` PVCs if you truly want to wipe storage. As of K8s 1.27+, `persistentVolumeClaimRetentionPolicy` gives you explicit control:

```yaml
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain     # or "Delete" — PVC fate when StatefulSet is deleted
    whenScaled: Retain      # or "Delete" — PVC fate when scaled down
```

---

## PART 5 — UPDATE STRATEGIES & SCALING

### 15. Pod Management Policies

```yaml
spec:
  podManagementPolicy: OrderedReady    # default — strict sequential create/delete
  # OR
  podManagementPolicy: Parallel         # all Pods created/deleted simultaneously
```

`Parallel` is useful when Pods **don't** need bootstrap ordering (e.g., Cassandra, which handles peer discovery itself) — speeds up scaling significantly.

### 16-18. Update Strategies: RollingUpdate, OnDelete, Partitioned

```yaml
spec:
  updateStrategy:
    type: RollingUpdate          # default
    rollingUpdate:
      partition: 0                # see below
```

#### RollingUpdate (default)
Updates Pods **in reverse ordinal order** (highest first): `db-2` → `db-1` → `db-0`, each fully replaced (new Pod, same name/PVC) before moving to the next. Unlike Deployments, there's no `maxSurge` — StatefulSets never create extra Pods beyond `replicas`, because ordinal identity is 1:1 with storage.

```
Update triggered (image v1→v2):
  db-2 terminated → recreated as v2 → Ready
  db-1 terminated → recreated as v2 → Ready
  db-0 terminated → recreated as v2 → Ready
```

#### OnDelete
Kubernetes does **not** automatically update anything — it only applies the new Pod template when **you manually delete** a specific Pod. Gives you full manual control over exactly when/which Pod gets updated — critical for databases where you want to manually verify replica health between each node's upgrade.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

```bash
kubectl delete pod db-2     # only db-2 gets recreated with new template; db-0, db-1 untouched
```

#### Partitioned Rolling Updates
```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2      # only Pods with ordinal >= 2 get updated
```

With `partition: 2` on a 5-replica StatefulSet, only `db-4` and `db-3` update on a template change; `db-2, db-1, db-0` stay on the old version untouched. This enables **canary-style rollouts** for stateful workloads: update the top partition, verify health, lower the partition number gradually to roll out further.

### 20. Scaling

```bash
kubectl scale statefulset db --replicas=5
# Creates db-3, then db-4 (in order, OrderedReady by default)

kubectl scale statefulset db --replicas=2
# Deletes db-4, then db-3 (reverse order) — PVCs data-db-3, data-db-4 RETAINED by default
```

Scaling back up later reattaches to the retained PVCs (`data-db-3` still has its old data) unless it was deleted or `persistentVolumeClaimRetentionPolicy.whenScaled: Delete` is set.

---

## PART 6 — LIMITATIONS & FAILURE BEHAVIOR

### 21. StatefulSet Limitations

- Storage must be backed by a **provisioner that supports dynamic provisioning** (or pre-created PVs) — StatefulSets don't magically create disks, they rely on `StorageClass`
- `ReadWriteOnce` volumes mean a Pod's storage is normally tied to being on **one node at a time** — moving a Pod to a new node requires the volume to detach/reattach (slow, and impossible mid-failure for some storage backends without manual intervention)
- No built-in cross-replica data replication — a StatefulSet gives you *stable identity and storage*, not database replication logic. **The app itself** (MySQL replication, Kafka ISR, MongoDB replica set protocol) must handle data sync between replicas
- Scaling down doesn't rebalance data — again, that's application-level logic
- Headless Service + DNS-based discovery requires your app to support this pattern (most clustered databases do, via config)

### 22. StatefulSet Failure Behavior

**Pod crash (container dies):** kubelet restarts it per `restartPolicy`, same node, same PVC — fast recovery, no rescheduling.

**Node failure (node goes NotReady):**
```
Node hosting db-1 dies
   │
   ▼
Node controller marks Node NotReady after timeout (~40s default)
   │
   ▼
Pod db-1 marked "Unknown"/Terminating, but NOT immediately deleted
   │  (K8s can't be SURE the old Pod/process is truly dead — risk of "split brain"
   │   if two db-1 instances end up running against the same storage)
   ▼
If using ReadWriteOnce block storage: the volume can't attach to a new node until
   the old node confirms detach, or you manually force-delete the Pod:
   kubectl delete pod db-1 --grace-period=0 --force
   │
   ▼
Once safely removed, StatefulSet controller recreates db-1 on a healthy node,
   reattaches data-db-1
```

This delay is a **deliberate safety trade-off** — StatefulSets favor consistency (never risk two processes writing to the same PVC) over instant availability. This is exactly why database StatefulSets often need manual intervention during real node failures, and why `terminationGracePeriodSeconds` and `--force` deletes matter operationally.

---

## PART 7 — DIAGRAMS

```
                   ┌──────────────────────┐
                   │   StatefulSet "db"    │
                   │  replicas: 3           │
                   │  serviceName: db        │
                   └───────────┬───────────┘
                               │ creates/manages, ordered
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
          ┌────────┐      ┌────────┐      ┌────────┐
          │  db-0   │      │  db-1   │      │  db-2   │
          │ PVC:    │      │ PVC:    │      │ PVC:    │
          │data-db-0│      │data-db-1│      │data-db-2│
          └────┬───┘      └────┬───┘      └────┬───┘
               │                │                │
               └────────────────┴────────────────┘
                                │
                    matched by selector
                                │
                     ┌──────────▼──────────┐
                     │  Headless Service    │
                     │  "db" clusterIP:None  │
                     └──────────┬──────────┘
                                │
                     CoreDNS generates per-Pod records
                                │
      db-0.db.ns.svc.cluster.local  db-1.db.ns.svc.cluster.local  db-2...
```

---

## PART 8 — DATABASE-SPECIFIC EXAMPLES

### MySQL (primary + replicas pattern)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql                 # MUST match the headless Service name — links DNS
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:8.0
        command:
        - bash
        - -c
        - |
          # mysql-0 = primary, mysql-1/2 = replicas — derive role from ordinal in Pod name
          ordinal=$(hostname | grep -o '[0-9]*$')
          echo "server-id=$((100 + ordinal))" > /mnt/conf.d/server-id.cnf
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef: { name: mysql-secret, key: root-password }
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      volumes:
      - name: conf
        emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
```

**Key idea:** the `initContainers` script uses the Pod's own ordinal (derived from `hostname`, which equals the Pod name, e.g. `mysql-1`) to compute a unique `server-id` — this pattern (deriving per-instance config from ordinal) is the single most common StatefulSet trick for databases.

### PostgreSQL (Patroni-style HA, conceptual)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels: { app: postgres }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      containers:
      - name: postgres
        image: postgres:16
        env:
        - name: POD_NAME
          valueFrom: { fieldRef: { fieldPath: metadata.name } }
        - name: PATRONI_KUBERNETES_NAMESPACE
          valueFrom: { fieldRef: { fieldPath: metadata.namespace } }
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: ["ReadWriteOnce"]
      resources: { requests: { storage: 50Gi } }
```

Real-world Postgres HA typically uses an operator (Patroni, Zalando's postgres-operator, CloudNativePG) built **on top of** StatefulSet primitives — leader election and failover logic live in the app/operator layer, not in StatefulSet itself.

### MongoDB (Replica Set)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: mongo
  replicas: 3
  selector:
    matchLabels: { app: mongo }
  template:
    metadata:
      labels: { app: mongo }
    spec:
      containers:
      - name: mongo
        image: mongo:7
        command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: data
          mountPath: /data/db
  volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: ["ReadWriteOnce"]
      resources: { requests: { storage: 30Gi } }
```

After all 3 Pods are `Running`, you initialize the replica set **once**, referencing stable DNS names:
```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-0.mongo.default.svc.cluster.local:27017" },
    { _id: 1, host: "mongo-1.mongo.default.svc.cluster.local:27017" },
    { _id: 2, host: "mongo-2.mongo.default.svc.cluster.local:27017" }
  ]
})
```
This is the canonical example of *why* stable per-Pod DNS matters: MongoDB stores these hostnames as permanent replica-set member identities.

### Kafka

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka
spec:
  clusterIP: None
  selector: { app: kafka }
  ports:
  - { port: 9092 }
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka
  replicas: 3
  podManagementPolicy: Parallel    # Kafka brokers handle their own peer discovery
  selector:
    matchLabels: { app: kafka }
  template:
    metadata:
      labels: { app: kafka }
    spec:
      containers:
      - name: kafka
        image: bitnami/kafka:3.7
        env:
        - name: KAFKA_CFG_BROKER_ID
          valueFrom: { fieldRef: { fieldPath: metadata.name } }  # id derived from pod name
        - name: KAFKA_CFG_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(POD_NAME).kafka.default.svc.cluster.local:9092"
        - name: POD_NAME
          valueFrom: { fieldRef: { fieldPath: metadata.name } }
        volumeMounts:
        - name: data
          mountPath: /bitnami/kafka
  volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: ["ReadWriteOnce"]
      resources: { requests: { storage: 100Gi } }
```

Kafka advertises **its own stable DNS name** to other brokers/clients — again relying entirely on the StatefulSet + headless Service DNS guarantee.

---

## PART 9 — HANDS-ON MINIKUBE LABS

```bash
minikube start --cpus=4 --memory=6144
```

### Lab 1 — Basic StatefulSet + stable identity
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  clusterIP: None
  selector: { app: web }
  ports: [{ port: 80 }]
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: ["ReadWriteOnce"]
      resources: { requests: { storage: 1Gi } }
EOF
kubectl get pods -w    # observe web-0, web-1, web-2 created IN ORDER
kubectl get pvc        # data-web-0, data-web-1, data-web-2
```

### Lab 2 — DNS resolution
```bash
kubectl run -it --rm dnsutils --image=busybox --restart=Never -- \
  nslookup web-0.web.default.svc.cluster.local
kubectl run -it --rm dnsutils --image=busybox --restart=Never -- \
  nslookup web.default.svc.cluster.local    # returns ALL pod IPs
```

### Lab 3 — Pod deletion → same identity
```bash
kubectl exec web-1 -- sh -c 'echo hello > /usr/share/nginx/html/index.html'
kubectl delete pod web-1
kubectl get pods -w                       # web-1 recreated with SAME name
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html   # "hello" — same PVC!
```

### Lab 4 — Scaling
```bash
kubectl scale statefulset web --replicas=5
kubectl get pods -w      # web-3, web-4 created IN ORDER after web-2
kubectl scale statefulset web --replicas=2
kubectl get pods -w      # web-4 deleted first, then web-3 (reverse order)
kubectl get pvc          # data-web-3, data-web-4 STILL EXIST (retained)
```

### Lab 5 — OnDelete update strategy
```bash
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"OnDelete"}}}'
kubectl set image statefulset/web nginx=nginx:1.26
kubectl get pods -o jsonpath='{.items[*].spec.containers[0].image}'   # unchanged!
kubectl delete pod web-1
kubectl get pod web-1 -o jsonpath='{.spec.containers[0].image}'        # now 1.26, only this one
```

### Lab 6 — Partitioned rollout (canary for stateful)
```bash
kubectl patch statefulset web -p \
  '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
kubectl set image statefulset/web nginx=nginx:1.27
kubectl get pods -w      # only web-2 (ordinal >= 2) updates; web-0, web-1 stay old
```

### Lab 7 — Simulated node failure / force delete
```bash
kubectl delete pod web-0 --grace-period=0 --force
kubectl get pods -w     # observe how fast StatefulSet recreates vs a graceful delete
```

---

## PART 10 — FAILURE SCENARIOS (summary table)

| Scenario | What happens | Recovery |
|---|---|---|
| **Pod deletion** (`kubectl delete pod`) | Pod terminated gracefully, StatefulSet recreates same name, reattaches same PVC | Automatic |
| **Node failure** | Pod stuck `Terminating`/`Unknown` until node confirmed dead or force-deleted; storage can't move until detached | May need `--force` delete + manual verification |
| **PVC failure** (PV lost/corrupted) | Pod `Pending`/`ContainerCreating` with `FailedMount` events; StatefulSet won't fabricate a new empty PVC automatically for that ordinal | Manual: restore from backup, recreate PVC, or restore via app-level replication resync |
| **Storage backend failure** (e.g., cloud EBS AZ outage) | All Pods whose PVCs live in that AZ stuck; if storage isn't zone-redundant, potential real data unavailability | Depends entirely on storage class's own HA/replication — StatefulSet doesn't fix storage backend problems |
| **Scaling down** | Highest-ordinal Pods removed first, in order; PVCs retained by default | Scaling back up reuses old PVC & data (unless deleted) |
| **Rolling update failure** (bad image on `db-2`) | `db-2` stuck `CrashLoopBackOff`; rollout **does not proceed** to `db-1`/`db-0` — later ordinals are protected | Fix image / `kubectl rollout undo` equivalent: patch back to old image, delete the bad Pod |

---

## PART 11 — COMPARISON: Deployment vs StatefulSet vs DaemonSet

| | Deployment | StatefulSet | DaemonSet |
|---|---|---|---|
| Purpose | Stateless, interchangeable replicas | Stable identity + storage for stateful apps | Exactly one Pod per (matching) node |
| Pod naming | Random suffix | Stable ordinal (`-0`, `-1`, ...) | One per node, node-tied |
| Scaling | Arbitrary `replicas` | Ordered, ordinal-based | Automatic — follows node count |
| Storage | Usually shared/none, not identity-bound | Per-Pod PVC via `volumeClaimTemplates` | Often `hostPath` for node-local access |
| Network identity | Shared Service VIP, no per-Pod DNS | Per-Pod stable DNS via headless Service | Runs on host network often, node-scoped |
| Update order | Parallel (rolling, with surge) | Sequential, reverse-ordinal | Per-node, rolling by node |
| Typical use | Web servers, stateless APIs | Databases, Kafka, Zookeeper, Elasticsearch | Log collectors (Fluentd), node monitors (node-exporter), CNI/kube-proxy agents |
| Deleted Pod replaced by | Totally new Pod (new name/IP/storage) | Same name/identity, same storage | Same node's Pod recreated when it reappears |

---

## PART 12 — INTERVIEW QUESTIONS & PRODUCTION BEST PRACTICES

### Interview Questions

1. What problem does StatefulSet solve that Deployment cannot?
2. What are the three "stable" guarantees a StatefulSet provides?
3. How does Pod naming differ between Deployment and StatefulSet?
4. What is a headless Service, and why is it required for StatefulSets?
5. Write out the full DNS name pattern for a StatefulSet Pod.
6. What does `clusterIP: None` actually change about Service behavior?
7. How are per-Pod PVCs created, and what naming pattern do they follow?
8. Does deleting a StatefulSet delete its PVCs? Why is this the default?
9. What controls PVC retention on delete/scale-down since K8s 1.27?
10. What's the default Pod creation/deletion order, and why does it matter for databases?
11. What's the difference between `OrderedReady` and `Parallel` pod management policy?
12. Explain `RollingUpdate` vs `OnDelete` update strategies.
13. What does `partition` do in a StatefulSet's rolling update, and why is it useful?
14. How would you implement a canary rollout for a stateful workload?
15. What happens to a StatefulSet Pod when its node fails unexpectedly?
16. Why does Kubernetes hesitate to reschedule a StatefulSet Pod immediately after node failure?
17. What does `--grace-period=0 --force` do, and what risk does it carry for stateful workloads?
18. How does an application (e.g., MySQL) typically derive a unique identity from its Pod ordinal?
19. Why is `ReadWriteOnce` access mode common for StatefulSet volumes, and what limitation does it impose?
20. Does StatefulSet handle data replication between replicas itself?
21. How does MongoDB rely specifically on StatefulSet DNS guarantees?
22. Why might Kafka use `podManagementPolicy: Parallel` instead of the default?
23. What happens during a rolling update if a new Pod at a high ordinal fails to become healthy?
24. How do you scale a StatefulSet down safely without losing needed data?
25. What's the risk of manually deleting a PVC tied to a live StatefulSet Pod?
26. Compare Deployment, StatefulSet, and DaemonSet along scaling behavior.
27. When would you choose a DaemonSet over a StatefulSet for infra tooling?
28. What role do Kubernetes Operators typically play on top of StatefulSets for databases?
29. What's a realistic RTO/RPO implication of using StatefulSets without an app-level HA/replication layer?
30. Design a production StatefulSet deployment strategy for a 3-node Kafka cluster, including update strategy choice and rationale.

### Production Best Practices

- Always pair a StatefulSet with a matching **headless Service** (`serviceName` field must match)
- Use `volumeClaimTemplates` with a real `storageClassName` backed by a dynamic provisioner appropriate to your cloud/on-prem storage
- Set `persistentVolumeClaimRetentionPolicy` explicitly — don't rely on ambiguous defaults for critical data
- Prefer **Operators** (Postgres Operator, Strimzi for Kafka, MongoDB Community Operator) over hand-rolled StatefulSets for complex clustered databases — they encode failover/backup logic StatefulSet alone doesn't provide
- Use `OnDelete` or `partition`-based rollout for production databases where you want manual gating between each node's upgrade
- Always test **node failure** and **force-delete** scenarios in staging before going to prod — this is the #1 surprising behavior new teams hit
- Monitor PVC binding status and `FailedMount` events proactively — storage issues in StatefulSets are silent until a Pod needs to (re)start
- Set resource `requests`/`limits` deliberately — database workloads are especially sensitive to CPU throttling and OOM kills
- Back up data **independently** of Kubernetes (StatefulSet + PVC is not a backup strategy)
- Document and rehearse your **manual recovery runbook** for the "node died, Pod stuck Terminating" scenario — this is not automatic, and pretending it is causes real incidents

---

That completes the StatefulSets chapter. Ready to continue to the next batch in the masterclass sequence — Services & Networking, ConfigMaps/Secrets, or wherever the approved plan takes us next — whenever you want to proceed.

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

# Kubernetes Jobs — Complete Mastery Guide
### From Absolute Beginner to Production Batch-Processing Expertise

---

# PART 1 — WHAT AND WHY

## What is a Job?

A Job is a Kubernetes controller that runs Pods to **completion**, rather than keeping them running forever. Where a Deployment's promise is "always have N Pods running," a Job's promise is "run this work until it finishes successfully N times, then stop."

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
        - name: hello
          image: busybox
          command: ["echo", "Hello from a Job"]
      restartPolicy: Never
```

## Job vs Deployment

| | Deployment | Job |
|---|---|---|
| Goal | Keep N Pods running **forever** | Run Pods until N **completions** happen, then stop |
| Pod exits successfully | Treated as a **failure** to be replaced! | This is the entire *point* — success is the desired end state |
| Use case | Long-running services (web servers, APIs) | Finite tasks (migrations, batch jobs, backups) |
| Rolling updates | Yes | Not applicable — there's no "running" state to roll |

**This is the single most important conceptual distinction**: a Deployment's ReplicaSet treats *any* container exit as something to immediately replace, whether success or failure — that's correct for a web server (it should never exit) but catastrophically wrong for a database migration script (it's *supposed* to exit, successfully, exactly once). Jobs exist because Kubernetes needs a controller whose reconciliation logic understands "successful completion" as a terminal, desired state — not a gap to fill.

## Job vs Pod

A bare Pod with `restartPolicy: Never` that fails just... stays failed, forever, with nobody retrying it. A Job wraps that same idea with **retry logic** (`backoffLimit`), **parallel execution** (`parallelism`/`completions`), and **completion tracking** — turning a single fire-and-forget Pod into a managed, retryable unit of work.

---

# PART 2 — JOB LIFECYCLE

```
Job created
    │
    ▼
Pod(s) created from .spec.template
    │
    ├──▶ Pod succeeds (exit 0) ──▶ counted toward .spec.completions
    │                                  │
    │                                  ▼
    │                        completions target met? ──▶ YES ──▶ Job: Complete
    │                                  │
    │                                  NO ──▶ create another Pod (if under parallelism)
    │
    └──▶ Pod fails (non-zero exit) ──▶ counted toward .spec.backoffLimit
                                           │
                                           ▼
                                 backoffLimit exceeded? ──▶ YES ──▶ Job: Failed
                                           │
                                           NO ──▶ create a replacement Pod (with backoff delay)
```

```bash
kubectl get job hello-job
# NAME         COMPLETIONS   DURATION   AGE
# hello-job    1/1           3s         10s
kubectl get job hello-job -o jsonpath='{.status.conditions}'
# type: Complete, status: "True"   (or type: Failed)
```

---

# PART 3 — CORE FIELDS

## completions

How many **successful** Pod completions are needed before the Job itself is considered done.
```yaml
spec:
  completions: 5    # need 5 successful runs total
```
Without this field, the default is `1` — one successful Pod, and the Job is complete.

## parallelism

How many Pods may run **at the same time** while working toward `completions`.
```yaml
spec:
  completions: 10
  parallelism: 3     # up to 3 Pods running simultaneously, until 10 total have succeeded
```
```
Timeline with completions=10, parallelism=3:

t=0   [Pod1][Pod2][Pod3]                    ← 3 running
t=1   [Pod1✓][Pod2][Pod3][Pod4]              ← Pod1 done, Pod4 starts to keep 3 running
t=2   [Pod2✓][Pod3][Pod4][Pod5]              ← same pattern continues...
...   until 10 total successes are reached
```

## backoffLimit

How many **Pod failures** are tolerated before the Job gives up and marks itself `Failed`.
```yaml
spec:
  backoffLimit: 4     # default is 6 if omitted
```
Each retry waits with **exponential backoff** (10s, 20s, 40s... capped at 6 minutes) between attempts — this prevents a rapidly-failing Job from hammering the cluster with retry attempts in a tight loop.

## restartPolicy (Job-specific behavior)

```yaml
spec:
  template:
    spec:
      restartPolicy: OnFailure    # or: Never — NEVER "Always" (invalid for Jobs)
```
- `OnFailure` — the **same Pod** restarts its container in place on failure (kubelet-local restart, per the Pods masterclass' Diagram D1) — counts toward the container's own restart count, not directly toward `backoffLimit` in the same way
- `Never` — a failed Pod is left as-is (not restarted in place); the **Job controller** creates a brand-new replacement Pod instead — this consumes one unit of `backoffLimit`
`restartPolicy: Always` is **rejected outright** by the API server for Jobs — it makes no sense for a controller whose entire purpose is recognizing completion.

## activeDeadlineSeconds

A hard wall-clock timeout for the **entire Job**, regardless of retries remaining.
```yaml
spec:
  activeDeadlineSeconds: 300    # kill everything and mark Failed after 5 minutes, no matter what
```
This takes priority over `backoffLimit` — even if retries are still available, exceeding this deadline terminates the Job immediately with reason `DeadlineExceeded`.

## successfulJobsHistoryLimit / failedJobsHistoryLimit

These apply specifically to **CronJobs** (Jobs created on a recurring schedule) — controlling how many completed/failed Job objects are kept around for inspection before being garbage collected.
```yaml
# on a CronJob:
spec:
  successfulJobsHistoryLimit: 3   # keep the last 3 successful Job objects
  failedJobsHistoryLimit: 1        # keep only the last 1 failed Job object
```
Without limits, every scheduled run's Job object would accumulate forever, cluttering `kubectl get jobs` and wasting etcd storage on objects nobody needs after the fact.

---

# PART 4 — PARALLEL AND INDEXED JOBS

## Parallel Jobs — two distinct patterns

**Pattern 1: fixed completion count** (`completions` + `parallelism`, as above) — a known total amount of identical work, split across concurrent workers, e.g., "process these 100 files, 5 at a time."

**Pattern 2: work queue** (`parallelism` set, `completions` omitted) — workers pull tasks from an external queue (e.g., Redis, SQS) until the queue is empty, then each exits successfully; the Job is done when **all** running Pods have exited successfully and none are left — the *coordination* of "is there more work" lives outside Kubernetes, in the queue itself, not in the Job's own field.

## Indexed Jobs

Each Pod gets a unique, fixed index (`0` to `completions - 1`), injected as an environment variable — letting each parallel worker know exactly which slice of work is "its" slice, without any external coordination system at all.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-job
spec:
  completions: 5
  parallelism: 5
  completionMode: Indexed        # <- the key field enabling this pattern
  template:
    spec:
      containers:
        - name: worker
          image: myimage
          command: ["sh", "-c", "echo Processing shard $JOB_COMPLETION_INDEX"]
          env:
            - name: JOB_COMPLETION_INDEX
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
      restartPolicy: Never
```
**Real-world value:** this is precisely how large-scale, embarrassingly-parallel workloads (distributed ML training shards, splitting a big dataset into N fixed partitions) get built natively in Kubernetes — each of the 5 Pods above knows, from its own index alone, exactly which 1/5th of the total dataset it's responsible for, with zero external queue or coordination service required.

---

# PART 5 — COMPLETION, CLEANUP, AND THE TTL CONTROLLER

## Job completion

A Job's `.status.conditions` reaches `type: Complete, status: "True"` the instant `completions` successful Pod exits have been observed — the Job object itself (and, by default, its completed Pods) **stick around indefinitely** unless explicitly cleaned up.

## Job cleanup and the TTL controller

Leaving completed Jobs (and their Pods) around forever accumulates clutter — the `ttlSecondsAfterFinished` field delegates automatic cleanup to the built-in **TTL controller**:

```yaml
spec:
  ttlSecondsAfterFinished: 3600    # delete this Job (and its Pods) 1 hour after it finishes
```
Internally, this is just another ordinary controller (Architecture masterclass, Part 2 §9 pattern) watching for Jobs with both a `Complete`/`Failed` condition **and** this field set, computing "has enough time passed since completion," and issuing a cascading delete (following `ownerReferences`, ReplicaSets guide §11–12 pattern) once the TTL expires.

```bash
kubectl get jobs
kubectl get job hello-job -o jsonpath='{.status.completionTime}'
# (compare against current time to manually estimate TTL-controller timing)
```

---

# PART 6 — YAML COOKBOOK (ALL SIX, LINE-BY-LINE)

## 1. Simple Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: simple-job
spec:
  template:                     # the Pod template — identical shape to a bare Pod's spec
    spec:
      containers:
        - name: task
          image: busybox
          command: ["echo", "done"]
      restartPolicy: Never       # REQUIRED for Jobs — Always is invalid
```
- `restartPolicy: Never` — on failure, the Job controller creates a fresh replacement Pod rather than the kubelet restarting the container in place
- No `completions`/`parallelism` specified → both default to `1` — run once, succeed once, done

## 2. Job with retries
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: retry-job
spec:
  backoffLimit: 5              # tolerate up to 5 failures before giving up
  template:
    spec:
      containers:
        - name: flaky-task
          image: myimage
          command: ["sh", "-c", "exit 1"]   # deliberately always fails, for demonstration
      restartPolicy: Never
```
- `backoffLimit: 5` — the Job will attempt this task up to 6 times total (initial + 5 retries) before marking itself `Failed`, with exponential backoff between attempts

## 3. Parallel Job (work queue style)
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: queue-worker-job
spec:
  parallelism: 4                # up to 4 workers running concurrently
  template:                      # completions OMITTED — queue-driven completion
    spec:
      containers:
        - name: worker
          image: my-queue-worker
          env:
            - name: QUEUE_URL
              value: "redis://queue-service:6379/tasks"
      restartPolicy: OnFailure
```
- No `completions` — each worker exits (successfully) once it finds the queue empty; the Job is done once all 4 have exited successfully with nothing left to pick up
- `restartPolicy: OnFailure` — a transient failure (e.g., a dropped connection) restarts the container in place rather than abandoning that worker slot entirely

## 4. Job with multiple completions
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: fixed-batch-job
spec:
  completions: 10                # need 10 total successes
  parallelism: 2                  # only 2 running at once
  template:
    spec:
      containers:
        - name: processor
          image: my-batch-processor
      restartPolicy: Never
```
- `completions: 10` + `parallelism: 2` — a known, fixed amount of work (10 units), processed 2 at a time, sequentially cycling through until all 10 have succeeded

## 5. Indexed Job
*(Full example already given in Part 4 — reproduced here for cookbook completeness.)*
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sharded-job
spec:
  completions: 5
  parallelism: 5
  completionMode: Indexed
  template:
    spec:
      containers:
        - name: shard-worker
          image: my-shard-processor
          command: ["sh", "-c", "process-shard.sh $JOB_COMPLETION_INDEX"]
      restartPolicy: Never
```

## 6. Job with resource limits
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: bounded-job
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 600      # hard 10-minute ceiling on the WHOLE job
  ttlSecondsAfterFinished: 1800    # auto-cleanup 30 minutes after completion
  template:
    spec:
      containers:
        - name: task
          image: myimage
          resources:
            requests:
              cpu: "500m"           # Resource Requests guide's mechanics apply unchanged
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
      restartPolicy: Never
```
- `resources` — governed by exactly the same scheduler bin-packing and cgroup-enforcement mechanics as any Pod (Resource Requests/Limits guides) — Jobs are not special here at all
- `activeDeadlineSeconds` + `backoffLimit` together — a batch task that's both retry-bounded AND wall-clock-bounded, whichever limit is hit first wins

---

# PART 7 — REAL-WORLD USE CASES

## Database migration
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 2
  template:
    spec:
      containers:
        - name: migrate
          image: myapp-migrations:1.0
          command: ["./migrate", "up"]
      restartPolicy: Never
```
Run once as part of a deployment pipeline (often as a Helm pre-install/pre-upgrade hook, or a CI/CD step that waits for `kubectl wait --for=condition=complete job/db-migration` before proceeding to roll out the application itself) — this is the canonical real-world reason Jobs exist at all: schema migrations must run to completion exactly once, in a controlled order, before dependent Pods start.

## Data processing
Fixed-completion or Indexed Jobs (Part 4) — splitting a large dataset into partitions, processing each independently and in parallel, with Kubernetes handling retry-on-failure for any partition that errors out without needing custom retry logic in the application itself.

## Backup
```yaml
# typically wrapped in a CronJob for recurring execution
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"              # 2 AM daily
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          containers:
            - name: backup
              image: my-backup-tool
              command: ["./backup.sh"]
          restartPolicy: OnFailure
```
A backup is a textbook Job/CronJob use case: it must run to completion, retry on transient failure (network blip to the backup destination), and leave a clean history of recent successes/failures for auditing — exactly the feature set Jobs + CronJob's history limits provide natively.

## Batch processing
The general case of "process N things, possibly in parallel, tolerate transient failures, then stop" — report generation, image/video transcoding queues, bulk email sends, ETL pipeline stages — all fit the same `completions`/`parallelism`/`backoffLimit` shape covered throughout this guide.

---

# PART 8 — TROUBLESHOOTING

**Job stuck, Pods keep getting recreated and failing repeatedly**
```bash
kubectl get job <name>
kubectl describe job <name>          # check backoffLimit progress in Events
kubectl logs job/<name>               # follows the most recent Pod's logs
```
Check the actual failure reason in the Pod logs first — a Job retrying a fundamentally broken command (bad command, missing file, wrong permissions) will retry it identically, and identically fail, `backoffLimit` times before giving up — retries never fix a deterministic bug.

**Job never completes, parallelism seems ignored**
Verify `completions` vs `parallelism` math — a common mistake is expecting `parallelism` alone to define "how many total runs," when it only bounds *concurrency*; without `completions` explicitly set (fixed-count mode) or a self-terminating queue-consumer pattern (queue mode), a Job may run far longer than expected.

**Job exceeded `activeDeadlineSeconds` and was killed mid-work**
```bash
kubectl get job <name> -o jsonpath='{.status.conditions}'
# reason: DeadlineExceeded
```
This is a hard, unconditional cutoff — even Pods that were about to succeed get terminated. Distinguish this from a `backoffLimit` failure by checking the condition reason specifically.

**CronJob isn't creating new Jobs on schedule**
```bash
kubectl get cronjob <name>
kubectl describe cronjob <name>       # check Last Schedule Time, and Events for skipped runs
```
Common causes: `concurrencyPolicy: Forbid` blocking a new run because the previous one is still active, or the CronJob controller itself being suspended (`spec.suspend: true`).

**Completed Jobs cluttering `kubectl get jobs` indefinitely**
Missing `ttlSecondsAfterFinished` — add it (Part 5) rather than manually cleaning up Job objects by hand.

---

# PART 9 — HANDS-ON MINIKUBE LABS

### Lab 1: Watch a basic Job run to completion
```bash
minikube start
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata: { name: lab1 }
spec:
  template:
    spec:
      containers: [{ name: task, image: busybox, command: ["sh","-c","sleep 5; echo done"] }]
      restartPolicy: Never
EOF
kubectl get job lab1 -w
kubectl logs job/lab1
```

### Lab 2: Watch retries happen with an intentionally-failing command
```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata: { name: lab2 }
spec:
  backoffLimit: 3
  template:
    spec:
      containers: [{ name: task, image: busybox, command: ["sh","-c","exit 1"] }]
      restartPolicy: Never
EOF
kubectl get pods -l job-name=lab2 -w
# watch 4 total Pods get created (1 initial + 3 retries) with increasing
# delays between them, then the Job settle into Failed
kubectl get job lab2 -o jsonpath='{.status.conditions}'
```

### Lab 3: Fixed completions + parallelism
```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata: { name: lab3 }
spec:
  completions: 6
  parallelism: 2
  template:
    spec:
      containers: [{ name: task, image: busybox, command: ["sh","-c","sleep 3; echo done"] }]
      restartPolicy: Never
EOF
kubectl get pods -l job-name=lab3 -w
# never more than 2 Running at once, 6 total successful completions before Complete
```

### Lab 4: Indexed Job
```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata: { name: lab4 }
spec:
  completions: 3
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      containers:
        - name: task
          image: busybox
          command: ["sh","-c","echo My index is $JOB_COMPLETION_INDEX"]
          env:
            - name: JOB_COMPLETION_INDEX
              valueFrom: { fieldRef: { fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index'] } }
      restartPolicy: Never
EOF
for p in $(kubectl get pods -l job-name=lab4 -o name); do kubectl logs $p; done
# each Pod printed a DIFFERENT index — 0, 1, 2
```

### Lab 5: TTL cleanup
```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata: { name: lab5 }
spec:
  ttlSecondsAfterFinished: 30
  template:
    spec:
      containers: [{ name: task, image: busybox, command: ["echo","done"] }]
      restartPolicy: Never
EOF
kubectl get job lab5 -w
# wait ~30s after completion, then it (and its Pod) vanish automatically
```

---

# PART 10 — CHEAT SHEET

```bash
kubectl create job my-job --image=busybox -- echo hello    # quick one-off Job
kubectl get jobs
kubectl describe job <name>
kubectl logs job/<name>
kubectl delete job <name>                    # cascades to delete its Pods too
kubectl wait --for=condition=complete job/<name> --timeout=60s   # useful in CI/CD pipelines
```

```
Field                     Default   Purpose
──────────────────────    ───────   ──────────────────────────────────
completions                 1        successful runs needed for Job completion
parallelism                 1        max concurrently running Pods
backoffLimit                 6        failures tolerated before marking Failed
activeDeadlineSeconds       none     hard wall-clock ceiling for the WHOLE job
restartPolicy               —        must be OnFailure or Never (never Always)
completionMode            NonIndexed  set to Indexed for per-Pod fixed indices
ttlSecondsAfterFinished    none     auto-delete this many seconds after finishing
```

---

# PART 11 — INTERVIEW QUESTIONS

**Fundamentals**
1. What's the fundamental difference in how a Deployment and a Job treat a Pod exiting successfully?
2. Why can't a Job's Pod template use `restartPolicy: Always`?
3. What does a bare Pod lack that a Job adds?

**Core fields**
4. What's the difference between `completions` and `parallelism`?
5. What happens when `backoffLimit` is exceeded?
6. What's the difference between `restartPolicy: OnFailure` and `restartPolicy: Never` in a Job's retry behavior?
7. What does `activeDeadlineSeconds` do, and how does it interact with `backoffLimit`?

**Parallel / Indexed**
8. Describe the two different ways a Job can run Pods in parallel.
9. What problem does `completionMode: Indexed` solve that plain `parallelism` doesn't?
10. In a work-queue-style Job with no `completions` set, how does Kubernetes know the Job is done?

**Cleanup**
11. What is `ttlSecondsAfterFinished`, and what component actually implements the cleanup?
12. What are `successfulJobsHistoryLimit`/`failedJobsHistoryLimit`, and which object type do they apply to?

**Scenario-based**
13. A Job keeps retrying and failing identically every time — is more retries the fix? Why or why not?
14. A CronJob isn't creating new runs on schedule — what are your first two hypotheses?
15. You need a database migration to run exactly once, successfully, before your app Pods start — how would you wire this into a deployment pipeline?
16. How would you process a 100-item dataset in exactly 10 fixed, evenly-sized parallel shards?

**Deeper internals**
17. Why does exponential backoff exist between Job retry attempts?
18. Is a Job's resource request/limit enforcement any different from a regular Pod's? Why or why not?
19. What Kubernetes object type owns the Pods a Job creates, and how would you verify that ownership on a live Pod?
20. Why is `DeadlineExceeded` a distinct failure reason from a `backoffLimit`-triggered failure, and how would you tell them apart from `kubectl` output alone?

---

*This guide covers Kubernetes Jobs from the Deployment-vs-Job distinction through parallel/Indexed execution patterns, TTL-based cleanup, and real-world batch-processing use cases — the complete arc from beginner to production batch-processing expertise.*

# Kubernetes CronJobs — Complete Mastery Guide
### From Absolute Beginner to Production Scheduling Expertise

---

# PART 1 — WHAT AND WHY

## What is a CronJob?

A CronJob is a controller that creates **Jobs** (from the Jobs guide) on a recurring schedule, expressed in standard cron syntax. Where a Job runs something to completion once, a CronJob's entire responsibility is deciding *when* to create the next Job — the actual work-running mechanics are 100% delegated to the Job it creates.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/5 * * * *"       # every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: busybox
              command: ["echo", "Hello from CronJob"]
          restartPolicy: OnFailure
```

## CronJob vs Job

| | Job | CronJob |
|---|---|---|
| Runs | Once (to `completions` count) | Repeatedly, on a schedule |
| You create it | Directly, when you need the work done | Once — it then creates Jobs for you, forever |
| Retry/parallelism logic | Lives here (`backoffLimit`, `parallelism`) | Delegated entirely to the Job it spawns |

**Architecturally, a CronJob adds exactly one thing on top of a Job: time.** It contributes nothing to how a single run behaves — that's still 100% governed by the Job semantics you already know from the Jobs guide, nested inside `.spec.jobTemplate`.

---

# PART 2 — THE FULL INTERNAL CHAIN: CronJob → Job → Pod → Container

```
┌────────────────────────────────────────────────────────────────┐
│                          CronJob controller                        │
│  (another ordinary controller in kube-controller-manager,           │
│   following the exact same watch/reconcile pattern as every         │
│   other one — Architecture masterclass, Part 2 §9)                   │
│                                                                      │
│  Every reconcile tick, for each CronJob:                              │
│    1. Compute: has the NEXT scheduled time (per .spec.schedule,        │
│       cron syntax) passed since the last recorded run?                  │
│    2. If yes: check concurrencyPolicy against any still-running          │
│       Job from a previous tick                                            │
│    3. If clear to proceed: CREATE a new Job object, populated              │
│       entirely from .spec.jobTemplate                                       │
└──────────────────────────────┬───────────────────────────────────┘
                                 │ creates (ownerReferences point back
                                 │  to the CronJob, exactly like the
                                 │  ReplicaSet→Pod ownership pattern)
                                 ▼
                   ┌───────────────────────────┐
                   │       Job controller         │   ← the EXACT SAME
                   │  (from the Jobs guide —       │     controller and
                   │   backoffLimit, completions,   │     logic as if you'd
                   │   parallelism — none of this    │     created this Job
                   │   is CronJob-specific at all)     │     by hand
                   └──────────────┬───────────────┘
                                   │ creates
                                   ▼
                        ┌───────────────────┐
                        │        Pod           │
                        └──────────┬─────────┘
                                   │ kubelet starts it via CRI
                                   ▼
                        ┌───────────────────┐
                        │      Container        │
                        └───────────────────┘
```

**The single most important architectural insight:** the CronJob controller has **zero knowledge of Pods or containers at all** — its entire job stops the instant it successfully creates a Job object. Everything below that line is indistinguishable from a hand-created Job. This layered-ignorance pattern (each controller knowing about exactly one level below it, per the Architecture masterclass Part 3 §15) is why CronJob's own feature set is so small — it doesn't need to reimplement retries, parallelism, or completion tracking, because the Job controller already does all of that.

---

# PART 3 — CRON SYNTAX

```
 ┌───────────── minute (0–59)
 │ ┌─────────── hour (0–23)
 │ │ ┌───────── day of month (1–31)
 │ │ │ ┌─────── month (1–12)
 │ │ │ │ ┌───── day of week (0–6, Sunday=0; also accepts names: SUN-SAT)
 │ │ │ │ │
 * * * * *
```

| Expression | Meaning |
|---|---|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour, on the hour |
| `0 2 * * *` | Every day at 2:00 AM |
| `0 2 * * 0` | Every Sunday at 2:00 AM |
| `0 0 1 * *` | First day of every month, midnight |
| `30 3 * * 1-5` | 3:30 AM, Monday through Friday |

## Common cron expression mistakes

1. **Confusing day-of-month and day-of-week fields** — `0 0 * * 1` (every Monday) is very different from `0 0 1 * *` (the 1st of every month) — a single-digit position swap with a completely different real-world meaning.
2. **Forgetting that BOTH day-of-month and day-of-week, if both restricted (not `*`), are OR'd together, not AND'd** — `0 0 1,15 * 1` means "midnight on the 1st, the 15th, OR any Monday" — not "the 1st or 15th, but only if it's also a Monday," which is the intuitive-but-wrong reading most people default to.
3. **Assuming `*/5` always aligns to :00** — `*/5 * * * *` does align to `:00, :05, :10...` because it's computed from the field's own start, but people sometimes mistakenly expect step values on other fields (like `*/7` on hours, which doesn't divide evenly into 24) to align sensibly — they don't; `*/7` on hours fires at `0,7,14,21`, not evenly spaced across a real day cycle in the way someone might assume.
4. **Off-by-one on day-of-week** — assuming `1` is Monday when authoring by hand; it's Sunday=`0`, so Monday is `1` — this one's actually correct in standard cron, but worth double-checking against your specific tool's convention since some non-Kubernetes cron dialects differ.

```bash
# always verify a new expression before trusting it in production:
kubectl create cronjob test-parse --image=busybox --schedule="30 3 * * 1-5" --dry-run=client -o yaml
```

## Timezone

By default, cron schedules run in the **kube-controller-manager's configured timezone** — historically this was a persistent source of confusion (UTC vs. local time mismatches between what someone typed and when it actually fired). Modern Kubernetes (1.27+) supports an explicit `timeZone` field:

```yaml
spec:
  schedule: "0 2 * * *"
  timeZone: "America/New_York"    # explicit IANA timezone — removes all ambiguity
```
**Always set this explicitly in production** — relying on the cluster's ambient timezone default is a classic source of "why did my backup run at 6 AM instead of 2 AM" incidents, especially in clusters that span regions or get migrated between cloud providers with different defaults.

---

# PART 4 — CONCURRENCY CONTROL

## concurrencyPolicy

Governs what happens if a scheduled time arrives while the **previous** Job is still running.

```yaml
spec:
  concurrencyPolicy: Forbid   # Allow | Forbid | Replace
```

| Value | Behavior |
|---|---|
| `Allow` (default) | Start the new Job anyway — multiple Jobs from this CronJob can run **simultaneously** |
| `Forbid` | Skip this scheduled run entirely if a previous Job is still active — no new Job is created |
| `Replace` | **Kill the currently-running Job** (and its Pods) and start the new one in its place |

```
Timeline, concurrencyPolicy: Forbid, schedule every 5 minutes, but each run takes 8 minutes:

t=0    Job A starts (will run until t=8)
t=5    scheduled trigger fires → Job A still running → SKIPPED, no Job B created
t=8    Job A completes
t=10   scheduled trigger fires → nothing running → Job C starts
```

**Choosing the right policy matters a lot in practice:** `Allow` for independent, safely-overlappable work (e.g., stateless report generation); `Forbid` for anything that would corrupt data or waste resources if run twice concurrently (most backups, most migrations); `Replace` for cases where only the *latest* attempt matters and a stale, still-running previous attempt should be abandoned (e.g., a monitoring snapshot job where an old, slow run is now pointless).

## startingDeadlineSeconds

If the CronJob controller itself was down, or the cluster was otherwise unable to create a Job at the scheduled time, this bounds **how late** a missed run is still allowed to start.

```yaml
spec:
  startingDeadlineSeconds: 200   # if more than 200s late, SKIP this run entirely rather than firing it late
```
Without this set, a CronJob that missed several scheduled times (e.g., the whole controller manager was down for an hour) will, by default, only attempt to catch up on a bounded number of the most recent missed runs (not run every single one that was missed, which could cause a thundering-herd catch-up storm) — but with no `startingDeadlineSeconds` at all, "late" runs can still fire well after their intended time. Setting this field gives you an explicit, deliberate cutoff instead of relying on the implicit default catch-up behavior.

## suspend

```yaml
spec:
  suspend: true    # pause scheduling entirely — no new Jobs created, existing running Jobs unaffected
```
Toggling this is the standard, non-destructive way to pause a CronJob (e.g., during a maintenance window) without deleting the object and losing its configuration/history — flip it back to `false` to resume normal scheduling.

---

# PART 5 — HISTORY, CLEANUP, AND FAILURE HANDLING

## successfulJobsHistoryLimit / failedJobsHistoryLimit (recap from the Jobs guide, CronJob-specific detail)

```yaml
spec:
  successfulJobsHistoryLimit: 3    # keep the 3 most recent successful Jobs (for inspection)
  failedJobsHistoryLimit: 1         # keep only the 1 most recent failed Job
```
Beyond these limits, the CronJob controller itself deletes the **oldest** excess Job objects (cascading to their Pods, via the same `ownerReferences` garbage-collection mechanism from the ReplicaSets guide) — this is CronJob-specific cleanup logic, distinct from the generic `ttlSecondsAfterFinished` field covered in the Jobs guide, though both can be used together.

## Missed schedules

If the CronJob controller is down (or the whole cluster is) across one or more scheduled times, upon recovery it looks at how many schedule times were missed:
- If the number of missed runs is small, it (subject to `startingDeadlineSeconds`) creates Jobs to catch up
- If **too many** schedules were missed (a hardcoded internal threshold, historically 100), the CronJob controller gives up catching up entirely and logs an error — it will **not** attempt to create 100+ backlogged Jobs at once, which would be both useless (stale work) and dangerous (a burst of simultaneous load)

```bash
kubectl describe cronjob <name>
# Events: "Cannot determine if job needs to be started: too many missed start times"
```

## Retries and failure handling

**A CronJob has no retry logic of its own** — every retry behavior (`backoffLimit`, `restartPolicy`) lives entirely inside `.spec.jobTemplate.spec`, governed by the ordinary Job mechanics from the Jobs guide. A failed scheduled run is handled exactly like any standalone failed Job — it does **not** automatically get "tried again sooner" by the CronJob controller; the next attempt happens at the next regularly scheduled time (or never, if `concurrencyPolicy: Forbid` and enough consecutive failures leave something in a stuck state — worth actively monitoring for).

---

# PART 6 — YAML EXAMPLES (LINE-BY-LINE)

## Every minute
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: every-minute
spec:
  schedule: "* * * * *"          # fires every single minute — testing/demo only in practice
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: task
              image: busybox
              command: ["date"]
          restartPolicy: OnFailure
```
- `jobTemplate.spec` — this entire block is a normal `Job.spec` (Jobs guide) — nothing here is CronJob-specific
- No `concurrencyPolicy` set → defaults to `Allow`; since this task is near-instant, overlap is essentially never observed in practice but is technically possible under load

## Daily backup
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"
  timeZone: "UTC"
  concurrencyPolicy: Forbid       # never run two backups at once
  startingDeadlineSeconds: 300     # if more than 5 min late, skip rather than run stale
  successfulJobsHistoryLimit: 7     # keep a week of successful run history
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600   # hard 1-hour ceiling on the backup itself
      template:
        spec:
          containers:
            - name: backup
              image: my-backup-tool:1.0
              command: ["./backup.sh"]
          restartPolicy: OnFailure
```
- `concurrencyPolicy: Forbid` — critical here: a backup job overlapping with itself could produce a corrupted or inconsistent snapshot
- `startingDeadlineSeconds: 300` — if the cluster had an outage and this backup would now start 6+ minutes late, skip it rather than run a backup at a confusing, unexpected time
- `activeDeadlineSeconds: 3600` (inside `jobTemplate`, a Job-level field) — protects against a hung backup process running forever and blocking every subsequent scheduled run under `Forbid`

## Weekly cleanup
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-cleanup
spec:
  schedule: "0 3 * * 0"          # 3 AM every Sunday
  concurrencyPolicy: Replace       # if last week's cleanup is somehow still running, abandon it
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cleanup
              image: my-cleanup-tool
              command: ["./cleanup-old-data.sh"]
          restartPolicy: OnFailure
```
- `concurrencyPolicy: Replace` — appropriate here because only the *latest* cleanup run matters; an old, still-running cleanup from a week ago being killed and superseded is the desired behavior, not a risk

## Database backup (fuller production example)
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 1 * * *"
  timeZone: "America/New_York"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      backoffLimit: 3
      template:
        spec:
          containers:
            - name: pg-dump
              image: postgres:16
              command:
                - sh
                - -c
                - "pg_dump -h $DB_HOST -U $DB_USER $DB_NAME > /backup/dump-$(date +%F).sql"
              envFrom:
                - secretRef:
                    name: db-credentials      # ConfigMaps/Secrets guide mechanics, unchanged
              volumeMounts:
                - name: backup-storage
                  mountPath: /backup
          restartPolicy: OnFailure
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: backup-pvc
```
- `envFrom.secretRef` — database credentials injected exactly as covered in the Secrets guide; no CronJob-specific handling needed
- `persistentVolumeClaim` — the dump file needs to survive past the Job/Pod's own lifetime (unlike an `emptyDir`, per the Pods masterclass §16), so it's written to real persistent storage instead

## Log cleanup
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-cleanup
spec:
  schedule: "0 4 * * *"
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cleanup
              image: busybox
              command: ["sh", "-c", "find /logs -mtime +7 -delete"]
              volumeMounts:
                - name: logs
                  mountPath: /logs
          restartPolicy: OnFailure
          volumes:
            - name: logs
              persistentVolumeClaim:
                claimName: log-storage-pvc
```
- `find /logs -mtime +7 -delete` — deletes anything older than 7 days; a classic, simple retention-policy pattern implemented with nothing more than a shell one-liner inside a scheduled container

---

# PART 7 — TROUBLESHOOTING

**CronJob never creates any Jobs at all**
```bash
kubectl get cronjob <name>
kubectl describe cronjob <name>
```
Check `suspend: true` first (easy to forget it was set), then verify the cron expression itself parses as intended (Part 3's common mistakes), then check `.status.lastScheduleTime` to confirm the controller is even attempting to evaluate this CronJob.

**Jobs are being skipped unexpectedly**
Check `concurrencyPolicy: Forbid` combined with a previous run that's hung/stuck (not actually making progress but not yet hitting `activeDeadlineSeconds` either) — this is the most common real-world cause of "why didn't my backup run last night," and pairing `Forbid` with a sensible `activeDeadlineSeconds` on the Job template prevents a single stuck run from silently blocking every future scheduled run indefinitely.

**"Too many missed start times" error**
Per Part 5 — the controller (or the whole cluster) was down long enough to miss the internal catch-up threshold; this requires manual intervention (verify the underlying work actually needs to happen for the missed window, then either accept the gap or manually trigger a one-off Job for it) rather than expecting automatic catch-up.

**A scheduled run fires at the wrong time**
Check `timeZone` first (Part 3) — if unset, verify the actual configured timezone of the cluster's control plane rather than assuming UTC.

**Manually triggering a CronJob's Job immediately, for testing**
```bash
kubectl create job manual-test-run --from=cronjob/daily-backup
```
This creates a one-off Job using the CronJob's current `jobTemplate`, without waiting for or affecting its actual schedule — the standard way to test a CronJob's underlying task logic on demand.

---

# PART 8 — MINIKUBE EXERCISES

### Exercise 1: Watch the full CronJob → Job → Pod chain
```bash
minikube start
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata: { name: lab1 }
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers: [{ name: task, image: busybox, command: ["date"] }]
          restartPolicy: OnFailure
EOF
kubectl get cronjob lab1 -w &
kubectl get jobs -w &
kubectl get pods -w
# watch a NEW Job appear every minute, each creating its own Pod
```

### Exercise 2: concurrencyPolicy: Forbid in action
```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata: { name: lab2 }
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers: [{ name: task, image: busybox, command: ["sleep", "90"] }]
          restartPolicy: OnFailure
EOF
kubectl get jobs -w
# task takes 90s but fires every 60s — watch every OTHER scheduled
# trigger get silently skipped, since the previous run is still active
```

### Exercise 3: Manual trigger
```bash
kubectl create job lab3-manual --from=cronjob/lab1
kubectl get jobs
kubectl logs job/lab3-manual
```

### Exercise 4: Suspend and resume
```bash
kubectl patch cronjob lab1 -p '{"spec":{"suspend":true}}'
kubectl get jobs -w    # confirm no new Jobs appear while suspended
kubectl patch cronjob lab1 -p '{"spec":{"suspend":false}}'
# scheduling resumes
```

### Exercise 5: History limits
```bash
kubectl patch cronjob lab1 -p '{"spec":{"successfulJobsHistoryLimit":2}}'
# wait several minutes
kubectl get jobs -l job-name!=""
# confirm only the 2 most recent successful Jobs are retained, older ones
# automatically garbage collected
```

---

# CHEAT SHEET

```bash
kubectl get cronjobs
kubectl describe cronjob <name>
kubectl create job <name> --from=cronjob/<cronjob-name>   # manual one-off trigger
kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'
kubectl get jobs --sort-by=.metadata.creationTimestamp    # see run history in order
```
```
Field                       Purpose
─────────────────────────  ───────────────────────────────────────
schedule                     standard 5-field cron expression
timeZone                     explicit IANA timezone (set this always)
concurrencyPolicy            Allow | Forbid | Replace
startingDeadlineSeconds      how late a missed run may still fire
suspend                      pause/resume scheduling non-destructively
successfulJobsHistoryLimit    retained successful Job objects
failedJobsHistoryLimit        retained failed Job objects
jobTemplate                   an entire ordinary Job spec, nested
```

---

# INTERVIEW QUESTIONS

**Fundamentals**
1. What does a CronJob add on top of a Job?
2. Trace the full ownership chain from CronJob down to a running container.
3. Does the CronJob controller know anything about Pods or containers directly?

**Cron syntax**
4. What are the five fields in a cron expression, in order?
5. What's a common mistake when combining restricted day-of-month and day-of-week fields?
6. Why should `timeZone` always be set explicitly in production?

**Concurrency**
7. Explain the difference between `Allow`, `Forbid`, and `Replace`.
8. Why might `Forbid` silently prevent a CronJob from ever running again? What field mitigates this?
9. When would `Replace` be the correct choice over `Forbid`?

**Failure handling**
10. What happens if the CronJob controller is down when a scheduled time passes?
11. What is the "too many missed start times" condition, and why does Kubernetes refuse to catch up in that case?
12. Where does retry logic for a failed scheduled run actually live?

**Cleanup**
13. What's the difference between `successfulJobsHistoryLimit`/`failedJobsHistoryLimit` and a Job's own `ttlSecondsAfterFinished`?
14. What happens to Jobs beyond the configured history limits?

**Scenario-based**
15. A nightly backup CronJob hasn't run in three days — walk through your diagnostic steps.
16. How would you test a CronJob's actual task logic without waiting for its schedule?
17. A CronJob fires at what looks like the wrong time of day — what's the first thing you check?
18. Why would pairing `concurrencyPolicy: Forbid` with `activeDeadlineSeconds` on the Job template be good practice?
19. You need a job that must never run twice concurrently but also must never be silently skipped for more than a day — how would you combine the available fields to enforce this?
20. Explain, architecturally, why CronJob's feature set is so much smaller than Job's.

---

*This guide covers Kubernetes CronJobs from cron syntax through concurrency control, missed-schedule handling, and the full CronJob → Job → Pod → Container ownership chain — the complete arc from beginner to production scheduling expertise.*


# Kubernetes Services — Complete Mastery Guide
### From Absolute Beginner to Production-Grade Networking Expert

---

# PART 1 — WHY SERVICES EXIST

## 1. The Problem Services Solve

Kubernetes constantly creates and destroys Pods. Every deployment rollout, crash, scale-up, scale-down, or node failure kills old Pods and creates new ones. This is normal and expected — Pods are *cattle, not pets*.

But this creates a networking nightmare: **how does anything reliably talk to a Pod that might not exist five seconds from now?**

## 2. The Pod IP Problem

Every Pod gets its own IP address when it's scheduled. This IP is:

- **Ephemeral** — destroyed when the Pod dies, even if a replacement Pod immediately takes its place
- **Unpredictable** — you cannot know it in advance
- **Not load-balanced** — if you have 5 replicas, you have 5 different IPs, and nothing distributes traffic across them for you

```
Deployment "backend" (3 replicas)

  Pod A  10.244.1.5   ─┐
  Pod B  10.244.2.9   ─┼─ How does a client reach "the backend"
  Pod C  10.244.3.2   ─┘   without hardcoding 3 changing IPs?

Pod C crashes → replaced by Pod D at 10.244.3.7
  Anyone who hardcoded 10.244.3.2 is now broken.
```

If your frontend hardcodes Pod IPs, it breaks constantly. You need a **stable, permanent address** that automatically tracks whichever Pods are currently healthy.

## 3. The Service Abstraction

A **Service** is a stable virtual IP (VIP) and DNS name that sits in front of a dynamic set of Pods. It:

1. Never changes its IP/DNS name for its entire lifetime
2. Continuously tracks which Pods are currently healthy and ready
3. Load-balances traffic across those Pods
4. Decouples "who is calling" from "which exact Pod answers"

```
                 ┌─────────────────────┐
   Client ─────▶ │  Service: backend   │
                 │  ClusterIP: 10.96.5.5│
                 └──────────┬──────────┘
                            │ load-balanced
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
           Pod A         Pod B         Pod C
        10.244.1.5    10.244.2.9    10.244.3.2
```

This is the single most important mental model: **a Service is not a process, not a Pod, not a listening socket in the traditional sense — it's a set of routing rules maintained cluster-wide, implemented by kube-proxy (or an eBPF equivalent) on every node.**

---

# PART 2 — HOW SERVICES FIND PODS

## 4. Service Selectors

A Service finds its Pods using **label selectors** — plain key-value matches, nothing more.

```yaml
selector:
  app: backend
  tier: api
```

Any Pod carrying **both** labels `app=backend` and `tier=api` is a match. Selectors are:

- Simple equality-based matching (no wildcards)
- Re-evaluated continuously, not once at creation time
- The *only* mechanism connecting a Service to its Pods — there's no direct "pointer"

If you have a Service with no matching Pods, it silently exists with zero endpoints — no error is thrown. This is the #1 cause of "Service isn't working" bugs (see Troubleshooting section).

## 5. Endpoints

Every time you create a Service with a selector, Kubernetes automatically creates a companion **Endpoints** object (legacy API, `v1`) listing the IP:port pairs of every currently matching, *ready* Pod.

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: backend        # same name as the Service
subsets:
  - addresses:
      - ip: 10.244.1.5
      - ip: 10.244.2.9
    ports:
      - port: 9376
```

Key facts:
- Only **Ready** Pods appear here (failed readiness probes = removed from Endpoints, traffic stops flowing, Pod is NOT killed)
- One Endpoints object per Service, named identically
- This object is *continuously reconciled* by the Endpoints controller watching Pod changes

## 6. EndpointSlices (the modern replacement)

The legacy `Endpoints` object has a hard scaling problem: it stores **all** addresses in a single object, and every single Pod IP change rewrites and re-transmits the *entire* object to every watcher. With 5,000 Pods behind a Service, that's brutal.

**EndpointSlices** (default since Kubernetes 1.17+, GA 1.21+) fix this by:

- Sharding addresses into slices of **max 100 endpoints each**
- Adding topology info (zone, node) for topology-aware routing
- Supporting multiple address types (IPv4, IPv6, FQDN)
- Reducing update fan-out — only the changed slice is rewritten

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: backend-abc12
  labels:
    kubernetes.io/service-name: backend   # links back to the Service
addressType: IPv4
ports:
  - name: http
    port: 9376
    protocol: TCP
endpoints:
  - addresses: ["10.244.1.5"]
    conditions:
      ready: true
      serving: true
      terminating: false
    nodeName: node-1
    zone: us-east-1a
  - addresses: ["10.244.2.9"]
    conditions:
      ready: true
```

Note the richer `conditions` block: `ready`, `serving`, and `terminating` are tracked *separately*, which enables graceful handling of Pods that are shutting down (still serving in-flight connections but not accepting new ones).

**You never write EndpointSlices by hand for a normal Service** — Kubernetes generates them automatically from the selector. You only hand-author them for Services *without* selectors (see ExternalName-adjacent patterns).

---

# PART 3 — SERVICE TYPES

## 7. ClusterIP (default)

Exposes the Service on a virtual IP reachable **only from inside the cluster**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP          # default; can be omitted
  selector:
    app: backend            # find Pods labeled app=backend
  ports:
    - port: 80               # Service's own port (what clients call)
      targetPort: 8080       # container port to actually forward to
      protocol: TCP
```

**Line-by-line:**
- `type: ClusterIP` — internal-only virtual IP, allocated from the cluster's service CIDR (e.g. `10.96.0.0/12`)
- `selector` — the label match rule that populates EndpointSlices
- `port: 80` — the port *other Pods* use when calling `backend:80`
- `targetPort: 8080` — the port the container inside the Pod is actually listening on; these can differ freely
- No `protocol` given for UDP-based apps? Add `protocol: UDP` explicitly.

Use case: internal microservice-to-microservice traffic (databases, internal APIs).

## 8. NodePort

Everything ClusterIP does, **plus** it opens a static port (30000–32767 by default) on **every node's** IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - port: 80              # ClusterIP port (still exists!)
      targetPort: 8080      # container port
      nodePort: 30080       # exposed on EVERY node at this port
```

- `nodePort: 30080` — reachable at `<any-node-ip>:30080`, even nodes running zero matching Pods (traffic hops over the cluster network to a node that has one)
- If you omit `nodePort`, Kubernetes auto-assigns one from the range
- A NodePort Service **is** a ClusterIP Service with an extra door — the ClusterIP still works too

Use case: quick external access without a cloud load balancer (bare-metal, testing, on-prem).

## 9. LoadBalancer

Everything NodePort does, **plus** it asks the cloud provider (AWS, GCP, Azure, etc.) to provision an external L4 load balancer that forwards to the NodePort.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb   # provider-specific tuning
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - port: 443
      targetPort: 8443
  externalTrafficPolicy: Local     # covered in section 22
```

- Requires a **cloud-controller-manager** integration; on bare Minikube it stays `<pending>` unless you run `minikube tunnel`
- Cloud provider allocates a real public IP/DNS and programs its LB to forward to the NodePort on cluster nodes
- Annotations are the *only* way to control provider-specific LB behavior (internal vs internet-facing, TLS termination, health check paths, etc.) — this is not standardized across clouds

Use case: production internet-facing entry points (though often you'd front it with an Ingress Controller's own single LoadBalancer Service instead of one per app).

## 10. ExternalName

No selector, no proxying, no Endpoints at all. It's a **pure DNS CNAME alias** living inside the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: prod-db.us-east-1.rds.amazonaws.com
```

- Any Pod resolving `external-db.default.svc.cluster.local` gets a **CNAME** to `prod-db.us-east-1.rds.amazonaws.com`
- No kube-proxy rules are created — this is resolved entirely at the DNS layer by CoreDNS
- No load balancing, no health checking — it's literally just an alias

Use case: giving external resources (managed databases, SaaS APIs, legacy on-prem services) an in-cluster name so app config never needs to change between environments.

## 11. Headless Services

Set `clusterIP: None` to disable the virtual IP entirely. Instead of load-balancing, DNS returns **all Pod IPs directly**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cassandra
spec:
  clusterIP: None            # <- this is what makes it "headless"
  selector:
    app: cassandra
  ports:
    - port: 9042
```

- A normal DNS `A` record lookup on `cassandra.default.svc.cluster.local` returns **every individual Pod IP**, not one VIP
- Used heavily with **StatefulSets**, where each Pod also gets a stable per-Pod DNS name: `cassandra-0.cassandra.default.svc.cluster.local`
- No load balancing happens at the Service level — the client (or client library, e.g. a Cassandra/MongoDB driver) is expected to do its own peer discovery and connection management

Use case: stateful, peer-aware systems (databases, Kafka, Elasticsearch) where clients need to know about *every* member, not just "one of them."

---

# PART 4 — DNS AND SERVICE DISCOVERY

## 12. DNS

CoreDNS (the default cluster DNS addon) watches the API server and auto-generates records for every Service:

| Service type | DNS behavior |
|---|---|
| ClusterIP | `A`/`AAAA` record → the VIP |
| Headless | `A`/`AAAA` records → all Pod IPs |
| ExternalName | `CNAME` → the external name |

Full DNS name format:
```
<service-name>.<namespace>.svc.<cluster-domain>
backend.default.svc.cluster.local
```

Within the **same namespace**, just `backend` resolves (via search-domain suffixing in `/etc/resolv.conf`). Cross-namespace, use `backend.other-namespace` at minimum.

Every Pod's `/etc/resolv.conf` looks like:
```
nameserver 10.96.0.10          # CoreDNS's own ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

`ndots:5` means any name with fewer than 5 dots gets the search domains appended first — this is a classic source of DNS latency bugs for external lookups (see Troubleshooting).

## 13. Service Discovery (the full picture)

Kubernetes gives you **two** parallel discovery mechanisms:

1. **DNS** (preferred) — `curl http://backend/api`
2. **Environment variables** — injected into every Pod at *creation time only*, for Services that existed *before* the Pod started:
   ```
   BACKEND_SERVICE_HOST=10.96.5.5
   BACKEND_SERVICE_PORT=80
   ```
   This is legacy, brittle (ordering-dependent), and effectively obsolete — always prefer DNS.

---

# PART 5 — THE DATA PLANE: HOW TRAFFIC ACTUALLY MOVES

## 14. kube-proxy

kube-proxy is a **per-node agent** (DaemonSet) that watches Services and EndpointSlices via the API server and programs the node's packet-forwarding rules accordingly. It does **not** proxy traffic through userspace anymore (that mode is deprecated) — it just writes rules that the kernel enforces.

It supports three backend modes:

### 15. iptables mode (long-time default)

kube-proxy writes chains of `iptables` NAT rules. For every Service, a chain does **random probabilistic DNAT** across the matching Pod IPs.

```
PREROUTING → KUBE-SERVICES → KUBE-SVC-BACKEND → (33% chance) KUBE-SEP-PODA → DNAT to 10.244.1.5:8080
                                               → (33% chance) KUBE-SEP-PODB → DNAT to 10.244.2.9:8080
                                               → (33% chance) KUBE-SEP-PODC → DNAT to 10.244.3.2:8080
```

- Rule evaluation is **O(n)** in the number of Services/endpoints — a linear chain walk. At thousands of Services this gets measurably slow.
- Uses the kernel's `conntrack` table so a connection's *return* traffic is automatically un-DNAT'd
- No real load-balancing algorithm — just weighted random chance via sequential probability rules

### 16. IPVS mode

Uses the kernel's IP Virtual Server (a real, in-kernel L4 load balancer originally built for LVS) instead of iptables chains.

- Uses **hash tables**, so lookups are O(1) regardless of Service count — scales far better at high Service counts
- Supports real load-balancing algorithms: round-robin, least-connection, destination hashing, source hashing, etc. (`--ipvs-scheduler`)
- Requires the `ip_vs` kernel modules loaded on every node
- Still relies on iptables for some edge cases (NAT for non-cluster-IP traffic)

### 17. eBPF-based networking (Cilium, and others)

Modern CNIs (most notably **Cilium**) can replace kube-proxy entirely with eBPF programs attached directly to network hooks in the kernel (XDP, tc, socket layer).

- Skips iptables/IPVS altogether — packet processing happens in-kernel, in eBPF bytecode, before it even reaches the normal networking stack
- Can do **socket-level load balancing**: for Pod-to-Service traffic, the destination is rewritten at `connect()` time in the *originating Pod's socket*, so the packet is born already addressed to the real Pod — no DNAT hop, no conntrack overhead
- Enables advanced features cheaply: L7-aware policies, per-request observability (Hubble), better multi-cluster routing
- This is where the ecosystem is heading — "kube-proxy replacement mode" is now a standard Cilium/Calico feature

```
                    iptables/IPVS                          eBPF (Cilium)
Client packet ──▶ kernel netfilter hooks ──▶ Pod    Client packet ──▶ eBPF program (already
                  (DNAT happens per-packet)                          rewritten at socket layer) ──▶ Pod
```

---

# PART 6 — PORTS: THE THREE NUMBERS THAT CONFUSE EVERYONE

## 18–20. `port`, `targetPort`, `nodePort`

```yaml
ports:
  - port: 80          # (18) What OTHER PODS dial: curl http://backend:80
    targetPort: 8080  # (19) What the CONTAINER is actually listening on
    nodePort: 30080   # (20) What EXTERNAL clients dial: <node-ip>:30080  (NodePort/LB only)
```

| Field | Who uses it | Example |
|---|---|---|
| `port` | Anything calling the Service by its ClusterIP/DNS name | `backend:80` |
| `targetPort` | Internal — maps to the container's actual listening port | container listens on `8080` |
| `nodePort` | External clients hitting a node directly | `node-ip:30080` |

Full flow for a NodePort Service:
```
External client → node-ip:30080 (nodePort)
                → Service VIP:80 (port)
                → Pod IP:8080 (targetPort)
                → container process actually listening on 8080
```

`targetPort` can also be a **named port** referencing the container spec (`targetPort: http`), which is more resilient — you can change the container's actual port number without touching the Service.

---

# PART 7 — TRAFFIC BEHAVIOR CONTROLS

## 21. Session Affinity

By default, every request is load-balanced independently — back-to-back requests from the same client may land on different Pods. To pin a client to one Pod:

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800     # default 3 hours
```

- Based on **client source IP**, not cookies (Services operate at L4, they have no concept of HTTP cookies)
- Implemented as sticky rules in iptables/IPVS keyed on source IP
- Breaks down behind NAT (many users appear as one IP) or with `externalTrafficPolicy: Cluster` (source IP gets masqueraded away before it reaches the node) — pair with `Local` policy for real client-IP affinity from outside

## 22. externalTrafficPolicy

Controls what happens to traffic entering via **NodePort or LoadBalancer** when it lands on a node with no local matching Pod.

```yaml
spec:
  externalTrafficPolicy: Local    # or: Cluster (default)
```

| Value | Behavior | Trade-off |
|---|---|---|
| `Cluster` (default) | Node forwards to *any* Pod cluster-wide, even on another node | Even spread, but adds an extra network hop and **masks the real client source IP** (SNAT to node IP) |
| `Local` | Node **only** forwards to Pods running on itself; drops the packet if none exist locally | Preserves real client source IP, avoids the extra hop, **but** traffic can be unevenly distributed and fails entirely on nodes with zero matching Pods (health checks against those nodes will fail, which cloud LBs use to route around them) |

This single setting is the source of a huge number of "why do I see the load balancer's IP instead of the real client IP in my logs" tickets.

## 23. internalTrafficPolicy

The same idea, but for traffic arriving via the **ClusterIP** from inside the cluster.

```yaml
spec:
  internalTrafficPolicy: Local    # or: Cluster (default)
```

- `Cluster` (default): any Pod cluster-wide can answer
- `Local`: a Pod on Node X calling the Service will only be routed to another Pod **on Node X**; if none exists there, the connection is dropped

Used for topology-aware, latency-sensitive setups (e.g., a node-local caching sidecar pattern) where you want traffic to never leave the node.

---

# PART 8 — THE COMPLETE INTERNAL TRAFFIC FLOW

## Client → Service → EndpointSlice → Pod (full internal walkthrough)

```
1. DNS RESOLUTION
   Pod's app calls http://backend
   → glibc/musl resolver reads /etc/resolv.conf
   → queries CoreDNS at 10.96.0.10
   → CoreDNS returns backend's ClusterIP, e.g. 10.96.5.5

2. PACKET SENT
   App opens a TCP connection to 10.96.5.5:80
   → packet leaves the Pod's network namespace via its veth pair
   → arrives at the node's root network namespace

3. KERNEL INTERCEPTION (iptables/IPVS mode)
   → netfilter PREROUTING hook catches packets destined for 10.96.5.5
     (this IP isn't real — it never appears on any interface; it exists
      only as a rule target)
   → kube-proxy's programmed rules perform DNAT:
        10.96.5.5:80  →  10.244.2.9:8080   (one of the ready endpoints,
                                             chosen via weighted random /
                                             IPVS scheduler)
   → conntrack records this so return traffic is automatically un-DNAT'd

   [eBPF mode: this rewrite can happen even earlier — at the socket's
    connect() syscall inside the ORIGINATING Pod, before a packet is
    even constructed, skipping the netfilter hop entirely]

4. ROUTING TO THE TARGET POD
   → if target Pod is on the same node: direct veth hop
   → if on a different node: encapsulated (VXLAN/Geneve) or routed
     natively depending on CNI, then delivered to the Pod's veth

5. POD RECEIVES
   → packet arrives at 10.244.2.9:8080, which the container is
     actually listening on
   → response follows the reverse path; conntrack un-DNATs it back
     to look like it came from 10.96.5.5:80, so the client never
     sees the real Pod IP

6. ENDPOINTSLICE ROLE (control plane, not data plane)
   → EndpointSlices don't touch any packet directly — they are the
     SOURCE OF TRUTH that kube-proxy/eBPF watches to know which
     Pod IPs are currently valid targets. A Pod failing readiness
     is removed from the EndpointSlice within milliseconds, and
     kube-proxy reprograms rules to stop sending it new traffic.
```

---

# PART 9 — TROUBLESHOOTING PLAYBOOK

## A. Service has no endpoints

```bash
kubectl get endpointslices -l kubernetes.io/service-name=backend
kubectl describe svc backend
```
If `Endpoints: <none>`:
- **Selector/label mismatch** (by far the most common cause — see below)
- Pods exist but are **not Ready** (failing readiness probe) — check `kubectl get pods -o wide` and `kubectl describe pod`
- Pods simply don't exist yet, or were scaled to 0

## B. Wrong selector

```bash
kubectl get svc backend -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels
```
Compare the two outputs by eye. A single typo (`app: backend` on the Service vs `app: Backend` or `app: backend-api` on the Pods) results in zero endpoints with **no error message anywhere** — this is the single most common Kubernetes networking bug for beginners.

## C. Wrong targetPort

Symptom: Endpoints show up fine, but every connection is refused or times out.
```bash
kubectl exec -it <pod> -- netstat -tlnp     # or: ss -tlnp
```
Confirm the container is *actually* listening on the port you set as `targetPort`. A very common mistake: app listens on `3000`, Service sets `targetPort: 8080` (copy-pasted from another manifest).

## D. Connection refused

- Usually means the packet *reached* the Pod but nothing was listening on that exact port → recheck `targetPort` (above)
- Could also mean a **NetworkPolicy** is blocking ingress to the Pod — check `kubectl get networkpolicy -A`
- Could mean the container crashed right after passing its readiness probe (race condition) — check `kubectl logs --previous`

## E. DNS failure

```bash
kubectl run tmp --rm -it --image=busybox:1.36 -- sh
# inside:
nslookup backend
nslookup backend.default.svc.cluster.local
cat /etc/resolv.conf
```
Common causes:
- CoreDNS Pods themselves are down/crashlooping: `kubectl -n kube-system get pods -l k8s-app=kube-dns`
- Wrong namespace assumed — short name `backend` only works within the same namespace
- `ndots:5` + external domain lookups causing 5x redundant queries and timeouts — mitigate with `dnsConfig` overrides or a trailing dot (`backend.com.`)
- NetworkPolicy blocking egress to CoreDNS's ClusterIP on port 53

## F. NodePort unavailable

- Port outside the allowed range (default `30000–32767`) — either pick a valid one or expand the range via API server flag `--service-node-port-range`
- Port already in use on the host by another process/Service
- Cloud security groups / firewall rules not allowing inbound traffic to that port range
- `externalTrafficPolicy: Local` and you're hitting a node with **zero** local matching Pods — connection is deliberately dropped by design, try a different node or switch policy

---

# PART 10 — YAML COOKBOOK (ALL TYPES, RUNNABLE)

```yaml
# --- ClusterIP ---
apiVersion: v1
kind: Service
metadata:
  name: svc-clusterip
spec:
  type: ClusterIP
  selector: { app: demo }
  ports:
    - port: 80
      targetPort: 8080
---
# --- NodePort ---
apiVersion: v1
kind: Service
metadata:
  name: svc-nodeport
spec:
  type: NodePort
  selector: { app: demo }
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
---
# --- LoadBalancer ---
apiVersion: v1
kind: Service
metadata:
  name: svc-loadbalancer
spec:
  type: LoadBalancer
  selector: { app: demo }
  ports:
    - port: 80
      targetPort: 8080
  externalTrafficPolicy: Local
---
# --- ExternalName ---
apiVersion: v1
kind: Service
metadata:
  name: svc-externalname
spec:
  type: ExternalName
  externalName: api.stripe.com
---
# --- Headless ---
apiVersion: v1
kind: Service
metadata:
  name: svc-headless
spec:
  clusterIP: None
  selector: { app: demo-stateful }
  ports:
    - port: 5432
---
# --- Session affinity + internalTrafficPolicy ---
apiVersion: v1
kind: Service
metadata:
  name: svc-sticky
spec:
  selector: { app: demo }
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP: { timeoutSeconds: 3600 }
  internalTrafficPolicy: Local
  ports:
    - port: 80
      targetPort: 8080
---
# supporting Deployment used by all Services above
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  replicas: 3
  selector:
    matchLabels: { app: demo }
  template:
    metadata:
      labels: { app: demo }
    spec:
      containers:
        - name: web
          image: hashicorp/http-echo
          args: ["-text=hello", "-listen=:8080"]
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet: { path: /, port: 8080 }
            initialDelaySeconds: 2
```

---

# PART 11 — MINIKUBE HANDS-ON LABS

### Lab 1: ClusterIP + Endpoints/EndpointSlice observation
```bash
minikube start
kubectl apply -f cookbook.yaml
kubectl get endpointslices -l kubernetes.io/service-name=svc-clusterip -o yaml
kubectl run test --rm -it --image=busybox:1.36 -- wget -qO- svc-clusterip
```

### Lab 2: Break the selector on purpose
```bash
kubectl patch svc svc-clusterip -p '{"spec":{"selector":{"app":"wrong-label"}}}'
kubectl get endpointslices -l kubernetes.io/service-name=svc-clusterip
# → empty. Revert:
kubectl patch svc svc-clusterip -p '{"spec":{"selector":{"app":"demo"}}}'
```

### Lab 3: NodePort access
```bash
minikube service svc-nodeport --url
curl $(minikube service svc-nodeport --url)
```

### Lab 4: LoadBalancer via tunnel
```bash
minikube tunnel &      # run in a separate terminal, needs sudo
kubectl get svc svc-loadbalancer -w
# watch EXTERNAL-IP move from <pending> to an address
```

### Lab 5: Headless Service DNS behavior
```bash
kubectl scale deployment demo --replicas=3
kubectl run test --rm -it --image=busybox:1.36 -- nslookup svc-headless
# → returns THREE A records instead of one VIP
```

### Lab 6: Watch kube-proxy's actual iptables rules
```bash
minikube ssh
sudo iptables -t nat -L KUBE-SERVICES -n | grep svc-clusterip
sudo iptables -t nat -L -n | grep KUBE-SVC
```

### Lab 7: Simulate a readiness failure and watch traffic stop
```bash
kubectl exec -it <one-demo-pod> -- sh -c "kill 1"   # crash the container
kubectl get endpointslices -w   # watch the Pod disappear from endpoints within seconds
```

---

# PART 12 — COMPARISON TABLES

### Service Types
| Type | Internal access | External access | Load balancing | Cloud dependency |
|---|---|---|---|---|
| ClusterIP | Yes (VIP) | No | Yes | No |
| NodePort | Yes | Yes (`node-ip:port`) | Yes | No |
| LoadBalancer | Yes | Yes (real public IP) | Yes | Yes |
| ExternalName | DNS alias only | N/A | No | No |
| Headless | DNS → all Pod IPs | No | No (client-side) | No |

### kube-proxy Modes
| Mode | Data structure | Lookup complexity | LB algorithms | Notes |
|---|---|---|---|---|
| iptables | Linear chains | O(n) | Weighted random only | Legacy default |
| IPVS | Hash tables | O(1) | RR, least-conn, hashing, etc. | Needs `ip_vs` modules |
| eBPF (Cilium etc.) | eBPF maps | O(1), socket-level | Configurable | Bypasses netfilter, lowest overhead |

### Traffic Policy Trade-offs
| Setting | Preserves client IP | Even distribution | Fails on zero-local-Pod nodes |
|---|---|---|---|
| `externalTrafficPolicy: Cluster` | No (SNAT) | Yes | No |
| `externalTrafficPolicy: Local` | Yes | No (uneven) | Yes |
| `internalTrafficPolicy: Cluster` | N/A | Yes | No |
| `internalTrafficPolicy: Local` | N/A | No | Yes |

---

# PART 13 — CHEAT SHEET

```bash
# Inspect
kubectl get svc -A
kubectl describe svc <name>
kubectl get endpointslices -l kubernetes.io/service-name=<name>

# Debug DNS
kubectl run dnsutils --rm -it --image=infoblox/dnstools -- bash
nslookup <svc>.<namespace>.svc.cluster.local

# Debug connectivity
kubectl exec -it <pod> -- curl -v http://<svc>:<port>
kubectl exec -it <pod> -- nc -zv <svc> <port>

# Watch live rule changes
minikube ssh -- sudo iptables -t nat -L -n
minikube ssh -- sudo ipvsadm -Ln     # IPVS mode only

# Port-forward for direct Pod debug (bypasses Service entirely)
kubectl port-forward pod/<name> 8080:8080
```

**Key defaults to memorize:**
- NodePort range: `30000–32767`
- Service CIDR: cluster-specific, commonly `10.96.0.0/12`
- Session affinity timeout default: `10800s` (3 hours)
- DNS `ndots`: `5`

---

# PART 14 — PRODUCTION BEST PRACTICES

1. **Always set readiness probes.** Without one, a Pod is "Ready" the instant it starts, even before it can actually serve traffic — you'll get real 502s during every rollout.
2. **Use named `targetPort`s** (`targetPort: http`) instead of numbers so container port changes don't require touching the Service.
3. **Default to `Cluster` traffic policy unless you specifically need client-IP preservation** — `Local` is a common source of uneven load and mysterious node-specific failures.
4. **Prefer IPVS or an eBPF-based CNI at scale.** iptables mode visibly degrades control-plane reconciliation time past a few thousand Services.
5. **Never rely on Service env vars** — DNS only. Env vars are a legacy footgun tied to Pod creation order.
6. **Use Headless Services + StatefulSets for anything peer-aware** (databases, message queues) — don't force a normal VIP onto a system that needs to see every peer.
7. **Put a single Ingress Controller (with one LoadBalancer Service) in front of many apps**, rather than provisioning a `LoadBalancer` type per microservice — this saves real money on cloud LB costs and centralizes TLS.
8. **Pin session affinity carefully.** `ClientIP` affinity is L4-only and breaks down behind corporate NATs/proxies; for HTTP session stickiness, do it at the Ingress/L7 layer instead.
9. **Monitor EndpointSlice churn.** Frequent Ready/NotReady flapping on a Service is an early warning sign of probe misconfiguration or resource starvation.
10. **Restrict Service traffic with NetworkPolicies** — Services alone provide zero security boundary; anything on the pod network can reach a ClusterIP by default.

---

# PART 15 — INTERVIEW QUESTIONS

**Conceptual**
1. Why can't clients just talk to Pod IPs directly?
2. What's the difference between a Service and an Endpoints/EndpointSlice object?
3. Why were EndpointSlices introduced to replace Endpoints?
4. Explain the difference between `port`, `targetPort`, and `nodePort`.
5. What happens to traffic when a Pod fails its readiness probe but is still running?

**Types**
6. When would you choose a Headless Service over a normal ClusterIP?
7. What's the actual difference between NodePort and LoadBalancer under the hood?
8. How does ExternalName differ from every other Service type architecturally?

**Data plane**
9. Compare iptables and IPVS kube-proxy modes in terms of performance at scale.
10. How does eBPF-based service routing avoid the DNAT/conntrack overhead of iptables?
11. Walk through, packet by packet, what happens when a Pod calls `http://my-svc`.

**Traffic policy**
12. What problem does `externalTrafficPolicy: Local` solve, and what does it cost you?
13. Why might session affinity silently fail to work as expected in production?

**Troubleshooting (scenario-based)**
14. A Service shows zero endpoints — walk through your full debugging process.
15. Users report "connection refused" errors — what are the first three things you check?
16. DNS resolution is slow for external domains from inside Pods — what's likely happening, and how do you fix it?

---

*This guide covers Kubernetes Services from the Pod IP problem through eBPF-based data planes, production traffic policies, and full troubleshooting — the complete arc from beginner to production-grade networking expert.*


# Kubernetes Ingress — Complete Mastery Guide
### From Absolute Beginner to Production-Grade HTTP Routing

---

# PART 1 — WHY INGRESS EXISTS

## 1. The Problem After Services

A `LoadBalancer` Service gets you one external IP per Service. That's fine for one app. It falls apart the moment you have 20 microservices, each needing its own public HTTP(S) endpoint — you'd be paying for and managing 20 cloud load balancers, none of which understand HTTP concepts like hostnames, URL paths, or TLS termination. They just forward raw TCP/UDP.

**Ingress** is Kubernetes' answer: a single object that describes **HTTP/HTTPS routing rules** — "requests for `api.example.com` go to the `api` Service, requests for `example.com/blog` go to the `blog` Service" — all fronted by **one** entry point.

## 2. What Ingress Actually Is

Ingress is **not** a running process. It is a Kubernetes API object — a declarative *set of routing rules*. By itself it does nothing. It requires a separate piece of software, the **Ingress Controller**, to read those rules and actually implement them.

This is the single most important distinction to internalize: **Ingress = the rulebook. Ingress Controller = the engine that enforces the rulebook.** Creating an Ingress object with no controller running is a no-op — nothing listens, nothing routes.

## 3. Ingress vs Service — the core distinction

| | Service | Ingress |
|---|---|---|
| OSI Layer | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| Understands hostnames/paths? | No | Yes |
| Understands TLS/SNI? | No (LoadBalancer can pass-through raw TLS, nothing more) | Yes — can terminate TLS |
| External IPs needed | One per `LoadBalancer` Service | One total, shared by all apps |
| Routing granularity | Whole Service, whole port | Host + path combinations |

## 4. Layer 4 vs Layer 7 — why it matters here

- **Layer 4 (Services)**: routing decisions are based purely on IP address and port. The load balancer/kube-proxy has zero visibility into what's inside the packet — it can't tell `example.com/api` apart from `example.com/blog`, because both look identical at L4 (same destination IP:port).
- **Layer 7 (Ingress)**: the controller actually terminates the HTTP connection, reads the `Host` header and URL path, and *then* decides where to forward — enabling routing rules that would be completely invisible at L4.

This is why Ingress must sit **in front of** Services, not replace them — Ingress hands off the final leg of the journey to a normal ClusterIP Service, which then does its usual L4 job.

---

# PART 2 — INGRESS CONTROLLERS

## 5. Ingress Controller

The controller is a normal Pod (usually run as a Deployment, exposed via its own `LoadBalancer` or `NodePort` Service) that:

1. Watches the API server for `Ingress` objects
2. Translates them into its own internal configuration (e.g., an NGINX config file, or an Envoy xDS config)
3. Reloads/updates itself to apply the new rules
4. Actually terminates and proxies the HTTP(S) traffic

Kubernetes ships **no built-in controller** — you must install one. Popular choices: **ingress-nginx** (community NGINX-based, most widely deployed), NGINX Inc.'s own controller, Traefik, HAProxy, Contour, and cloud-native ones (AWS ALB Ingress Controller, GKE Ingress).

## 6. NGINX Ingress (ingress-nginx)

The de facto default. Internally it's an NGINX process whose `nginx.conf` is regenerated every time Ingress objects change, watching via the Kubernetes API and using Lua (via OpenResty) for dynamic reloading without dropping connections.

```bash
# Minikube
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

On Minikube, the addon deploys the controller **and** configures it to bind on the node's IP, so `minikube ip` becomes your entry point.

On a real cloud cluster, you'd typically `helm install` it, and it provisions its own `LoadBalancer` Service — this is the **one and only** cloud load balancer your entire cluster needs, no matter how many apps sit behind it.

---

# PART 3 — THE FULL REQUEST PATH

```
Internet
   │
   ▼
Load Balancer               ← cloud LB (or MetalLB/minikube tunnel on-prem);
   │                          this is a plain L4 forwarder, knows nothing about HTTP
   ▼
Ingress Controller           ← real HTTP(S) server (NGINX/Envoy/etc.), terminates
   │                           TLS here, reads Host + Path, matches Ingress rules
   ▼
Service (ClusterIP)          ← ordinary L4 VIP, load-balances across Pods
   │
   ▼
Pods                         ← your application containers
```

Concretely, for `https://shop.example.com/cart`:

1. DNS resolves `shop.example.com` → the Load Balancer's public IP
2. TCP/TLS connection established to the Load Balancer, which forwards raw bytes to the Ingress Controller (L4 pass-through — the LB itself doesn't decrypt anything)
3. The Ingress Controller terminates TLS (decrypts using its configured certificate for `shop.example.com`)
4. It reads the `Host: shop.example.com` header and the path `/cart`
5. It matches these against Ingress rules, finds the target Service (e.g., `cart-service`)
6. It opens a normal internal HTTP connection to `cart-service`'s ClusterIP
7. The Service's usual kube-proxy/eBPF routing picks a ready Pod
8. The Pod handles the request and the response retraces the path

---

# PART 4 — ROUTING MECHANICS

## 7. Host-based routing

Route by the `Host` HTTP header — different domains/subdomains go to different Services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing-example
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-service
                port:
                  number: 80
    - host: blog.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: blog-service
                port:
                  number: 80
```
Any request with `Host: shop.example.com` → `shop-service`. Any with `Host: blog.example.com` → `blog-service`. Both share the same controller, same public IP, same certificate infrastructure.

## 8. Path-based routing

Route by URL path prefix under a **single** host.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing-example
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: api-service, port: { number: 8080 } }
          - path: /blog
            pathType: Prefix
            backend:
              service: { name: blog-service, port: { number: 80 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: frontend-service, port: { number: 80 } }
```
`pathType` matters:
- `Prefix` — matches `/api`, `/api/`, `/api/users`, etc. (segment-aware prefix match)
- `Exact` — matches only that literal path, nothing else
- `ImplementationSpecific` — controller decides (NGINX treats it like a regex if annotations enable that)

Order matters for overlapping prefixes — most controllers evaluate longest/most-specific match first, but always list more specific paths and verify controller behavior rather than assuming.

---

# PART 5 — TLS

## 9. TLS termination

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.example.com
      secretName: shop-tls-cert       # a Secret of type kubernetes.io/tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: shop-service, port: { number: 80 } }
```

- `tls.hosts` — which hostnames this cert block covers (must match SNI at connection time)
- `secretName` — a Kubernetes Secret holding `tls.crt` and `tls.key`
- The controller uses this to answer the TLS handshake **before** it even looks at HTTP headers (SNI happens at the TLS layer, one level below HTTP)

## 10. Certificates

You provide the cert/key yourself, or automate it. The overwhelmingly common production pattern is **cert-manager**, which watches Ingress objects (or its own `Certificate` CRDs) and automatically requests/renews certificates from Let's Encrypt via ACME.

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```
With this annotation present, cert-manager sees the Ingress, requests a cert for the listed hosts, completes an HTTP-01 or DNS-01 challenge, and writes the resulting cert straight into the `secretName` Secret referenced above — fully automated renewal thereafter.

Manually creating a TLS secret:
```bash
kubectl create secret tls shop-tls-cert --cert=cert.pem --key=key.pem
```

---

# PART 6 — MISC MECHANICS

## 11. Default backend

If a request's `Host`/path matches **no** rule in any Ingress, the controller falls back to a **default backend** — typically a simple Pod that returns a 404 page.

```yaml
spec:
  defaultBackend:
    service:
      name: default-404-service
      port:
        number: 80
```
Most controllers (ingress-nginx included) also ship their own built-in default backend if you don't specify one, which just returns a bare `404`.

## 12. Annotations

Because the Ingress **spec** is intentionally generic (portable across controllers), controller-specific behavior is bolted on via annotations — a well-known escape hatch.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/limit-rps: "10"
```
These are entirely controller-specific — an annotation meaningful to ingress-nginx does nothing on Traefik. Always check your controller's own annotation reference docs.

## 13. ingressClassName

When multiple controllers run in the same cluster (e.g., an internal one and an internet-facing one), `ingressClassName` tells Kubernetes **which controller** should implement a given Ingress object.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx    # <- must match an IngressClass name
```
An Ingress with no `ingressClassName` and no default `IngressClass` marked (`ingressclass.kubernetes.io/is-default-class: "true"`) is ignored by controllers that require explicit class matching — this is a very common "my Ingress just does nothing" bug on newer clusters.

## 14. Load balancing

Once the controller picks a Service, the actual Pod selection follows the same L4 Service load-balancing rules covered in the Services guide (round-robin by default for NGINX, configurable via annotations like `nginx.ingress.kubernetes.io/upstream-hash-by` for consistent hashing).

Note: **NGINX Ingress talks to Pod IPs directly**, bypassing kube-proxy's Service VIP entirely in most configurations (it reads Endpoints/EndpointSlices itself and load-balances across Pod IPs at the HTTP layer) — this gives it finer-grained L7 load balancing than a plain Service could offer alone.

## 15. Rewrite rules

Rewrite the path before forwarding upstream — useful when your app expects paths different from what's exposed publicly.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service: { name: api-service, port: { number: 80 } }
```
A request to `/api/users` is captured by the regex groups `(/|$)(.*)`, and `$2` (`users`) is substituted into the rewrite target — so the backend receives `/users`, not `/api/users`.

## 16. HTTP/HTTPS handling

Standard pattern: expose both port 80 and 443, and force redirect.
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"       # auto-redirect HTTP → HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```
Without a `tls:` block defined, ingress-nginx serves plain HTTP only; add `tls:` and it automatically also listens on 443 with that cert and (by default) redirects 80 → 443.

## 17. DNS relationship

Ingress has **no direct DNS integration** by default — you (or `external-dns`, a common add-on) must separately point your domain's DNS record at the Ingress Controller's external IP/hostname.

```
shop.example.com   A     <ingress-controller-external-ip>
```
With **external-dns** installed, it watches Ingress objects' `host` fields and automatically creates/updates the matching DNS records in your provider (Route53, Cloudflare, etc.) — removing the manual step.

---

# PART 7 — YAML COOKBOOK

### Single application
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: single-app
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: app-service, port: { number: 80 } }
```

### Multiple applications (host-based)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-app
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: shop-service, port: { number: 80 } } }
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: admin-service, port: { number: 80 } } }
```

### Host-based routing (repeated for clarity — see Part 4 §7)

### Path-based routing (repeated for clarity — see Part 4 §8)

### HTTPS/TLS
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: https-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [app.example.com]
      secretName: app-tls-cert
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: app-service, port: { number: 80 } } }
```

### Supporting resources for all examples above
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector: { app: demo }
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  replicas: 2
  selector: { matchLabels: { app: demo } }
  template:
    metadata: { labels: { app: demo } }
    spec:
      containers:
        - name: web
          image: hashicorp/http-echo
          args: ["-text=hello", "-listen=:8080"]
          ports: [{ containerPort: 8080 }]
```

---

# PART 8 — MINIKUBE SETUP

```bash
minikube start
minikube addons enable ingress
kubectl get pods -n ingress-nginx      # wait for controller Running

# apply your Ingress + Service + Deployment
kubectl apply -f ingress-cookbook.yaml

# Minikube doesn't have real DNS, so map the hostname locally:
echo "$(minikube ip) app.example.com" | sudo tee -a /etc/hosts

curl http://app.example.com
```
For TLS testing on Minikube, generate a self-signed cert:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout key.pem -out cert.pem -subj "/CN=app.example.com"
kubectl create secret tls app-tls-cert --cert=cert.pem --key=key.pem
curl -k https://app.example.com
```

---

# PART 9 — TROUBLESHOOTING PLAYBOOK

## 404
- No `Ingress` rule matches the request's `Host`/path → check `kubectl describe ingress` for the actual rules registered
- `ingressClassName` mismatch — the Ingress exists but **no controller is actually watching it** (see "wrong ingress class" below); the request may be hitting the controller's built-in default backend, which returns 404 for anything unmatched
- Typo in `host:` field vs the actual request's `Host` header

## 502 (Bad Gateway)
- The controller reached the Service but the **Pod** refused the connection or crashed mid-request
- `targetPort` mismatch on the underlying Service (same root cause as plain Service 502s)
- Backend Pod is overwhelmed / OOM-killed mid-request
- Check: `kubectl logs -n ingress-nginx <controller-pod>` for upstream connection errors

## 503 (Service Unavailable)
- The Service behind the Ingress has **zero ready endpoints** — same root causes as a bare Service with no endpoints (bad selector, failing readiness probes, scaled to 0)
- ingress-nginx specifically returns 503 from its own default backend when the resolved Service has no valid upstreams

## DNS failure
- The domain doesn't actually point at the Ingress Controller's external IP — verify with `dig shop.example.com` and compare to `kubectl get svc -n ingress-nginx`
- On Minikube: forgot to add the `/etc/hosts` entry (there's no real DNS in a local cluster)
- `external-dns` misconfigured or missing required cloud provider IAM permissions to write records

## TLS failure
- `secretName` doesn't exist, or exists but doesn't contain valid `tls.crt`/`tls.key` — `kubectl describe secret <name>`
- Certificate's Common Name/SAN doesn't include the requested hostname — browser/`curl` will show a hostname mismatch error
- cert-manager stuck: `kubectl describe certificate <name>` and `kubectl describe certificaterequest` to see ACME challenge failures (commonly: HTTP-01 challenge blocked because the Ingress isn't actually reachable from the internet yet)

## Wrong ingress class
- Symptom: Ingress object exists, looks perfectly correct, but **nothing happens** — no error, no logs, no traffic
- Check: `kubectl get ingress -o wide` — the `CLASS` column should show your controller's class name, not `<none>`
- Fix: set `ingressClassName` explicitly, or mark one `IngressClass` as default with the annotation `ingressclass.kubernetes.io/is-default-class: "true"`

## Service unavailable (from the controller's perspective)
- The Ingress correctly matched a rule, but the referenced **Service name/port doesn't exist** — check for typos between the Ingress's `backend.service.name` and the actual Service metadata name
- Port name/number mismatch — if using a named port, confirm the Service actually defines a port with that name

---

# PART 10 — PRODUCTION CONSIDERATIONS

1. **Run exactly one Ingress Controller Service of type `LoadBalancer`** — every app's Ingress shares it; don't provision per-app load balancers.
2. **Always set explicit `ingressClassName`.** Relying on an implicit default class is fragile once a second controller is introduced.
3. **Automate TLS with cert-manager.** Manual certificate rotation is a guaranteed future outage.
4. **Set resource requests/limits on the controller Pods** — it's now a single point of failure for *all* HTTP traffic into the cluster; under-provisioning it degrades everything at once.
5. **Run the controller with multiple replicas + PodDisruptionBudget** for HA; a single-replica Ingress Controller is a common, easily-avoidable outage cause.
6. **Rate-limit and cap body size at the Ingress layer** (`nginx.ingress.kubernetes.io/limit-rps`, `proxy-body-size`) to protect backend Pods from abuse before it ever reaches them.
7. **Prefer WAF/security annotations or a dedicated API gateway** for anything internet-facing needing more than basic routing (auth, complex rate limiting, request transformation) — Ingress is deliberately minimal by design.
8. **Separate internal and external Ingress Controllers** (two `IngressClass`es) if you have both internal-only and public-facing services — don't expose internal admin tools through the same public LB.
9. **Watch controller logs and metrics** (ingress-nginx exposes Prometheus metrics) — upstream latency and error-rate dashboards catch backend issues before users report them.
10. **Version-pin the controller** and test upgrades in staging — config-generation logic changes between major versions have historically caused subtle routing regressions.

---

# PART 11 — INTERVIEW QUESTIONS

**Conceptual**
1. What is the difference between an Ingress object and an Ingress Controller?
2. Why can't a plain `LoadBalancer` Service do what Ingress does?
3. Explain the difference between Layer 4 and Layer 7 routing in the context of Kubernetes.

**Routing**
4. Walk through how host-based routing differs from path-based routing.
5. What's the difference between `Prefix`, `Exact`, and `ImplementationSpecific` `pathType`s?
6. What happens when a request matches no rule in any Ingress?

**TLS**
7. Where does TLS termination happen in the full request path, and why does that matter for Layer 7 routing?
8. How does cert-manager automate certificate issuance for an Ingress?
9. What's a `kubernetes.io/tls` Secret expected to contain?

**Configuration**
10. What is `ingressClassName` for, and what happens if it's omitted with no default class set?
11. Why are so many Ingress features controller-specific annotations instead of part of the core spec?

**Full flow**
12. Trace a request from `Internet → Load Balancer → Ingress Controller → Service → Pod`, explaining what happens at each hop.
13. Does the Ingress Controller use kube-proxy's Service routing, or does it bypass it? Explain.

**Troubleshooting (scenario-based)**
14. Users get a 502 on one specific endpoint — walk through your debugging steps.
15. An Ingress object looks correct but traffic never reaches it — what's the first thing you check?
16. A newly requested Let's Encrypt certificate is stuck pending — what are the likely causes?

---

*This guide covers Kubernetes Ingress from the Service-vs-Ingress distinction through TLS automation, controller internals, and full production troubleshooting — the complete arc from beginner to Ingress networking expert.*


# Kubernetes Networking — Complete Mastery Guide
### From Networking Fundamentals to Production-Grade CNI Internals

---

# PART 0 — NETWORKING FUNDAMENTALS (PREREQUISITES)

Everything Kubernetes does is built on ordinary Linux networking primitives. Nothing magic happens — it's namespaces, routes, and iptables/eBPF rules, orchestrated automatically. So the fundamentals first.

## IP addresses
A 32-bit (IPv4) number identifying a network interface. `10.244.1.5` — nothing routes anywhere without one.

## Subnets
A contiguous IP range, expressed as CIDR: `10.244.1.0/24` means the first 24 bits are the network portion (`10.244.1`), leaving 256 addresses (`.0`–`.255`) in that subnet. Kubernetes carves up a large CIDR (e.g. `10.244.0.0/16`) into one `/24` per node, so each node "owns" its own slice of Pod IPs.

## Routing
When a packet's destination isn't on the local subnet, the kernel consults its **routing table** to decide which interface/gateway to forward it through.
```bash
ip route
# 10.244.2.0/24 via 10.244.0.1 dev eth0   ← "to reach node 2's Pod subnet, hop via this gateway"
```

## Ports
A 16-bit number identifying *which application* on a host a packet belongs to, on top of the IP that identifies *which host*. A socket is uniquely identified by the 4-tuple (source IP, source port, dest IP, dest port).

## NAT (Network Address Translation)
Rewriting the source and/or destination IP:port of a packet in flight. Kubernetes uses this constantly:
- **DNAT** (Destination NAT) — Service VIP → real Pod IP (this is exactly what kube-proxy's iptables/IPVS rules do)
- **SNAT** (Source NAT/masquerading) — Pod IP → node IP, needed so return traffic for internet-bound packets can find its way back through the node

## DNS
Translates names to IPs. In Kubernetes, CoreDNS plays this role cluster-wide (detailed in Part 2).

## TCP vs UDP
- **TCP** — connection-oriented, ordered, reliable (retransmits, acknowledgments); used for HTTP, gRPC, databases
- **UDP** — connectionless, no delivery guarantee, lower overhead; used for DNS queries, some metrics/streaming protocols

Kubernetes networking constructs (Services, NetworkPolicies) must be told explicitly which protocol applies — `protocol: TCP` vs `protocol: UDP` — because the underlying rules differ.

---

# PART 1 — THE KUBERNETES NETWORKING MODEL

Kubernetes imposes exactly **three non-negotiable rules** on any network implementation (this is "the Kubernetes networking model"):

1. Every Pod gets its **own unique IP** — no NAT between Pods, cluster-wide
2. **Pods can reach all other Pods'** IPs directly, across every node, without NAT
3. **Nodes can reach all Pods**, and vice versa, without NAT

This is a deliberately flat model: no port-mapping gymnastics like classic Docker networking. Every Pod behaves like it has its own real network interface directly on a giant shared L3 network — even though, physically, it's namespaced processes on a handful of physical/virtual machines. Making that illusion real is the entire job of the **CNI plugin**.

---

# PART 2 — TRAFFIC FLOW SCENARIOS

## 1. Pod-to-Pod (same node)

```
Pod A (netns)              Pod B (netns)
 eth0: 10.244.1.5           eth0: 10.244.1.6
   │  veth-a                  │  veth-b
   └──────┐              ┌────┘
          ▼              ▼
     ┌─────────────────────────┐
     │   Linux bridge (cni0)   │   ← root network namespace, on the node
     └─────────────────────────┘
```
Both Pods' `veth` (virtual ethernet) interfaces attach to a shared bridge in the node's root namespace. A packet from Pod A to Pod B never leaves the node — the bridge switches it directly, like a virtual Ethernet switch.

## 2. Pod-to-Pod (cross-node)

```
Node 1                                    Node 2
Pod A (10.244.1.5)                        Pod B (10.244.2.6)
   │                                          │
  veth ── cni0 bridge ── eth0 (node1) ══════ eth0 (node2) ── cni0 bridge ── veth
                          physical/overlay network
```
The node's route table knows "10.244.2.0/24 is reachable via node 2's IP." Depending on the CNI:
- **Overlay mode** (Flannel VXLAN): the packet gets encapsulated (wrapped in another IP/UDP packet) so it can traverse infrastructure that doesn't know about Pod IPs at all
- **Native routing mode** (Calico BGP, Cilium native routing): the underlying network fabric is taught real routes to each node's Pod CIDR — no encapsulation overhead, but requires L3 reachability/BGP peering between nodes

## 3. Pod-to-Service

Already covered in depth in the Services guide — recap: destination is a virtual IP that never appears on any interface; kube-proxy (iptables/IPVS) or eBPF rewrites it to a real Pod IP via DNAT before it's ever actually routed.

## 4. Pod-to-Internet

```
Pod (10.244.1.5) ──▶ node's root netns ──▶ SNAT (masquerade to node's public/private IP)
                                          ──▶ node's default route ──▶ Internet
```
Since `10.244.x.x` addresses are private and unroutable on the internet, the node **masquerades** (SNATs) outbound Pod traffic to look like it came from the node itself. This is a standard iptables `MASQUERADE` rule installed by the CNI plugin. Return traffic arrives at the node, gets un-SNAT'd via conntrack, and delivered back to the originating Pod.

## 5. Internet-to-Pod

This requires **inbound** exposure — a `NodePort`, `LoadBalancer`, or Ingress Controller (covered fully in prior guides). There is no default path for unsolicited internet traffic to reach a Pod; it must come in through one of these explicit doors, get DNAT'd down to a Service VIP, then again to a Pod IP.

## 6. Node-to-Pod

A process running directly on a node (e.g., a kubelet health check, or `kubectl exec` machinery) reaches a Pod IP directly via the node's local route to its own `cni0` bridge — no NAT needed, since the flat networking model guarantees direct node-to-Pod reachability.

## 7. Node-to-Service

Same DNAT mechanism as Pod-to-Service — kube-proxy's rules apply cluster/node-wide, not just to Pod-originated traffic. A node process calling a ClusterIP gets DNAT'd exactly the same way.

---

# PART 3 — DNS AND SERVICE RESOLUTION

## 8. Cluster DNS / 9. CoreDNS

CoreDNS runs as a Deployment (typically 2 replicas) in `kube-system`, exposed via its own ClusterIP Service (conventionally `10.96.0.10`). It:
- Watches Services/Endpoints via the API server
- Auto-generates DNS records for every Service (see prior guide, Part 4 §12)
- Serves as the `nameserver` for every Pod's `/etc/resolv.conf`

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system get cm coredns -o yaml    # the actual Corefile config
```

## 10. kube-proxy / Service IP / 11. EndpointSlice

Fully covered in the Services guide. Recap of the chain: **Service selector → EndpointSlice (source of truth) → kube-proxy/eBPF programs rules → DNAT rewrites Service IP to a real Pod IP.**

```bash
kubectl get endpoints backend
kubectl get endpointslices -l kubernetes.io/service-name=backend
```

---

# PART 4 — CNI: THE ENGINE UNDER EVERYTHING

## 12. CNI (Container Network Interface)

CNI is a **specification**, not a product — a simple contract: when a Pod is created, the kubelet calls a CNI plugin binary with "ADD this Pod's network namespace to the network," and the plugin is responsible for:
1. Creating a `veth` pair
2. Attaching one end to the Pod's network namespace, one end to the node's bridge/routing infra
3. Assigning the Pod an IP (from its own IPAM)
4. Setting up whatever routing/overlay/eBPF program is needed to satisfy the "any Pod reaches any Pod" model

Nothing in core Kubernetes implements Pod networking — **it is 100% delegated to whichever CNI plugin you install.** This is why "Kubernetes networking" and "your CNI's networking" are often the same conversation.

## 13. CNI plugins — the landscape

| Plugin | Data plane | Model | Notable feature |
|---|---|---|---|
| **Flannel** | iptables (or CNI defaults) | VXLAN overlay (simplest) | Easiest to set up, minimal features, no NetworkPolicy support built-in |
| **Calico** | iptables, IPVS, or eBPF (Calico's own eBPF mode) | BGP-based native routing (no overlay needed if L3 reachable), or VXLAN/IPIP overlay fallback | Rich NetworkPolicy engine, widely used in enterprise |
| **Cilium** | eBPF-native | Native routing, overlay (VXLAN/Geneve), or full kube-proxy replacement | L3–L7 NetworkPolicies, Hubble observability, service mesh-lite features, best-in-class performance |

## 14. Calico
Uses **BGP** (Border Gateway Protocol, the same protocol that runs the internet's backbone routing) to advertise each node's Pod CIDR to every other node — turning the physical network's own routers/switches (or a lightweight BGP daemon per node, `bird`/`BIRD2`) into the thing that actually routes Pod traffic, with zero encapsulation overhead when the underlying network supports it. Falls back to IPIP/VXLAN encapsulation when BGP peering across L3 boundaries isn't feasible (e.g., different cloud subnets without route propagation).

## 15. Cilium
Built entirely around **eBPF**. Instead of writing iptables rules, it attaches eBPF programs at multiple kernel hooks (XDP for earliest packet interception, tc for traffic control, and socket-layer hooks for the fastest Pod-to-Service path). This enables:
- Full kube-proxy replacement (see prior Services guide, §17)
- L7-aware NetworkPolicies (e.g., "allow GET but not DELETE on this HTTP path" — not just IP/port)
- Hubble: live, per-flow network observability without sidecars

## 16. Flannel
The simplest CNI: one UDP/VXLAN overlay tunnel between every pair of nodes, a `flanneld` daemon per node keeping a local subnet lease file that other tools (Docker/containerd) read. No NetworkPolicy enforcement of its own — commonly paired with Calico purely for policy enforcement ("Canal" = Flannel + Calico policy engine).

---

# PART 5 — LOW-LEVEL LINUX BUILDING BLOCKS

## 17. Network namespaces

A Linux kernel feature giving a process group its **own isolated network stack** — its own interfaces, routing table, iptables rules, port space. Every Pod is, under the hood, one network namespace shared by all its containers.

```bash
# find and enter a Pod's network namespace directly from the node
crictl inspect <container-id> | grep netns
nsenter --net=/var/run/netns/<ns-id> ip addr
```

## 18. veth pairs

A **virtual Ethernet pair** is two connected virtual interfaces — anything sent into one instantly appears on the other, like a virtual patch cable. One end lives inside the Pod's network namespace (appears as `eth0` to the Pod), the other end lives in the node's root namespace (visible as `veth1234abcd`), attached to the CNI bridge.

```bash
ip link | grep veth      # on the node — see every Pod's node-side veth end
```

## 19. Bridges

A Linux software switch (`cni0`, `docker0`, etc.) that all the local veth "node-side" ends attach to. It behaves like a real L2 Ethernet switch — learning MAC addresses, forwarding frames only where needed, flooding on unknowns.

```bash
ip link show cni0
bridge link show
```

## 20. Routing tables

Every node (and every Pod's own namespace) has its own routing table deciding "for this destination, use which interface/gateway."

```bash
ip route show
# default via 192.168.1.1 dev eth0
# 10.244.1.0/24 dev cni0 proto kernel scope link
# 10.244.2.0/24 via 192.168.1.12 dev eth0     ← route to another node's Pod subnet
```

## 21. iptables

The traditional Linux packet-filtering/NAT framework, structured as tables (`filter`, `nat`, `mangle`) each containing chains of rules evaluated in order. Both kube-proxy and most CNIs lean on it heavily for DNAT, masquerading, and (for Calico's non-eBPF mode) NetworkPolicy enforcement.

```bash
sudo iptables -t nat -L -n -v
sudo iptables -t filter -L -n -v
```

## 22. eBPF

Extended Berkeley Packet Filter — a way to run small, verified, sandboxed programs **directly in the kernel**, attached to hooks like network interfaces, syscalls, or socket operations, without writing a kernel module. This is the technology underpinning Cilium and modern kube-proxy-replacement modes — faster than iptables because rules become compiled, indexed kernel bytecode instead of a sequentially-walked rule list.

```bash
# on a Cilium node:
cilium status
cilium bpf lb list       # inspect eBPF-programmed load-balancing entries directly
```

---

# PART 6 — NETWORK POLICY

## 23. NetworkPolicy

By default, **all Pods can reach all other Pods**, cluster-wide, with zero restrictions — the flat model has no built-in security boundary. `NetworkPolicy` objects add **allow-list** firewall rules, enforced by the CNI (not all CNIs support this — Flannel alone does not; Calico and Cilium do).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels: { app: backend }        # this policy applies TO Pods labeled app=backend
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: frontend } }   # only frontend Pods may connect in
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector: { matchLabels: { app: database } }   # backend may only call database
      ports:
        - protocol: TCP
          port: 5432
    - to:                                                     # ...and DNS, always needed
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

**Line-by-line:**
- `podSelector` — which Pods this policy governs (the *target*, not the source)
- `policyTypes` — declares this policy restricts both inbound (`Ingress`) and outbound (`Egress`) traffic; **once any policy selects a Pod for a direction, that direction becomes default-deny** for that Pod except for what's explicitly allowed
- `ingress[].from` — allowed sources (by Pod label, namespace label, or CIDR block)
- `egress[].to` — allowed destinations
- The DNS egress rule is easy to forget — without it, once you lock down egress, DNS resolution silently breaks because port 53 to CoreDNS is no longer implicitly allowed

## 24. Ingress/Egress traffic direction

- **Ingress** (in a NetworkPolicy context) = traffic **coming into** the selected Pod
- **Egress** = traffic **leaving** the selected Pod

This is a different meaning from the `Ingress` *object* covered in the previous guide — same word, different layer (L7 HTTP routing object vs L3/L4 firewall direction). Don't conflate them.

---

# PART 7 — FULL PACKET-FLOW WALKTHROUGH

### Scenario: Pod A (node 1) calls a Service backed by a Pod on node 2, over an overlay CNI (e.g. Flannel VXLAN)

```
1. Pod A's app opens a socket to Service VIP 10.96.5.5:80
   → packet leaves Pod A's netns via its veth, arrives at node 1's cni0 bridge

2. Node 1's netfilter PREROUTING hook (kube-proxy's iptables rules) DNATs:
   10.96.5.5:80 → 10.244.2.9:8080   (a Pod living on node 2)

3. Node 1 checks its routing table for 10.244.2.0/24
   → route says "via VXLAN tunnel to node 2"

4. Node 1's flanneld-programmed VXLAN interface (flannel.1) ENCAPSULATES
   the packet: wraps the original packet (still addressed to 10.244.2.9)
   inside a new UDP packet addressed to node 2's real IP, port 8472

5. This outer packet travels across the real physical/cloud network
   (which only understands node IPs — 10.244.x.x is invisible to it)

6. Node 2's flannel.1 interface receives the UDP packet, DECAPSULATES it,
   recovering the original packet addressed to 10.244.2.9:8080

7. Node 2 routes the now-decapsulated packet to its local cni0 bridge
   → delivered to Pod B's veth → arrives inside Pod B's netns as if it
     arrived directly, no NAT visible to the application

8. Pod B's app responds; the reverse path un-DNATs (via conntrack on
   node 1) so Pod A sees a reply that looks like it came straight from
   10.96.5.5:80 — Pod A never learns Pod B's real identity or that
   an encapsulation tunnel was involved at all
```

### Same scenario under Cilium (eBPF, native routing, no overlay)
```
1. Pod A's app calls connect() to 10.96.5.5:80
   → an eBPF program attached at the SOCKET layer intercepts this
     BEFORE a packet is even built, and rewrites the destination
     directly to 10.244.2.9:8080 (chosen via the eBPF-maintained
     load-balancing map, itself fed by EndpointSlice updates)

2. The packet is constructed already correctly addressed —
   no DNAT hop, no conntrack table needed for this leg

3. Node 1 routes to 10.244.2.0/24 via a REAL route (BGP/native,
   no encapsulation) straight to node 2's interface

4. Node 2's eBPF program (tc hook) receives it, hands it to Pod B's
   veth → delivered

5. Response follows a symmetric, equally short path
```
Compare the two: the eBPF/native-routing path skips both the DNAT rewrite *and* the encapsulation/decapsulation overhead entirely — this is the concrete performance story behind "eBPF is faster."

---

# PART 8 — COMMAND REFERENCE

```bash
# --- Interfaces & addresses ---
ip addr                          # show all interfaces + IPs (run inside a Pod's netns or on a node)
ip route                         # show the routing table
ip link                          # show interfaces, including veth pairs on the node

# --- Sockets & connections ---
ss -tlnp                         # listening TCP sockets with owning process
ss -tunap                        # all TCP+UDP sockets, numeric, with process info

# --- Entering a Pod's network namespace directly from the node ---
crictl inspect <container-id> | grep -i pid
nsenter -t <pid> -n ip addr      # view the Pod's network namespace from outside, without kubectl exec

# --- iptables inspection ---
sudo iptables -t nat -L KUBE-SERVICES -n
sudo iptables -t nat -L -n -v --line-numbers

# --- kubectl-native debugging ---
kubectl exec -it <pod> -- sh -c "ip addr && ip route"
kubectl get endpoints <svc>
kubectl get endpointslices -l kubernetes.io/service-name=<svc>
kubectl get networkpolicy -A
kubectl describe pod <pod>       # check assigned Pod IP, node, events
```

---

# PART 9 — TROUBLESHOOTING LABS

### Lab 1: Trace a Pod's own network identity
```bash
kubectl run test --image=busybox:1.36 --restart=Never -it -- sh
ip addr show eth0
ip route
cat /etc/resolv.conf
exit
```

### Lab 2: Watch the DNAT rule for a Service live
```bash
kubectl expose deployment demo --port=80 --target-port=8080 --name=demo-svc
minikube ssh
sudo iptables -t nat -L -n | grep demo-svc
```

### Lab 3: Break DNS with an egress NetworkPolicy (and understand why)
```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: deny-all-egress }
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress: []
EOF
kubectl exec -it test -- nslookup kubernetes.default   # now fails — no DNS egress allowed
# fix: add an egress rule allowing UDP/53 to kube-system, or delete the policy to confirm
kubectl delete networkpolicy deny-all-egress
```

### Lab 4: Cross-node Pod-to-Pod trace with nsenter
```bash
# find two Pods scheduled on different nodes
kubectl get pods -o wide
# on the node hosting Pod A:
crictl ps | grep <pod-a-container>
nsenter -t <pid> -n ping -c3 <pod-b-ip>
# watch it succeed with zero NAT visible from inside the namespace
```

### Lab 5: Simulate a broken CNI route
```bash
# on a node, temporarily delete the route to another node's Pod subnet
sudo ip route del 10.244.2.0/24
kubectl exec -it test -- ping <pod-on-node-2>   # fails — timeout
sudo ip route add 10.244.2.0/24 via <node2-ip>  # restore manually (CNI normally manages this)
```

---

# PART 10 — TROUBLESHOOTING SCENARIOS (PRODUCTION)

| Symptom | Likely cause | First checks |
|---|---|---|
| Pod stuck in `ContainerCreating`, events show CNI errors | CNI plugin crash, IP pool exhausted, misconfigured CNI config file | `kubectl describe pod`, CNI daemonset logs (`calico-node`, `cilium`), `/etc/cni/net.d/` on the node |
| Cross-node Pod traffic times out, same-node traffic works | Overlay tunnel down, BGP peering broken, MTU mismatch causing silent packet drops on encapsulated traffic | `ip route` on both nodes, `cilium status`/`calicoctl node status`, check MTU consistency (a very common overlay bug) |
| DNS intermittently fails under load | CoreDNS under-provisioned (too few replicas/CPU) for query volume, or `ndots:5` amplifying query count | `kubectl top pod -n kube-system`, CoreDNS `Corefile` cache settings, consider `dnsConfig` tuning |
| Egress to the internet fails after enabling NetworkPolicy | Default-deny egress with no explicit DNS/internet allow rule | Check `kubectl get networkpolicy`, verify DNS (UDP/53) and destination CIDR are explicitly allowed |
| One node's Pods can't reach Services, others fine | kube-proxy Pod on that node crashed/stale rules | `kubectl -n kube-system get pods -o wide -l k8s-app=kube-proxy`, restart the Pod on that node |
| High latency only on Service (VIP) traffic, direct Pod IP traffic is fine | iptables mode at high Service count (linear rule evaluation) | Check kube-proxy mode (`kubectl -n kube-system get cm kube-proxy -o yaml`), consider migrating to IPVS or eBPF |
| NetworkPolicy applied but traffic still not blocked | CNI doesn't enforce NetworkPolicy at all (e.g., plain Flannel) | Verify CNI supports policy enforcement; Flannel alone requires pairing with Calico's policy engine |

---

# PART 11 — COMPLETE NETWORKING DIAGRAM

```
                                   INTERNET
                                       │
                            ┌──────────▼──────────┐
                            │   Cloud Load Balancer │  (L4 pass-through)
                            └──────────┬──────────┘
                                       │
                            ┌──────────▼──────────┐
                            │  Ingress Controller  │  (L7: Host/Path/TLS)
                            └──────────┬──────────┘
                                       │  plain HTTP to Service
   ┌───────────────────────────────────┼───────────────────────────────────┐
   │  CLUSTER                          ▼                                    │
   │                        ┌─────────────────────┐                        │
   │            DNS ──────▶ │  Service (ClusterIP) │ ◀── kube-proxy/eBPF   │
   │         (CoreDNS)      │  10.96.x.x  (virtual) │     programs DNAT     │
   │                        └──────────┬───────────┘                        │
   │                                   │ EndpointSlice-driven                │
   │            ┌──────────────────────┼──────────────────────┐            │
   │            ▼                      ▼                      ▼            │
   │   ┌─────────────────┐   ┌─────────────────┐    ┌─────────────────┐   │
   │   │ Node 1           │   │ Node 1 (same)    │    │ Node 2           │   │
   │   │ ┌─────┐          │   │ ┌─────┐          │    │ ┌─────┐          │   │
   │   │ │Pod A│──veth────┤   │ │Pod B│──veth────┤    │ │Pod C│──veth────┤   │
   │   │ └─────┘   \      │   │ └─────┘   /      │    │ └─────┘    \     │   │
   │   │         cni0 bridge (L2 switch)         │    │          cni0    │   │
   │   └────────────┬─────┘   └──────────────────┘    └────────┬─────────┘   │
   │                │  CNI: overlay (VXLAN/Geneve) or native routing (BGP/eBPF) │
   │                └───────────────────────────────────────────┘             │
   │                          physical/virtual node network                    │
   └─────────────────────────────────────────────────────────────────────────┘
```

---

# PART 12 — PACKET-FLOW CHEAT SHEET

```
Pod-to-Pod (same node):     veth → cni0 bridge → veth                    (pure L2, no NAT)
Pod-to-Pod (cross node):    veth → cni0 → node route → [encap?] → peer node → cni0 → veth
Pod-to-Service:             veth → DNAT (iptables/IPVS/eBPF) → real Pod IP → (as above)
Pod-to-Internet:            veth → cni0 → SNAT (masquerade to node IP) → default route → Internet
Internet-to-Pod:            LB → NodePort/Ingress → DNAT to Service VIP → DNAT to Pod IP
Node-to-Pod:                node root netns → direct route to cni0 → veth (no NAT)
Node-to-Service:            node root netns → DNAT (same as Pod-to-Service) → Pod IP

Key commands per hop:
  See Pod's own view:        kubectl exec -it <pod> -- ip addr / ip route
  See node's DNAT rules:     iptables -t nat -L -n  (or `cilium bpf lb list` for eBPF)
  See node's Pod routes:     ip route
  See cross-node reachability: ping / nsenter into netns directly
  See what Service resolves to: kubectl get endpointslices -l kubernetes.io/service-name=<svc>
  See DNS resolution path:   cat /etc/resolv.conf ; nslookup <svc>
  See policy enforcement:    kubectl get networkpolicy -A ; describe target Pod
```

---

# PART 13 — INTERVIEW QUESTIONS

**Fundamentals**
1. What are the three rules of the Kubernetes networking model?
2. Why does Pod-to-Internet traffic require SNAT but Pod-to-Pod traffic doesn't?
3. Explain the difference between DNAT and SNAT with a Kubernetes example of each.

**CNI**
4. What exactly does a CNI plugin do when a Pod is created?
5. Compare Flannel, Calico, and Cilium in terms of data plane and routing model.
6. Why does Calico's BGP mode avoid encapsulation overhead, and when does it fall back to overlay?
7. What makes Cilium's eBPF approach faster than a traditional iptables-based CNI?

**Low-level mechanics**
8. What is a network namespace, and how does it relate to a Pod?
9. Explain what a veth pair is and how it connects a Pod to a node's bridge.
10. Why is `nsenter` useful for debugging Pod networking, and how does it differ from `kubectl exec`?

**NetworkPolicy**
11. What is the default Pod-to-Pod connectivity behavior with zero NetworkPolicies applied?
12. Why must you explicitly allow DNS egress once you apply a default-deny egress policy?
13. Does every CNI enforce NetworkPolicy? Give an example of one that doesn't by default.

**Full-flow / scenario**
14. Trace a packet end-to-end from a Pod on Node 1 to a Pod on Node 2 under a VXLAN overlay CNI.
15. How does that same trace change under Cilium's eBPF socket-layer load balancing?
16. Why might cross-node Pod traffic silently drop or perform poorly due to MTU settings?

**Troubleshooting**
17. Same-node Pod traffic works, but cross-node traffic fails — what do you check first?
18. DNS works for internal Services but times out for external domains — what's a likely cause?
19. A NetworkPolicy is applied and traffic is still not blocked — what's your first hypothesis?

---

*This guide covers Kubernetes networking from IP/subnet/routing fundamentals through CNI internals (Flannel, Calico, Cilium), full packet-flow tracing, NetworkPolicy, and production troubleshooting — the complete arc from beginner to CNI-level networking specialist.*

This is an enormous scope — full first-principles teaching on both ConfigMaps and Secrets at the depth you just saw in Incident 1's reveal, **plus 115 interview questions each requiring 8 distinct sub-answers**, plus external secret manager coverage, a security checklist, and a mock interview. Done properly (not compressed), this is a multi-part chapter, not a single response.

Here's how I'll sequence it so nothing gets shallow:

- **Part 1 (this message):** ConfigMaps, complete — purpose through immutability, at full depth
- **Part 2:** Secrets, complete — purpose through projected tokens and RBAC
- **Part 3:** base64-vs-encryption deep dive, encryption at rest, external secret managers (AWS Secrets Manager, Vault, External Secrets Operator)
- **Part 4:** Security risks, troubleshooting scenarios, security checklist
- **Part 5–8+:** The interview bank, broken into beginner → intermediate → advanced → scenario/troubleshooting/architecture batches (115 questions × 8 parts each is genuinely too large to compress — I'll do roughly 15–20 questions per installment at full depth)
- **Final:** Mock interview

Say "continue" after each part. Starting now with Part 1.

---

# ConfigMaps & Secrets — Complete Security Masterclass
## Part 1 of 8+: ConfigMaps, From First Principles

---

# 1. THE PROBLEM BEFORE THE SOLUTION

Before defining ConfigMap, you need to understand the problem it exists to solve — otherwise the definition is just words to memorize.

**A container image is a packaged, immutable bundle** — define "immutable" here since it matters: it means *unchangeable once built*. When you build a container image, you bake your application code and its dependencies into a fixed set of filesystem layers. That image file does not change after it's built.

Here's the problem this creates: imagine your application needs to know the URL of its database. If you write that URL directly into your application code before building the image, then:

```
┌──────────────────────────────────────────────────────────────┐
│  WITHOUT external configuration:                                │
│                                                                    │
│  App code:  db_url = "postgres://dev-db.internal:5432/myapp"      │
│                          │                                          │
│                          ▼                                          │
│                  docker build → image "myapp:1.0"                    │
│                          │                                             │
│         ┌────────────────┼────────────────┐                            │
│         ▼                 ▼                 ▼                            │
│    Deploy to DEV     Deploy to STAGING   Deploy to PROD                    │
│    (needs dev-db)     (needs staging-db)  (needs prod-db)                   │
│         │                 │                 │                                │
│         ▼                 ▼                 ▼                                │
│    Works fine        WRONG DATABASE!    WRONG DATABASE!                       │
│                       (still pointing    (still pointing                       │
│                        at dev-db,          at dev-db,                           │
│                        baked into the       baked into the                        │
│                        image)                image)                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

You'd have to **rebuild the entire image**, three separate times, with three different hardcoded URLs, just to run in three environments. That's slow, error-prone, and means your dev database's address is permanently frozen inside a file that gets pushed to a container registry — a registry that staging and production servers might also have access to.

**The fix, conceptually:** separate *what your application does* (the code, baked into the image) from *how it's configured for this specific place it's running* (the database URL, which changes per environment). The code stays identical everywhere; only the configuration changes at deployment time.

**ConfigMap is Kubernetes's built-in mechanism for this separation, for non-sensitive configuration data.** (We'll get to *why* "non-sensitive" matters — that's the entire reason Secrets exist as a separate thing, covered in Part 2.)

---

# 2. WHAT A CONFIGMAP IS — DEFINITION, THEN DIAGRAM

**Definition:** A ConfigMap is a Kubernetes object that stores configuration data as a set of key-value pairs, which can then be injected into Pods in several different ways, without that data ever being baked into a container image.

```
┌─────────────────────────────────────────────────────────────┐
│                     ConfigMap object                            │
│   (lives in etcd, part of the cluster's stored state,             │
│    exactly like any other Kubernetes object — a Deployment,        │
│    a Service, a Pod are all stored the same way)                     │
│                                                                        │
│   key: DATABASE_URL     value: "postgres://prod-db:5432/myapp"         │
│   key: LOG_LEVEL         value: "info"                                   │
│   key: MAX_CONNECTIONS    value: "100"                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │  referenced by name from a Pod's spec
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                            Pod                                          │
│   "Please give me the values from ConfigMap X" (several ways to ask,      │
│    covered below: as env vars, or as files)                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

**Term to define: "etcd."** etcd is the database Kubernetes uses to store literally everything about the cluster's state — every Pod, every Deployment, every ConfigMap, every Secret. When you "create" any Kubernetes object, what actually happens is: your request goes to the Kubernetes API server (the front door to the cluster), which validates it and writes it into etcd. A ConfigMap, structurally, is just another record in that same database, no different in kind from a Deployment record.

---

# 3. THE FULL FLOW: Pod → ConfigMap/Secret → Application

You asked for this flow explicitly — here it is at the right level of detail, with every step named:

```
┌───────────────────────────────────────────────────────────────────────┐
│  STEP 1 — SOMEONE CREATES THE CONFIGMAP                                   │
│  kubectl apply -f configmap.yaml                                            │
│         │                                                                     │
│         ▼                                                                      │
│  API server validates the YAML, writes it into etcd as a stored object          │
└───────────────────────────────────────────────────────────────────────────┘
                                     │
┌───────────────────────────────────────────────────────────────────────┐
│  STEP 2 — SOMEONE CREATES A POD THAT REFERENCES IT                          │
│  The Pod's spec contains a REFERENCE — just the ConfigMap's NAME,             │
│  not a copy of its data                                                          │
└───────────────────────────────────────────────────────────────────────────┘
                                     │
┌───────────────────────────────────────────────────────────────────────┐
│  STEP 3 — THE POD GETS SCHEDULED TO A NODE                                    │
│  (A "node" is a physical or virtual machine that's part of the cluster           │
│   and actually runs containers. "Scheduling" is the process of the                │
│   control plane deciding which node a Pod should run on.)                          │
└───────────────────────────────────────────────────────────────────────────┘
                                     │
┌───────────────────────────────────────────────────────────────────────┐
│  STEP 4 — THE KUBELET ON THAT NODE PREPARES TO START THE CONTAINER            │
│  Before starting the actual container process, the kubelet:                       │
│    a. Reads the Pod's spec, sees it references ConfigMap "app-config"               │
│    b. Fetches the CURRENT content of that ConfigMap from the API server               │
│       (kubelet never reads etcd directly — only through the API server,                │
│        which is the single controlled gateway to all cluster data)                       │
│    c. Builds the environment variables and/or files based on that content                 │
└───────────────────────────────────────────────────────────────────────────┘
                                     │
┌───────────────────────────────────────────────────────────────────────┐
│  STEP 5 — THE CONTAINER PROCESS ACTUALLY STARTS                               │
│  The application code inside the container reads:                                 │
│    - Environment variables (if using env/envFrom), OR                              │
│    - Files at a mount path (if using a volume mount)                                 │
│  From the APPLICATION's point of view, it has no idea these values came              │
│  from a ConfigMap at all — it just sees env vars or files, like any                    │
│  normal program running on any normal machine                                            │
└───────────────────────────────────────────────────────────────────────────┘
```

**Why is this worth drawing out this explicitly?** Because the most important fact in this whole diagram is: **the application never knows Kubernetes exists.** This is a deliberate design principle. Your Python/Java/Go code just calls `os.environ["DATABASE_URL"]` or opens a file — completely ordinary operations any program does on any operating system. Kubernetes's entire job is making sure the *right values* are sitting there, waiting, by the time your code asks for them.

---

# 4. ConfigMap `data` — THE BASIC BUILDING BLOCK

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: prod
data:
  DATABASE_URL: "postgres://prod-db.internal:5432/myapp"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  app.properties: |
    server.port=8080
    server.timeout=30s
```

**Explaining every line, per your rule 9:**

- `apiVersion: v1` — every Kubernetes object type belongs to an "API group" and version. ConfigMap is one of the original, foundational object types, so it lives in the plain `v1` core group (no prefix needed, unlike newer object types which use something like `apps/v1`).
- `kind: ConfigMap` — tells Kubernetes what *type* of object this YAML describes. This single word is what causes the API server to treat this data as a ConfigMap rather than, say, a Pod or a Service.
- `metadata.name: app-config` — the unique name this object will be known by, within its namespace. This is the name a Pod will reference later.
- `metadata.namespace: prod` — **term to define:** a "namespace" in Kubernetes is a way of partitioning a cluster into separate logical sections, so that names don't collide and access can be controlled per-section. A ConfigMap named `app-config` in namespace `prod` is a completely different object from one named `app-config` in namespace `staging` — same name, unrelated objects.
- `data:` — the actual key-value pairs. Every value here **must be a string** — this is a hard rule worth calling out as an edge case (rule 23): even `MAX_CONNECTIONS: "100"` is stored as the *text* "100", not the number 100. Kubernetes ConfigMaps have no concept of numeric or boolean types — everything is text.
- `app.properties: |` — this is worth pausing on, because it looks different from the others. The `|` symbol (a YAML feature, not a Kubernetes-specific one) means "everything indented below this, keep exactly as multi-line text, preserving line breaks." This lets you store an **entire configuration file's content** as the value of a single key. This is precisely how you'd ship a full `nginx.conf` or `application.properties` file through a ConfigMap — one key, whose value is the file's entire text content.

**WHY does this file-content-as-a-single-value trick matter?** Because many applications don't read individual environment variables at all — they expect to open an actual config *file* on startup. ConfigMap supports both patterns (individual key-value pairs for env vars, or whole files for volume mounts), and this is exactly why both patterns exist side by side in the same object type — different applications need different consumption styles.

---

# 5. CONSUMING A CONFIGMAP AS ENVIRONMENT VARIABLES — TWO WAYS

## 5a. `env` + `valueFrom` — one variable at a time, explicit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: prod
spec:
  containers:
    - name: app
      image: myapp:1.0
      env:
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_URL
```

**Line by line:**
- `env:` — a list of individual environment variables to set inside this specific container.
- `name: DATABASE_URL` — this is the name the environment variable will have **inside the running container**. Important: this name is chosen here, independently of what the ConfigMap calls its key — though in this example they happen to match.
- `valueFrom:` — instead of a hardcoded `value:`, this says "don't give me a literal string — go fetch this from somewhere else."
- `configMapKeyRef:` — specifically: fetch it from a ConfigMap key.
- `name: app-config` — which ConfigMap object to look in.
- `key: DATABASE_URL` — which key, inside that ConfigMap's `data`, to pull the value from.

**What Kubernetes is doing internally, step by step, WHAT not WHY (WHY was covered in section 3):**
```
1. kubelet reads this Pod spec
2. Sees env[0] wants its value sourced from configMapKeyRef
3. kubelet calls the API server: "give me ConfigMap 'app-config' in namespace 'prod'"
4. API server returns the ConfigMap's current data
5. kubelet looks up the specific key "DATABASE_URL" within that data
6. kubelet takes that value and constructs an environment variable
   named "DATABASE_URL" (per THIS Pod's env[0].name field) with that value
7. This full env var list is handed to the container runtime, which
   starts the container process with these as its environment
```

**Why choose this method over the bulk method below?** Because the *name inside the container* and the *key in the ConfigMap* are independently specified. This means you can name your ConfigMap key however you like (perhaps matching a company-wide naming convention) while still delivering it to the application under whatever name *that specific application's code* expects — exactly the fix (Option C) from Incident 1's teardown.

## 5b. `envFrom` — bulk import, every key at once

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: prod
spec:
  containers:
    - name: app
      image: myapp:1.0
      envFrom:
        - configMapRef:
            name: app-config
```

**Line by line:**
- `envFrom:` — different from `env:` — this doesn't let you name variables individually; it says "take *everything* in the referenced object and turn each key into an identically-named environment variable."
- `configMapRef:` — (note: no `Key` in this name, unlike `configMapKeyRef` above — because you're not selecting one key, you're taking all of them)
- `name: app-config` — which ConfigMap to pull everything from.

**Result inside the container**, given the ConfigMap from section 4:
```
DATABASE_URL=postgres://prod-db.internal:5432/myapp
LOG_LEVEL=info
MAX_CONNECTIONS=100
```
(Note: `app.properties` — the multi-line file content key — is **silently skipped** here. This is an edge case worth knowing explicitly: `envFrom` only works for keys whose names are valid environment variable names, and even then, complex multi-line values don't make practical sense as a single environment variable. Kubernetes emits a warning event in this situation rather than crashing anything.)

**Trade-off, stated plainly, distinguishing production guidance from Kubernetes behavior (rule 17):** `envFrom` is a **convenience**, not a Kubernetes requirement to use it. It's faster to write when a ConfigMap has many keys. The production-best-practice trade-off (not a Kubernetes rule) is: `envFrom` hides the name-mapping entirely, which is exactly the class of bug from Incident 1. Many security-conscious teams prefer explicit `env`/`valueFrom` specifically so every mapping is visible on inspection.

---

# 6. CONSUMING A CONFIGMAP AS A VOLUME MOUNT

**First, define "volume" and "mount," since these terms get thrown around loosely.** A **volume**, in Kubernetes, is a storage location that can be attached to a Pod. A **mount** (or "mounting") is the act of making that storage appear at a specific path inside a container's filesystem — the same general concept as plugging in a USB drive and having it show up as a folder on your computer.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: prod
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/myapp/config
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

**Line by line:**
- `spec.volumes:` — declared at the **Pod** level (not inside the container) — this is because a volume is a resource the whole Pod has access to; multiple containers within one Pod could mount the same volume, which is exactly the shared-volume pattern from earlier masterclasses.
- `name: config-volume` — an internal label for this volume, used only to link it to a `volumeMounts` entry below — this name has no meaning outside this Pod spec.
- `configMap.name: app-config` — says this volume's content comes from the ConfigMap named `app-config`.
- `spec.containers[0].volumeMounts:` — declared **inside** the specific container that wants access to it.
- `name: config-volume` — must match the volume name declared above — this is how Kubernetes knows *which* volume this mount refers to.
- `mountPath: /etc/myapp/config` — the directory path, inside the container's filesystem, where this volume's content will appear.

**What actually appears at `/etc/myapp/config` inside the running container:**
```
/etc/myapp/config/
├── DATABASE_URL       (a file, containing the text: postgres://prod-db.internal:5432/myapp)
├── LOG_LEVEL           (a file, containing the text: info)
├── MAX_CONNECTIONS      (a file, containing the text: 100)
└── app.properties        (a file, containing the full multi-line properties content)
```

**Every single key in the ConfigMap's `data` becomes a separate file**, named after the key, whose entire content is that key's value. This is a genuinely different consumption model from environment variables — worth an explicit diagram since this is exactly the kind of "commonly confused" concept your rules ask me to compare directly:

```
┌───────────────────────────── ConfigMap "app-config" ─────────────────────────────┐
│  data:                                                                              │
│    DATABASE_URL: "postgres://..."                                                    │
│    LOG_LEVEL: "info"                                                                   │
└─────────────────────┬───────────────────────────────┬─────────────────────────────┘
                        │                               │
             AS ENV VARS (envFrom)              AS VOLUME MOUNT
                        │                               │
                        ▼                               ▼
        ┌───────────────────────────┐    ┌───────────────────────────────┐
        │ Inside container process's  │    │ Inside container's filesystem:  │
        │ environment:                  │    │   /etc/myapp/config/              │
        │   DATABASE_URL=postgres://...  │    │     ├── DATABASE_URL (a FILE,       │
        │   LOG_LEVEL=info                 │    │     │   containing that text)        │
        │                                    │    │     └── LOG_LEVEL (a FILE,            │
        │ App reads via:                       │    │         containing "info")             │
        │   os.environ["DATABASE_URL"]           │    │                                          │
        │                                          │    │ App reads via:                            │
        │                                            │    │   open("/etc/myapp/config/LOG_LEVEL")     │
        └───────────────────────────────────────┘    └───────────────────────────────────────────┘
```

**Why would you choose the volume-mount approach at all, if env vars are simpler?** Two real reasons:
1. Your application expects to read an actual **file** on startup (many applications, especially anything using a mature config-file-based framework, are written this way and can't easily be changed to read env vars instead).
2. **Volume-mounted ConfigMaps update automatically, without restarting the Pod** — covered fully in the next section, and this is the single biggest practical difference between the two methods.

---

# 7. UPDATES — WHAT HAPPENS WHEN A CONFIGMAP CHANGES

This is one of the most misunderstood areas in all of Kubernetes, and you specifically asked me to cover "updates," so let's be exhaustive here (per rule 23, don't skip edge cases).

## Case 1: environment variables (`env` or `envFrom`) — DO NOT update automatically

```
Timeline:

t=0:    Pod starts. kubelet fetches ConfigMap, builds env vars,
        passes them to the container process at startup.
        Container process now has DATABASE_URL="old-value" sitting
        in its own memory, as part of its process environment.

t=10m:  Someone runs: kubectl apply -f updated-configmap.yaml
        (changes DATABASE_URL to "new-value")

        The ConfigMap OBJECT in etcd is now updated.
        The RUNNING CONTAINER PROCESS's environment is UNCHANGED —
        it already copied the old value into its own memory at t=0,
        and nothing tells it to go re-check.

t=15m:  Application still uses "old-value" — completely unaware
        anything changed, because nothing re-executed the code
        that reads os.environ["DATABASE_URL"]
```

**WHY does this happen?** This is a fact about how operating systems work in general, not something Kubernetes specifically chose to make difficult. Environment variables are copied into a process's memory **once**, at the moment that process starts. This is true for *any* program on *any* operating system, not a Kubernetes-specific limitation — Kubernetes is simply not exempt from this fundamental OS behavior. Once the container process is running, Kubernetes has no mechanism (and no OS-level mechanism exists) to reach into a running process's memory and silently change one of its environment variables.

**The only way to make the application see the new value:** restart the container process entirely, so it starts fresh and re-reads the (now-updated) ConfigMap.
```bash
kubectl rollout restart deployment/myapp -n prod
```

## Case 2: volume mounts — DO update, but with two important caveats

```
Timeline:

t=0:    Pod starts. Volume mounted at /etc/myapp/config, files
        written with current ConfigMap content.

t=10m:  ConfigMap updated via kubectl apply.

t=10m + (up to ~60-90 seconds, kubelet's periodic sync interval):
        kubelet notices the ConfigMap changed, and updates the
        FILES on disk inside the container to reflect the new content.
```

**Caveat 1 — this is not instant.** The kubelet checks for ConfigMap changes on a periodic sync cycle (roughly once per minute, though the exact interval is an implementation detail, not a guaranteed contract — per rule 16, distinguishing guaranteed behavior from implementation specifics). There will always be a real, if short, delay.

**Caveat 2 — the FILE updates, but the APPLICATION might not notice.** This is the part almost everyone misses. Updating the file on disk is a filesystem-level change. Whether your *application* notices that change depends entirely on whether your application code is written to watch that file for changes (using something like a filesystem watcher) and reload its configuration — most applications simply read a config file **once, at startup**, and never look at it again for the rest of their process lifetime, exactly like the environment variable case above, just one layer removed.

```
┌────────────────────────────────────────────────────────────────┐
│  What DOES update automatically: the FILE CONTENT on disk           │
│  What does NOT automatically update: the APPLICATION'S IN-MEMORY      │
│  understanding of that config, UNLESS the app explicitly re-reads it     │
└────────────────────────────────────────────────────────────────────┘
```

## Common misconception, stated explicitly (rule 18)

**Misconception:** "Volume-mounted ConfigMaps are 'live' — my app will always see the latest config."
**Reality:** Only the *file* is live. Your *application* is live only if you specifically wrote it to watch for file changes. This is an application-code responsibility, not something Kubernetes does for you.

## Comparison table — env vars vs volume mounts, for updates specifically

| | Environment variables | Volume mount |
|---|---|---|
| Does the underlying value change when the ConfigMap changes? | Never — frozen at container start, forever, until restart | Yes, the file content updates (with a short delay) |
| Does the running application automatically see the new value? | Never, without a restart | Only if the app itself watches the file and reloads |
| Delay before change is visible at all | N/A (never happens without a restart) | ~60-90 seconds typically, not instant |
| Common production trick to force a real update | `kubectl rollout restart` | Same — if the app doesn't self-reload, still requires a restart |

---

# 8. IMMUTABLE CONFIGMAPS

**Define "immutable" again, in this specific context:** an immutable ConfigMap is one that, once created, cannot be edited — Kubernetes will reject any attempt to change its `data` field.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v3
  namespace: prod
data:
  DATABASE_URL: "postgres://prod-db.internal:5432/myapp"
  LOG_LEVEL: "info"
immutable: true
```

**The one new line:** `immutable: true`. Once this ConfigMap is created with this field set, trying to `kubectl apply` a change to its `data` will fail outright with an error — the API server actively rejects it.

**WHY would you want this?** Two genuinely separate reasons, worth distinguishing:

**Reason 1 — a real performance mechanism (WHAT/internal, not opinion).** Kubernetes normally has to continuously **watch** every ConfigMap for changes, in case any Pod using it via a volume mount needs its files updated (per section 7). "Watching" here means: the API server keeps an open connection to every kubelet that might care, ready to push a notification the instant something changes. In a very large cluster, with thousands of ConfigMaps, this watching mechanism consumes real API server resources. If a ConfigMap is marked `immutable: true`, Kubernetes knows, with certainty, that it will *never* need to push an update for this object — so it can skip setting up that watch entirely, reducing load on the API server. This is a genuine internal optimization, not a matter of opinion or best practice — it's a documented mechanical effect.

**Reason 2 — a safety practice (production guidance, not a Kubernetes mandate — rule 17 applies).** Making critical, version-pinned configuration immutable prevents someone from accidentally running `kubectl edit configmap app-config` directly against a live production ConfigMap and silently changing behavior for every Pod using it, with no review process. The common pattern this enables: instead of editing a ConfigMap in place, you create a brand-new one with a version suffix (`app-config-v3`, `app-config-v4`) each time config needs to change, and update your Deployment to reference the new name — which then goes through your normal deployment/rollout process, complete with the usual visibility and rollback options, rather than a silent, untracked in-place edit.

## What happens if you try to edit an immutable ConfigMap anyway

```bash
$ kubectl edit configmap app-config-v3 -n prod
```
Expected result:
```
error: configmaps "app-config-v3" is invalid: data: Forbidden:
updates to immutable fields are forbidden
```

**The only way to change it:** delete it and create a new one (or, more commonly in practice, just create a differently-named new ConfigMap and update your Deployment to point at it — see Reason 2 above).

---

# 9. BEGINNER SUMMARY (for ConfigMaps specifically — full chapter summary comes after Secrets in Part 2)

- A ConfigMap stores non-sensitive configuration as key-value pairs, separate from your container image, so the same image can run in different environments with different settings.
- You can consume a ConfigMap two ways: as environment variables (`env`/`envFrom`) or as files via a volume mount.
- Environment variables are frozen at container startup — changing the ConfigMap never updates a running container's env vars.
- Volume-mounted files DO update automatically (with a short delay), but only if your application code actually re-reads them — most applications don't, by default.
- Marking a ConfigMap `immutable: true` prevents edits entirely, both as a performance optimization (skips the change-watching mechanism) and as a production safety practice (forces config changes through a proper versioned rollout instead of silent in-place edits).

# 10. ADVANCED SUMMARY

- The kubelet fetches ConfigMap content from the API server (never directly from etcd) at container-creation time, and again periodically for volume-mounted ConfigMaps to detect changes.
- `envFrom` silently drops keys that aren't valid environment variable names (like multi-line file-content keys), emitting a warning event rather than failing — a source of confusing "my variable just isn't there" bugs.
- The gap between "ConfigMap key name" and "environment variable name inside the container" is fully controllable with `env`/`valueFrom.configMapKeyRef`, and fully invisible (relying on exact name matching) with `envFrom` — this is architecturally why Incident 1's bug was possible at all: Kubernetes has zero visibility into what environment variable names your application code actually expects.
- `immutable: true` is a real, measurable API-server load reduction in large clusters, not merely a stylistic recommendation — Kubernetes skips establishing a watch for objects it knows can never change.

---

That's ConfigMaps complete, at the depth your rules require. Say **"continue"** for Part 2: Secrets from first principles — purpose, Secret types, `stringData` vs `data`, base64 (and exactly why it is not encryption, explained mechanically), Secret volumes, environment variables, and projected service account tokens.

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
# Kubernetes Autoscaling — HPA & KEDA Complete Mastery Guide
### From Absolute Beginner to Production Event-Driven Scaling

---

# PART 1 — AUTOSCALING FUNDAMENTALS

## What autoscaling means
Automatically adjusting the amount of compute capacity dedicated to a workload, in response to actual demand, without a human manually running `kubectl scale` every time traffic changes.

## Vertical vs horizontal scaling

| | Vertical (scale up/down) | Horizontal (scale out/in) |
|---|---|---|
| What changes | A single Pod's CPU/memory requests/limits | The **number** of Pod replicas |
| Kubernetes mechanism | VerticalPodAutoscaler (VPA) — out of scope here | HorizontalPodAutoscaler (HPA), KEDA |
| Requires Pod restart? | Yes (resizing typically means recreating the Pod) | No — new Pods are simply added/removed alongside existing ones |
| Ceiling | Limited by the biggest single node available | Limited only by cluster-wide capacity across many nodes |

**This guide is entirely about horizontal scaling** — HPA and KEDA never resize an existing Pod; they only ever change the replica count of a Deployment/StatefulSet/etc.

## Why autoscaling exists
Static replica counts force a choice between two bad options: **over-provision** (pay for peak capacity 24/7, even at 3 AM when traffic is 5% of peak) or **under-provision** (fine most of the time, falls over during real spikes). Autoscaling continuously matches capacity to actual demand — the entire economic and reliability case for it in one sentence.

---

# PART 2 — HPA ARCHITECTURE

## The full chain
```
Application Load
      │
      ▼
Metrics (CPU/memory usage on each Pod, or custom application metrics)
      │
      ▼
Metrics Server (aggregates resource metrics cluster-wide, exposes via
                 the Kubernetes Metrics API)
      │
      ▼
HPA Controller (polls the Metrics API periodically, compares current
                 vs target utilization, computes desired replica count)
      │
      ▼
Deployment (.spec.replicas is PATCHED by the HPA controller)
      │
      ▼
ReplicaSet (reconciles toward the new replica count — ReplicaSets guide)
      │
      ▼
Pods (created/deleted to match)
```

**Critical architectural fact:** the HPA controller **never talks to Pods directly** — it only ever reads aggregated metrics and **writes a new `.spec.replicas` value onto the target Deployment** (or StatefulSet, ReplicaSet). Everything below that point is the exact same ordinary reconciliation machinery from the ReplicaSets guide — HPA doesn't create Pods itself; it just changes a number that the Deployment/ReplicaSet controllers then act on, completely unaware that an autoscaler (rather than a human running `kubectl scale`) is the one that changed it.

```
┌───────────────┐   watches Metrics API   ┌───────────────┐
│  HPA Controller │◀───────────────────────│  metrics-server │
└───────┬───────┘                         └───────────────┘
        │ PATCH .spec.replicas
        ▼
┌───────────────┐
│   Deployment    │
└───────┬───────┘
        │ (ordinary reconciliation, per ReplicaSets guide)
        ▼
   ReplicaSet ──▶ Pods
```

## metrics-server
A cluster add-on (not part of core Kubernetes — must be installed separately) that scrapes the kubelet's `/metrics/resource` endpoint on every node, aggregates CPU/memory usage per Pod, and exposes it via the **Metrics API** (`metrics.k8s.io`) — this is precisely what powers both `kubectl top` and the HPA's CPU/memory-based scaling.

```bash
kubectl top pods       # same data source the HPA itself reads
kubectl get apiservices | grep metrics
```

## CPU-based scaling — the canonical example

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app             # the Deployment this HPA controls
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # target: average 70% of REQUESTED cpu across all Pods
```
**Line-by-line:**
- `scaleTargetRef` — which workload this HPA patches `.spec.replicas` on
- `minReplicas`/`maxReplicas` — the hard floor and ceiling; the HPA will never scale outside this range regardless of metric readings
- `metrics[].type: Resource` — built-in CPU/memory scaling (as opposed to `Pods`, `Object`, or `External` types, covered below)
- `averageUtilization: 70` — **percentage of the Pod's own CPU *request*, not an absolute value** — this is the single most misunderstood field in all of HPA: if a Pod requests `500m` CPU, `70%` target means the HPA aims to keep average actual usage around `350m` per Pod, scaling replicas up/down to hold that ratio. **A Pod with no CPU request set cannot be used with utilization-based scaling at all** — there's nothing to compute a percentage of.

## The actual scaling formula
```
desiredReplicas = ceil( currentReplicas × ( currentMetricValue / targetMetricValue ) )
```
### Numerical example
```
currentReplicas = 4
each Pod requests 500m CPU → targetMetricValue = 70% of 500m = 350m
currentMetricValue (average across all 4 Pods) = 490m

desiredReplicas = ceil( 4 × (490 / 350) ) = ceil(4 × 1.4) = ceil(5.6) = 6
```
The HPA controller computes this **every sync period** (default 15s), and patches `.spec.replicas` to `6` if it's outside the tolerance band (default ±10%, to avoid thrashing on tiny fluctuations).

## Memory-based scaling
```yaml
metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```
Mechanically identical to CPU — same percentage-of-request formula. **Memory-based HPA is used far less often than CPU** in practice, because memory usage in many applications doesn't scale down cleanly even under low load (caches, connection pools, JIT-compiled runtimes holding onto allocated heap) — scaling based on memory can produce flapping or simply never trigger a scale-down at all for such workloads.

## Custom metrics and external metrics

```yaml
metrics:
  - type: Pods                      # a metric describing individual Pods (e.g., requests/sec per Pod)
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  - type: External                   # a metric from OUTSIDE the cluster entirely
    external:
      metric:
        name: sqs_queue_length
        selector:
          matchLabels: { queue: "orders" }
      target:
        type: AverageValue
        averageValue: "30"
```
- `type: Pods` — requires a **custom metrics adapter** (e.g., Prometheus Adapter) translating an app-level metric (requests/sec, queue depth per Pod) into the Kubernetes Custom Metrics API
- `type: External` — requires an **external metrics adapter** exposing a metric with no natural Kubernetes object association at all (a cloud queue's depth, a third-party API's rate limit remaining) via the External Metrics API

**This is exactly the gap KEDA was built to fill more simply** — wiring up a custom/external metrics adapter by hand for every different metric source (SQS, Kafka, RabbitMQ, Prometheus...) is real, repetitive infrastructure work; KEDA packages dozens of these adapters as a single, unified system (Part 4).

---

# PART 3 — HPA BEHAVIOR: SCALE-UP, SCALE-DOWN, STABILIZATION

## The problem behavior policies solve
Left unconstrained, an HPA reacting instantly to every metric fluctuation would **thrash** — scaling up and down repeatedly within seconds as load naturally jitters, which is disruptive (new Pods take time to become Ready, per the Pods masterclass) and wasteful.

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0        # react to increases immediately
      policies:
        - type: Percent
          value: 100                        # can DOUBLE replica count per step
          periodSeconds: 60
        - type: Pods
          value: 4                           # OR add up to 4 Pods per step
          periodSeconds: 60
      selectPolicy: Max                       # use whichever policy allows MORE scaling
    scaleDown:
      stabilizationWindowSeconds: 300          # wait 5 min of sustained LOW usage before scaling down
      policies:
        - type: Percent
          value: 10                             # remove at most 10% of replicas per step
          periodSeconds: 60
      selectPolicy: Min
```
**Line-by-line:**
- `stabilizationWindowSeconds` (scaleUp: `0`) — react to load increases as fast as possible; no debate needed — under-provisioning during a real spike is usually worse than a slightly premature scale-up
- `stabilizationWindowSeconds` (scaleDown: `300`) — the asymmetry is deliberate: the HPA looks at the **highest** recommended replica count over the last 5 minutes before scaling down, specifically to avoid prematurely shrinking capacity during a brief lull in an otherwise sustained spike
- `policies` — multiple simultaneous rate limits; `selectPolicy: Max` (scale-up) picks whichever policy permits the *most* aggressive scaling for that step, `selectPolicy: Min` (scale-down) picks the most conservative

## Diagram: asymmetric scaling behavior in action
```
Load:      ▁▁▂▅█████▅▂▃▅███▅▂▁▁▁▁▁▁▁▁▁▁▁▁
Replicas:  2 2 3 6 8 8 8 8 7 5 6 8 8 8 7 6 6 6 6 6 6 5 4 3 2 2 2
                ↑ scales UP fast, no delay
                                              ↑ scales DOWN slowly,
                                                only after 5 min of
                                                sustained lower load,
                                                even though load itself
                                                dropped much earlier
```

---

# PART 4 — KEDA

## Why KEDA exists
HPA's native metrics story covers exactly two built-in things well (CPU, memory) and requires manually standing up a custom/external metrics adapter for anything else. **KEDA (Kubernetes Event-Driven Autoscaling)** packages dozens of ready-made "scalers" for common event sources (message queues, streaming platforms, cloud services, Prometheus, and more) as a single CNCF project, removing the need to build or operate your own metrics adapter for each one.

## KEDA architecture
```
┌──────────────────┐
│   Event Source     │  (RabbitMQ, Kafka, SQS, Prometheus, ...)
└─────────┬─────────┘
          │ KEDA's scaler polls this source directly
          ▼
┌──────────────────┐
│  KEDA Operator      │
│  (watches            │
│   ScaledObjects)      │
└─────────┬─────────┘
          │
    ┌──────┴───────────────────────┐
    ▼                                 ▼
┌───────────────────┐      ┌───────────────────────┐
│ KEDA Metrics Adapter │      │  KEDA itself can ALSO   │
│ (exposes the event     │      │  scale a Deployment        │
│  source's metric via     │      │  DIRECTLY to/from ZERO,       │
│  the standard Kubernetes  │      │  something native HPA          │
│  External Metrics API)      │      │  CANNOT do at all                │
└──────────┬────────────┘      └───────────────────────┘
           │
           ▼
┌───────────────────┐
│   HPA (KEDA creates    │   ← KEDA doesn't REPLACE HPA — for
│   and manages this        │      1-to-N scaling, it creates a
│   for you, automatically)  │      standard HPA object under the
└──────────┬────────────┘   hood and simply feeds it metrics
           │                  from the event source
           ▼
    Deployment → ReplicaSet → Pods
```
**This is the single most important architectural fact about KEDA:** for scaling from 1 replica upward, **KEDA generates and manages an ordinary HPA object for you** — it's not a competing scaling mechanism, it's a **metrics-sourcing layer that feeds the existing HPA machinery**. The one thing KEDA does that HPA architecturally cannot is scale a workload **to and from zero**, which requires an entirely separate mechanism (below), since HPA's `minReplicas` can never go below `1`.

## ScaledObject — the core KEDA resource

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaledobject
spec:
  scaleTargetRef:
    name: order-processor          # the Deployment to scale
  minReplicaCount: 0                 # KEDA CAN go to zero — HPA alone cannot
  maxReplicaCount: 20
  cooldownPeriod: 300                 # seconds to wait at zero-activity before scaling to 0
  pollingInterval: 15                  # how often KEDA checks the event source
  triggers:
    - type: rabbitmq
      metadata:
        queueName: orders
        host: amqp://rabbitmq.default.svc.cluster.local:5672
        queueLength: "20"              # target: 1 replica per 20 queued messages
```
**Line-by-line:**
- `minReplicaCount: 0` — the feature HPA fundamentally cannot offer; when the queue is empty, this workload can scale all the way down to **zero running Pods**, costing nothing
- `pollingInterval: 15` — how often KEDA's scaler checks RabbitMQ's actual queue depth (independent of, and typically more frequent than, the metrics-server-based HPA sync loop)
- `cooldownPeriod: 300` — after activity stops, wait 5 minutes of sustained inactivity before scaling to zero — this is KEDA's equivalent of HPA's scale-down stabilization window, specifically guarding the *last* step down to zero
- `triggers` — this is where KEDA's real value lives: a `type: rabbitmq` scaler that natively understands "queue length," translated automatically into a metric the underlying HPA can consume — no custom Prometheus Adapter or hand-written exporter required

## Scale-to-zero, mechanically
```
Queue empty for cooldownPeriod (300s)
        │
        ▼
KEDA operator scales the Deployment to 0 replicas directly
(bypassing HPA entirely for this specific transition, since
 HPA's own minReplicas floor is 1 and it never manages the 0↔1 edge)
        │
        ▼
   ... time passes, workload is completely idle, zero cost ...
        │
        ▼
New message arrives in the queue
        │
        ▼
KEDA's own polling loop detects activity → scales Deployment
directly from 0 to 1 replica (again, bypassing HPA for this
specific edge) → from 1 replica onward, the KEDA-managed HPA
object takes over normal scaling as load increases further
```

## ScaledJob — the other KEDA resource type
Instead of scaling a long-running Deployment's replica count, a `ScaledJob` creates a **new Kubernetes Job** (per the Jobs guide) for each unit of pending work — appropriate when each message/event should be processed by an entirely separate, run-to-completion Pod rather than a shared pool of long-running workers.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: sqs-job-processor
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: processor
            image: my-sqs-processor
        restartPolicy: Never
  maxReplicaCount: 50            # max CONCURRENT Jobs, not a queue depth setting
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
        queueLength: "1"           # roughly: one Job per queued message
```
**When to choose ScaledJob over a ScaledObject-managed Deployment:** ScaledJob fits naturally when work items are discrete, independent, and benefit from Job-level guarantees (retries via `backoffLimit`, a clean per-item completion signal) — e.g., processing individual uploaded files — whereas a ScaledObject-managed Deployment fits a shared worker-pool model where long-lived processes continuously pull from a queue.

## Trigger examples across event sources

### Kafka
```yaml
triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.default.svc:9092
      consumerGroup: my-consumer-group
      topic: orders
      lagThreshold: "50"        # scale up when consumer lag exceeds 50 messages
```

### AWS SQS
```yaml
triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
      queueLength: "5"
    authenticationRef:
      name: keda-aws-credentials    # references a separate TriggerAuthentication object
```

### Prometheus
```yaml
triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: http_requests_total
      query: sum(rate(http_requests_total{app="my-app"}[2m]))
      threshold: "100"
```
This is KEDA's most flexible trigger — any value obtainable via a PromQL query becomes a scaling signal, effectively subsuming most custom-metrics use cases without needing the Prometheus Adapter's own separate configuration syntax at all.

---

# PART 5 — HPA VS KEDA: AN HONEST COMPARISON

| | HPA (native) | KEDA |
|---|---|---|
| CPU/memory scaling | Built-in, zero extra installation | Delegates to the same mechanism (or its own resource scalers) |
| Scale to/from zero | **Not possible** — `minReplicas` floor is 1 | Native, first-class feature |
| Event-source variety | Requires hand-building a metrics adapter per source | Dozens of pre-built scalers (queues, streams, cloud services, Prometheus) |
| Job-based scaling | Not applicable — HPA only targets replica-count workloads | `ScaledJob` — a genuinely different, complementary model |
| Operational simplicity | Simpler — one built-in Kubernetes object, nothing extra to install | Requires installing and operating the KEDA operator itself |
| Maturity/ubiquity | Core Kubernetes, universally available | CNCF project, extremely widely adopted but still an added dependency |
| Best fit | Steady, resource-bound web/API workloads scaling on CPU/memory | Queue/stream consumers, bursty event-driven workloads, anything needing scale-to-zero |

**Neither is universally better — they solve overlapping but distinct problems, and are frequently used together in the same cluster** (KEDA for queue-driven background workers, plain HPA for the request-serving API tier) rather than as a forced either/or choice.

---

# PART 6 — PRODUCTION SCENARIOS AND TROUBLESHOOTING

## Scenario: HPA shows `<unknown>` for current metrics
```bash
kubectl get hpa
# TARGETS: <unknown>/70%
kubectl describe hpa my-app-hpa
```
Almost always: metrics-server isn't installed, isn't healthy, or the target Pods don't have CPU **requests** set at all (utilization-based scaling has nothing to compute a percentage against without a request, per Part 2).

## Scenario: HPA scales up correctly but application still struggles under load
Check whether new Pods are actually reaching `Ready` fast enough (Pods masterclass' readiness-gating) — HPA scaling replica count up doesn't help if each new Pod takes 90 seconds to warm up and the traffic spike is over in 30. This is a **cold-start problem**, not an HPA misconfiguration — solutions live outside HPA entirely (pre-warming, faster startup, `minReplicas` set higher as a baseline).

## Scenario: KEDA ScaledObject never scales past 1, despite obvious growing queue depth
```bash
kubectl get scaledobject
kubectl describe scaledobject <name>
kubectl get hpa    # KEDA creates one automatically — inspect IT directly too
```
Check the trigger's `metadata` values carefully (typo'd queue name/URL is extremely common) and confirm KEDA's operator Pod itself has network access and correct credentials (`TriggerAuthentication`) to actually reach the event source.

## Scenario: Workload never scales to zero despite an apparently empty queue
Check `cooldownPeriod` — a value that's simply longer than your observation window will look like "it's stuck," when it's actually just still counting down; also confirm the metric genuinely reads zero (a lingering "1 in-flight but not-yet-acked" message in some queue systems can keep reported depth just above zero indefinitely).

## Scenario: Rapid, thrashing scale up/down cycles
Check `behavior.scaleDown.stabilizationWindowSeconds` (HPA) or `cooldownPeriod` (KEDA) — both default to a nonzero value specifically to prevent this, so a thrashing pattern strongly suggests one of these was explicitly set too low, or a `stabilizationWindowSeconds: 0` was applied to scale-down as well as scale-up.

---

# PART 7 — DECISION GUIDE

```
Does your workload need to scale based on CPU/memory only,
and never needs to go below 1 replica?
        │
        ├── YES → plain HPA is sufficient and simplest
        │
        └── NO, continue:
                │
                Does it need to react to an external event source
                (queue depth, stream lag, a custom Prometheus metric)?
                        │
                        ├── YES → KEDA
                        │
                        Does it need to scale to/from ZERO when idle?
                        │
                        ├── YES → KEDA (this alone rules out plain HPA)
                        │
                        Is each unit of work naturally a discrete,
                        independent, run-to-completion task?
                        │
                        ├── YES → KEDA's ScaledJob
                        └── NO (shared long-running worker pool) →
                              KEDA's ScaledObject (which manages an
                              HPA under the hood for you)
```

---

# CHEAT SHEET

```bash
kubectl get hpa
kubectl describe hpa <name>
kubectl top pods                       # the same data source HPA's CPU/memory scaling reads
kubectl autoscale deployment my-app --cpu-percent=70 --min=2 --max=10   # quick imperative HPA creation

kubectl get scaledobjects
kubectl describe scaledobject <name>
kubectl get scaledjobs
```
```
HPA field                              Meaning
──────────────────────                 ───────────────────────────────
minReplicas / maxReplicas               hard floor/ceiling (min ALWAYS ≥ 1)
averageUtilization                       % of REQUESTED resource, not absolute
behavior.scaleUp/scaleDown                rate-limiting + stabilization windows

KEDA field                              Meaning
──────────────────────                 ───────────────────────────────
minReplicaCount: 0                       the one thing HPA alone can't do
pollingInterval                          how often the event source is checked
cooldownPeriod                           delay before the final scale-to-zero step
triggers                                  the event-source-specific scaler config
```

---

# INTERVIEW QUESTIONS

**Fundamentals**
1. What's the difference between vertical and horizontal scaling?
2. What problem does autoscaling solve economically and operationally?

**HPA mechanics**
3. Walk through the full chain from application load to a new Pod being created via HPA.
4. Does the HPA controller ever create Pods directly? Explain what it actually does.
5. What does `averageUtilization: 70` actually mean, and why does it require the Pod to have a CPU request set?
6. Write out and explain the HPA scaling formula.
7. Why is the scale-down stabilization window typically much longer than scale-up's?

**Metrics**
8. What is metrics-server, and what does it have to do with `kubectl top`?
9. What's the difference between the `Resource`, `Pods`, and `External` metric types in an HPA spec?
10. Why can't native HPA easily scale on a queue's depth without extra infrastructure?

**KEDA**
11. What does KEDA fundamentally add that HPA alone cannot provide?
12. Explain KEDA's relationship to HPA — does KEDA replace it or build on it?
13. What's the difference between a ScaledObject and a ScaledJob, and when would you choose each?
14. What does `cooldownPeriod` control, specifically?
15. Why is scale-to-zero architecturally impossible for plain HPA, regardless of configuration?

**Production**
16. An HPA shows `<unknown>` for its current metric — what are your first two hypotheses?
17. A KEDA-scaled workload appears "stuck" at zero despite new messages arriving — what would you check?
18. Why might scaling replica count up fail to actually help during a sudden traffic spike?
19. Describe a realistic setup where both plain HPA and KEDA are used together in the same cluster, for different workloads.
20. Why should HPA and KEDA never be described as "one is strictly better than the other"?

---

*This guide covers Kubernetes autoscaling from vertical-vs-horizontal fundamentals through HPA's metrics/behavior mechanics and KEDA's event-driven, scale-to-zero architecture — the complete arc from beginner to production autoscaling design.*
# Kubernetes Autoscaling — Complete Masterclass (Beginner → Production)

---

## PART 1 — THE BIG PICTURE: HOW ALL THE AUTOSCALERS RELATE

Before diving into each mechanism, understand this: Kubernetes autoscaling isn't one system — it's **three independent controllers operating at different layers**, each solving a different question:

| Layer | Question it answers | Mechanism |
|---|---|---|
| Pod resources | "Is this ONE Pod sized correctly (CPU/mem request)?" | Vertical Pod Autoscaler (VPA) |
| Pod count | "Do I have ENOUGH Pod replicas for current load?" | Horizontal Pod Autoscaler (HPA) / KEDA |
| Node count | "Do I have ENOUGH nodes for all these Pods to be scheduled?" | Cluster Autoscaler (CA) / Karpenter |

They are **decoupled but chained** — a scaling decision at one layer cascades into demand at the next:

```
                     ┌────────────────────────────────────────────┐
                     │              THE FULL CHAIN                 │
                     └────────────────────────────────────────────┘

  Real traffic/load hits the Application
              │
              ▼
  Metrics reported (CPU%, memory, queue depth, custom/external metric)
              │
              ▼
  HPA (or KEDA) evaluates: "current metric vs target → desired replicas"
              │
              ▼
  HPA patches Deployment/StatefulSet's spec.replicas
              │
              ▼
  ReplicaSet controller creates/deletes Pod objects to match
              │
              ▼
  New Pods are Pending, need scheduling
              │
              ▼
  Scheduler tries to place them on existing Nodes
              │
        ┌─────┴─────┐
        ▼             ▼
   Fits on a       Doesn't fit anywhere
   node → Pod      (insufficient CPU/mem
   scheduled        on all nodes)
   normally              │
                          ▼
                  Pods stay Pending
                          │
                          ▼
                  Cluster Autoscaler notices unschedulable Pods
                          │
                          ▼
                  CA provisions a NEW NODE (cloud API call)
                          │
                          ▼
                  New node joins cluster → scheduler places
                  the pending Pods on it
```

This is the mental model to hold onto for the entire chapter: **HPA/KEDA decide "how many Pods." Scheduler decides "where." Cluster Autoscaler decides "do we need more room to put them."** VPA is orthogonal — it's about right-sizing each Pod's resource footprint, independent of replica count.

---

## PART 2 — HORIZONTAL POD AUTOSCALER (HPA)

### 1. What HPA is

The HPA is a Kubernetes control loop that automatically adjusts `spec.replicas` on a Deployment/StatefulSet/ReplicaSet based on observed metrics (CPU, memory, custom, or external), keeping a target metric value roughly constant by adding/removing Pods.

### 7-8. CPU-based and Memory-based scaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp                  # the Deployment being scaled
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # target: average 70% of requested CPU across Pods
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:                       # fine-grained scale-up/down speed control (see below)
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
```

**Line by line:**
- `scaleTargetRef` — the workload HPA controls (never targets Pods directly; always a controller with a `replicas` field)
- `minReplicas`/`maxReplicas` — hard floor/ceiling, regardless of metric pressure — critical safety rails
- `averageUtilization: 70` for CPU means: **percentage of the Pod's CPU `request`**, not absolute cores. This is why every Pod targeted by an HPA **must** have `resources.requests.cpu` set — without it, "70% of what?" is undefined and the HPA can't compute anything (this is failure mode #1, covered later)
- `behavior` — controls how fast HPA reacts. `stabilizationWindowSeconds` on scale-down means: look back over this window and use the **highest** recommended replica count seen, to avoid dropping Pods too eagerly. Scale-up defaults to fast/no stabilization window (react to spikes quickly), scale-down defaults to slower (avoid flapping).

### Internals — how HPA actually computes desired replicas

```
Every ~15s (sync period):
  desiredReplicas = ceil( currentReplicas *  ( currentMetricValue / desiredMetricValue ) )
```

Example: 4 replicas running, average CPU utilization is 140% of request, target is 70%:
```
desiredReplicas = ceil( 4 * (140/70) ) = ceil(4 * 2) = 8
```

HPA reads metrics from the **Metrics API** — for CPU/memory this is served by **metrics-server** (aggregates kubelet's cAdvisor stats); for custom/external metrics, from an **adapter** implementing the `custom.metrics.k8s.io` or `external.metrics.k8s.io` API.

```
Application Pods
      │ (cAdvisor stats via kubelet)
      ▼
metrics-server  ──implements──►  metrics.k8s.io API
      │
      ▼
HPA controller (polls every ~15s) ──► computes desired replicas
      │
      ▼
PATCH Deployment.spec.replicas
```

### 9-10. Custom Metrics and External Metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-custom-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods                     # CUSTOM metric, per-Pod average
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "500"         # scale to keep ~500 req/s per Pod
  - type: External                 # EXTERNAL metric — from a system outside the cluster
    external:
      metric:
        name: queue_messages_ready
        selector:
          matchLabels:
            queue: orders
      target:
        type: AverageValue
        averageValue: "30"           # e.g., 30 messages per replica → scale accordingly
```

- **Custom metrics** (`type: Pods` or `type: Object`) come from **inside the cluster** — usually scraped by Prometheus and exposed via the **Prometheus Adapter**, implementing the `custom.metrics.k8s.io` API
- **External metrics** come from **outside** the cluster entirely — a cloud SQS queue depth, a managed Kafka lag metric, a third-party SaaS metric — via an adapter implementing `external.metrics.k8s.io`

```
Prometheus scrapes app  ──► Prometheus Adapter  ──► custom.metrics.k8s.io API ──► HPA
Cloud queue (SQS/PubSub) ──► Cloud provider adapter ──► external.metrics.k8s.io API ──► HPA
```

---

## PART 3 — KEDA (EVENT-DRIVEN SCALING)

### 3. What KEDA is

**KEDA** (Kubernetes Event-Driven Autoscaling) is a CNCF project that extends the HPA model to scale based on **event sources** — message queue depth, Kafka consumer lag, cron schedules, HTTP request rate, cloud-native queues (SQS, Azure Service Bus, Pub/Sub) — and crucially, supports **scale-to-zero**, which vanilla HPA cannot do (HPA's `minReplicas` minimum is 1).

```
KEDA architecture:

Event Source (Kafka, SQS, RabbitMQ, cron, Prometheus...)
       │
       ▼
KEDA Scaler (polls the event source)
       │
       ▼
KEDA Operator ──creates/manages──► HPA object (KEDA generates this for you)
       │                                  │
       ▼                                  ▼
ScaledObject CRD                   standard HPA scaling logic (0→1 handled by KEDA itself)
```

### 11-12. Event-driven scaling & Scale-to-zero (YAML)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor          # Deployment to scale
  minReplicaCount: 0                # KEDA CAN go to zero — vanilla HPA cannot
  maxReplicaCount: 30
  cooldownPeriod: 300                # seconds of no events before scaling to 0
  pollingInterval: 15                 # how often KEDA checks the event source
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: order-group
      topic: orders
      lagThreshold: "50"              # scale so each replica handles ~50 lag
```

**Internals:** below `minReplicaCount` when it's `0`, KEDA itself (not the standard HPA machinery) watches the event source and, on the **first** event/message, scales the Deployment from 0 → 1 directly. Once at 1+ replicas, KEDA hands control to a normal Kubernetes HPA object it created under the hood, which then scales 1→N using the same reconciliation math as regular HPA. When the queue empties and `cooldownPeriod` elapses with no events, KEDA scales back to 0.

```
Queue empty, 0 replicas
       │  message arrives
       ▼
KEDA scaler detects event → scales Deployment 0 → 1
       │
       ▼
Generated HPA object takes over, scales 1 → N based on lag
       │  queue drains, cooldownPeriod passes
       ▼
KEDA scales back down to 0
```

**Production use case:** background job processors, batch workers, anything with bursty/intermittent load where paying for idle replicas 24/7 is wasteful — classic serverless-on-Kubernetes pattern.

---

## PART 4 — VERTICAL POD AUTOSCALER (VPA)

### 2. What VPA is

VPA automatically adjusts a Pod's `resources.requests` (and optionally `limits`) based on **observed historical usage**, rather than you guessing values manually. It solves a different problem than HPA: "is each individual Pod's resource allocation correctly sized," not "how many Pods do I need."

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"          # or "Off" (recommend only), "Initial" (set at creation only)
  resourcePolicy:
    containerPolicies:
    - containerName: myapp
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
```

**Internals:** VPA has 3 components: a **Recommender** (analyzes historical usage from metrics-server/Prometheus, computes suggested requests), an **Updater** (in `Auto` mode, evicts Pods whose requests are significantly off from the recommendation so they get recreated with new values), and an **Admission Controller webhook** (mutates new Pod specs at creation time to apply the current recommendation).

```
Historical usage data
       │
       ▼
VPA Recommender ──► computes suggested requests/limits
       │
       ▼
"Auto" mode: VPA Updater evicts mis-sized Pods
       │
       ▼
Pod recreated → VPA Admission Webhook intercepts creation,
                 injects the recommended resources.requests
```

⚠️ **Critical incompatibility:** VPA and HPA **must not** both target CPU/memory on the same workload simultaneously — VPA changing `requests` mid-flight while HPA is calculating utilization percentages **against those same requests** creates a feedback loop / undefined behavior. If you need both, use VPA on non-CPU/memory metrics only, or use VPA in `Off`/recommendation-only mode to inform manual tuning, and let HPA own CPU/memory scaling.

---

## PART 5 — CLUSTER AUTOSCALER (NODE-LEVEL)

### 4-5. Cluster Autoscaler and Node Autoscaling

The **Cluster Autoscaler (CA)** watches for Pods that are `Pending` because no existing node has room, and provisions new nodes (via cloud provider APIs — ASG, node pools, etc.) to fit them. It also removes underutilized nodes when their workloads could be consolidated elsewhere, to save cost.

```
CA reconcile loop (simplified):

loop every ~10s:
    pendingPods = get Pods in Pending phase with FailedScheduling events
    for each pendingPod:
        if no existing node CAN fit it (even hypothetically):
            find a node group whose node template WOULD fit it
            scale that node group up by however many nodes are needed
    for each node:
        if node utilization is low AND all its Pods could be
           rescheduled elsewhere without violating constraints:
            cordon + drain + terminate that node (scale down)
```

```yaml
# Example: annotations that influence CA behavior on a node group (cloud-specific, AWS EKS example)
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-status
  namespace: kube-system
# (actual CA config typically lives in its Deployment args, not shown in full here)
```

Typical CA deployment flags (illustrative):
```yaml
        command:
        - ./cluster-autoscaler
        - --nodes=1:10:my-node-group          # min:max:node-group-name
        - --scale-down-delay-after-add=10m     # wait before considering scale-down post scale-up
        - --scale-down-unneeded-time=10m        # node must be underused this long before removal
        - --skip-nodes-with-local-storage=false
        - --balance-similar-node-groups
```

### 13. Node Provisioning

```
Pod Pending (unschedulable — insufficient node capacity)
       │
       ▼
CA identifies which node group's instance type WOULD satisfy
   this Pod's requests/affinity/taints
       │
       ▼
CA calls cloud API: "add N nodes to this ASG/node-pool"
       │
       ▼
Cloud provisions VM(s), bootstraps kubelet, node joins cluster
   (this step takes real wall-clock time — often 1-5 minutes,
    the dominant latency source in the whole autoscaling chain)
       │
       ▼
Scheduler places the previously-Pending Pods on the new node(s)
```

**Karpenter** (increasingly replacing Cluster Autoscaler on AWS/others) works similarly but skips the "node group" abstraction entirely — it provisions **right-sized individual instances** on demand per pending Pod's exact requirements, often faster and more cost-efficient than fixed-shape ASG-based CA.

---

## PART 6 — THE FULL CHAIN, DIAGRAMMED

```
┌──────────────┐
│ Application   │  traffic increases → CPU/queue depth rises
└───────┬──────┘
        │ metrics
        ▼
┌──────────────┐
│  HPA / KEDA   │  computes: need 8 replicas (was 4)
└───────┬──────┘
        │ patches Deployment.replicas = 8
        ▼
┌──────────────┐
│    Pods       │  4 new Pod objects created, phase=Pending
└───────┬──────┘
        │
        ▼
┌──────────────┐
│  Scheduler    │  tries to place each Pod on existing Nodes
└───────┬──────┘
        │
   ┌────┴─────┐
   ▼           ▼
 fits      doesn't fit
 on Node    anywhere
   │           │
   ▼           ▼
Pod runs   stays Pending, FailedScheduling event
              │
              ▼
        ┌──────────────┐
        │Cluster        │  sees pending Pods, provisions new node(s)
        │Autoscaler     │
        └───────┬──────┘
                │ cloud API call, new VM boots (~1-5 min)
                ▼
        ┌──────────────┐
        │    Nodes      │  new node joins, Ready
        └───────┬──────┘
                │
                ▼
        Scheduler places the previously-Pending Pods
```

**Key latency insight:** HPA reacts in ~15-30 seconds. Cluster Autoscaler reacts in **minutes** (cloud VM boot time dominates). This asymmetry is the #1 source of "why is my app still slow even though HPA already scaled up" incidents — Pods exist but are stuck Pending waiting for nodes.

---

## PART 7 — INTERACTIONS BETWEEN AUTOSCALERS

| Combo | Interaction |
|---|---|
| HPA + Cluster Autoscaler | Normal, expected chain — HPA drives Pod count, CA drives node count to fit them. No conflict. |
| HPA + VPA (same resource, e.g. CPU) | ⚠️ Conflict — both changing things that affect the same utilization calculation. Avoid, or split VPA to non-overlapping metrics/`Off` mode. |
| KEDA + Cluster Autoscaler | Same relationship as HPA + CA — KEDA's generated HPA drives replica count, CA reacts to resulting Pending Pods. Scale-to-zero means CA may also scale nodes down to zero in that node group if supported. |
| VPA + Cluster Autoscaler | Indirect — VPA changing a Pod's `requests` upward can itself cause the Pod to no longer fit its current node, triggering CA to add capacity on next scheduling attempt. |
| Multiple HPAs on the same workload | Not supported — one HPA per `scaleTargetRef`. Multiple metrics within **one** HPA object are fine (HPA takes the **max** computed replica count across all configured metrics). |

---

## PART 8 — PRODUCTION SCENARIOS & COMMON MISTAKES

### Insufficient resource requests

If `resources.requests.cpu` isn't set, `averageUtilization` targets are **mathematically undefined** — HPA can't compute a percentage of nothing. `kubectl describe hpa` will show `<unknown>` for that metric, and the HPA effectively stalls on that signal.

```bash
kubectl describe hpa myapp-hpa
# Metrics: ( current / target )
#   resource cpu on pods: <unknown> / 70%
```

**Fix:** always set `resources.requests` on any workload targeted by CPU/memory-based HPA.

### Metrics unavailable

```bash
kubectl get apiservices | grep metrics
# v1beta1.metrics.k8s.io   ... False (MissingEndpoints)
```

Common causes: `metrics-server` not installed/crashed, custom metrics adapter (Prometheus Adapter) misconfigured or its target Prometheus query returning nothing, network policy blocking metrics-server from reaching kubelets.

**Diagnose:**
```bash
kubectl top pods              # fails if metrics-server is down
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl logs -n kube-system deploy/metrics-server
```

### Scaling too aggressively

Default HPA scale-up has almost no dampening — a brief traffic spike can trigger a large replica jump, then just as quickly scale back down, causing **cost churn** and **scheduling thrash** on the node/CA layer.

**Fix — use `behavior` to bound scale-up rate:**
```yaml
behavior:
  scaleUp:
    policies:
    - type: Pods
      value: 4                # add at most 4 Pods per period
      periodSeconds: 60
    stabilizationWindowSeconds: 60
```

### Scaling oscillation ("flapping")

Symptom: replicas bounce 4→8→4→8 repeatedly. Usually caused by:
- Target utilization set too close to actual steady-state load (no headroom/hysteresis)
- No `stabilizationWindowSeconds` on scale-down, so HPA reacts instantly to every metric dip
- Multiple metrics conflicting (CPU says scale down, custom metric says scale up, decision flips depending on which "wins" that cycle — remember HPA takes the **max** across all metrics)

**Fix:** widen the target utilization band, add a scale-down stabilization window (default is already 300s in `autoscaling/v2`, but verify it's not been reduced), and if using multiple metrics, ensure they don't structurally disagree.

### Cold starts

When scaling from 0 (KEDA) or adding fresh replicas under a fast spike, new Pods must: pull image (if not cached) → start container → pass readiness probe → **then** actually start receiving traffic. If your image is large or your app has a slow boot sequence, this "cold start" latency can mean real users hit errors/timeouts before capacity actually arrives.

**Mitigations:**
- Keep images small, use pre-pulled/cached base layers
- Use `startupProbe` correctly so readiness isn't gated behind an artificially slow default
- For latency-critical scale-from-zero cases, consider keeping `minReplicaCount: 1` instead of `0` (small idle cost, eliminates cold start on first request)
- Pre-warm via scheduled minimum replicas ahead of known traffic patterns (e.g., cron-based KEDA trigger before a known daily peak)

### Insufficient nodes / stuck Pending

If Cluster Autoscaler can't add capacity fast enough (cloud quota limits, node group max size reached, or CA not installed/misconfigured at all), HPA-created Pods simply queue up `Pending` indefinitely.

```bash
kubectl get pods --field-selector=status.phase=Pending
kubectl describe pod <pending-pod>
# Events: "0/5 nodes are available: 5 Insufficient cpu"
kubectl logs -n kube-system deploy/cluster-autoscaler | tail -50
# look for "max node group size reached" or cloud quota errors
```

**Fix:** raise CA's `--nodes=min:max` ceiling, request higher cloud quota, or diversify node groups/instance types (Karpenter handles this more gracefully than classic CA).

---

## PART 9 — YAML: FULL PRODUCTION EXAMPLE

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
      - name: api
        image: myregistry/api:2.1.0
        resources:
          requests:
            cpu: 250m            # REQUIRED for HPA CPU-based scaling to function
            memory: 256Mi
          limits:
            memory: 512Mi
        readinessProbe:
          httpGet: { path: /readyz, port: 8080 }
          periodSeconds: 5
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 30
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 65 }
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - { type: Pods, value: 5, periodSeconds: 60 }
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - { type: Percent, value: 25, periodSeconds: 60 }
```

---

## PART 10 — HANDS-ON MINIKUBE LABS

```bash
minikube start --cpus=4 --memory=6144
minikube addons enable metrics-server
```

### Lab 1 — Basic CPU-based HPA
```bash
kubectl create deployment cpu-demo --image=k8s.gcr.io/hpa-example --requests=cpu=200m
kubectl expose deployment cpu-demo --port=80
kubectl autoscale deployment cpu-demo --cpu-percent=50 --min=1 --max=6
kubectl get hpa cpu-demo -w
```

Generate load:
```bash
kubectl run -it --rm load-generator --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://cpu-demo; done"
```
Watch replicas climb, then stop the load generator and watch the (slower) scale-down.

### Lab 2 — Missing resource requests (failure demo)
```bash
kubectl create deployment no-requests --image=nginx
kubectl autoscale deployment no-requests --cpu-percent=50 --min=1 --max=5
kubectl describe hpa no-requests
# Metrics: <unknown> / 50% — demonstrates the #1 misconfiguration
```

### Lab 3 — HPA `behavior` tuning
```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: tuned-hpa
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: cpu-demo }
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 50 } }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
      policies: [{ type: Pods, value: 1, periodSeconds: 60 }]
EOF
kubectl get hpa tuned-hpa -w
```

### Lab 4 — Cluster Autoscaler simulation (conceptual, minikube multi-node)
```bash
minikube start --nodes=2 --cpus=2 --memory=2048
kubectl create deployment resource-hog --image=nginx --replicas=1
kubectl set resources deployment resource-hog --requests=cpu=1500m
kubectl scale deployment resource-hog --replicas=10
kubectl get pods -o wide
kubectl describe pod <pending-pod>   # observe "Insufficient cpu" event
# (real Cluster Autoscaler requires a cloud provider; minikube demonstrates
#  the PENDING state that would trigger it in a real cloud cluster)
```

### Lab 5 — VPA recommendation-only mode (requires VPA installed)
```bash
# after installing VPA components:
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: cpu-demo-vpa
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: cpu-demo }
  updatePolicy: { updateMode: "Off" }
EOF
kubectl describe vpa cpu-demo-vpa   # shows Recommendation section after some data collection
```

---

## PART 11 — TROUBLESHOOTING CHEAT SHEET

```bash
kubectl get hpa                              # current/target/min/max/replicas at a glance
kubectl describe hpa <name>                   # Events section shows scaling decisions & failures
kubectl top pods / kubectl top nodes           # requires metrics-server
kubectl get apiservices | grep metrics         # is the metrics API available?
kubectl get pods --field-selector=status.phase=Pending
kubectl describe pod <pending-pod>             # scheduling failure reasons
kubectl logs -n kube-system deploy/cluster-autoscaler
kubectl get events --sort-by=.metadata.creationTimestamp | grep -i scal
```

---

## PART 12 — INTERVIEW QUESTIONS

1. What are the three main Kubernetes autoscalers, and what does each control?
2. Why must a Pod have `resources.requests.cpu` set for CPU-based HPA to function?
3. Walk through the full chain from a traffic spike to a new node joining the cluster.
4. What's the formula HPA uses to compute desired replicas?
5. What's the difference between `type: Resource`, `type: Pods`, and `type: External` metrics in HPA?
6. What does the `behavior` field control, and why does scale-down typically use a longer stabilization window than scale-up?
7. Why can't HPA scale to zero, and how does KEDA solve that?
8. Explain KEDA's architecture: ScaledObject, Scaler, and the HPA it generates.
9. What's the fundamental risk of running VPA and HPA on the same resource metric simultaneously?
10. What are VPA's three components, and what does each do?
11. What's the difference between VPA's `Auto`, `Off`, and `Initial` update modes?
12. How does Cluster Autoscaler decide when to add a node?
13. How does Cluster Autoscaler decide when to remove a node?
14. Why is Cluster Autoscaler's reaction time typically much slower than HPA's?
15. What's the practical difference between Cluster Autoscaler and Karpenter?
16. What happens to Pods when HPA scales up but no node has capacity?
17. What metric source powers basic CPU/memory HPA, and what component provides it?
18. Where do custom metrics come from, architecturally?
19. Where do external metrics come from, and give an example?
20. What does it mean when `kubectl describe hpa` shows `<unknown>` for a metric?
21. What causes HPA "flapping," and how do you fix it?
22. What is a cold start in the context of scaling, and how would you mitigate it for a scale-to-zero workload?
23. Why might raising `maxReplicas` alone fail to fix a capacity problem?
24. What cloud-level constraint commonly blocks Cluster Autoscaler from adding nodes even when configured correctly?
25. How does KEDA behave differently from vanilla HPA once replica count is above zero?
26. Why does HPA take the maximum computed replica count when multiple metrics are configured?
27. What's the role of the scheduler in the overall autoscaling chain?
28. Design an autoscaling strategy for a Kafka-consuming background worker with highly bursty, intermittent load.
29. Design an autoscaling strategy for a latency-critical user-facing API with a strict SLA, considering cold starts.
30. Explain a production incident where HPA scaled correctly but users still experienced degraded performance — what layer was likely the bottleneck?

---

## PRODUCTION BEST PRACTICES SUMMARY

- Always set accurate `resources.requests` — it's the foundation every autoscaling layer depends on
- Never combine HPA and VPA on the same CPU/memory metric for the same workload
- Tune `behavior.scaleDown.stabilizationWindowSeconds` deliberately — default of 300s is usually reasonable, don't reduce it without understanding the flapping risk
- Use KEDA (not vanilla HPA) for event-driven/queue-based workloads, especially where scale-to-zero saves meaningful cost
- Monitor the **Pending Pod** count as a first-class SLO signal — it's your earliest warning that HPA has "won" but Cluster Autoscaler hasn't "caught up" yet
- Set realistic `maxReplicas`/node-group max sizes with cloud quota in mind — an HPA ceiling that exceeds what CA can actually provision is a silent trap
- For latency-critical services, keep `minReplicas` (or KEDA's `minReplicaCount`) above zero to avoid cold-start-induced SLA violations
- Prefer Karpenter over classic Cluster Autoscaler where available — faster, more flexible node provisioning reduces the weakest link in the whole chain

---

That completes the Autoscaling chapter. Ready to continue to the next batch of the masterclass — happy to proceed wherever the approved 8-batch plan sequences next (Services & Networking, ConfigMaps/Secrets, Ingress, or RBAC/Security).

# Kubernetes Storage — Complete Mastery Guide
### From Absolute Beginner to Production Storage Architecture

---

# PART 1 — FUNDAMENTALS

## Why containers are ephemeral

A container's writable layer lives only as long as the container itself — when a container is destroyed (crash, redeploy, node failure), everything written to its local filesystem is gone. This isn't a bug; it's the entire point of the container model — images are immutable, and any given container instance is meant to be disposable and replaceable (Pods masterclass, Diagram D2's "brand-new Pod, not the same one" principle extends down to the filesystem too).

## Why persistent storage is needed

Some data must **outlive** any individual Pod: a database's actual rows, uploaded user files, application state that took real time/cost to compute. If that data only exists inside a container's writable layer, replacing the Pod — which happens constantly and is supposed to be safe, per the entire self-healing architecture covered in the Architecture masterclass — would silently destroy it. Kubernetes' storage system exists specifically to decouple **data lifetime from Pod lifetime**.

## Storage fundamentals — the core problem shape

```
Pod lifetime:     [created]──[running]──[deleted]
                                              │
Data lifetime needed:                          │
  (must survive past this point) ──────────────┴──────────▶ (continues)
```
Everything in this guide is infrastructure built to make that gap survivable.

---

# PART 2 — VOLUMES: THE BASE CONCEPT

## What a Volume actually is
A Volume is **storage attached to a Pod**, with a lifetime tied to *some* scope — sometimes the Pod's own lifetime (ephemeral), sometimes far beyond it (persistent). The word "Volume" in Kubernetes covers both extremes; the specific **type** of volume determines which.

## emptyDir — Pod-lifetime-scoped
```yaml
apiVersion: v1
kind: Pod
metadata: { name: scratch-space-demo }
spec:
  containers:
    - name: app
      image: myapp
      volumeMounts:
        - name: cache
          mountPath: /cache
  volumes:
    - name: cache
      emptyDir:
        sizeLimit: 1Gi          # optional cap on how large this can grow
```
- `emptyDir: {}` — created empty when the Pod is assigned to a node, exists only as long as **that specific Pod object** exists (already covered in the Pods masterclass §16 as the shared-volume mechanism between containers) — deleted permanently the instant the Pod is removed, even if a "replacement" Pod immediately takes its place
- `sizeLimit` — without this, an `emptyDir` can grow unbounded and contribute to the ephemeral-storage/DiskPressure scenario from the Resource Management guide

## hostPath — node-lifetime-scoped (use with extreme caution)
```yaml
apiVersion: v1
kind: Pod
metadata: { name: hostpath-demo }
spec:
  containers:
    - name: app
      image: myapp
      volumeMounts:
        - name: node-logs
          mountPath: /var/log/host
  volumes:
    - name: node-logs
      hostPath:
        path: /var/log
        type: Directory
```
- `hostPath` — mounts an actual directory **from the underlying node's own filesystem** directly into the Pod — data survives Pod deletion, but is tied to that **specific node**, not to the Pod or the cluster as a whole
- `type: Directory` — validates the path must already exist as a directory (other values: `DirectoryOrCreate`, `File`, `FileOrCreate`, `Socket`, etc.) — an explicit safety check against typos creating unexpected paths

**Why "extreme caution":** if the Pod is rescheduled to a *different* node (which happens routinely — node failure, draining, normal rescheduling), it loses access to that data entirely, since the new node has its own, unrelated `/var/log`. `hostPath` also has serious security implications (a container writing to `/var/log` or, worse, `/etc` on the actual node) — it's appropriate almost exclusively for node-level system agents (log collectors, monitoring daemons) that are *specifically supposed to* read that node's own files, never for general application data.

---

# PART 3 — THE PERSISTENT STORAGE ABSTRACTION LAYER

## The full chain
```
Pod
 │  references a PVC by name
 ▼
PersistentVolumeClaim (PVC)     ← "I need 10Gi of storage, ReadWriteOnce"
 │  bound to a matching (or dynamically provisioned) PV
 ▼
PersistentVolume (PV)             ← "Here is 10Gi of actual storage, ReadWriteOnce"
 │  provisioned according to a
 ▼
StorageClass                       ← "Here's HOW to make one of these, and with what driver"
 │  delegates actual creation to
 ▼
CSI Driver                          ← the plugin that talks to the real storage backend
 │
 ▼
Actual Storage                       (an AWS EBS volume, a GCP PD, a Ceph RBD image, an NFS export...)
```

**Why this many layers exist, architecturally:** this is the same separation-of-concerns idea from the rest of Kubernetes — a Pod author (developer) shouldn't need to know *how* storage is actually provisioned (cloud API calls, credentials, backend-specific parameters); they just declare a **need** (a PVC). A cluster operator configures **how** that need gets fulfilled (a StorageClass, pointing at a CSI driver) completely separately. This mirrors the exact same "declare desired state, let a controller figure out how" pattern from the Architecture masterclass, applied to storage specifically.

## PersistentVolume (PV)

A cluster-scoped object representing **one concrete piece of actual storage**, provisioned and ready to be claimed.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-manual-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  nfs:
    server: 192.168.1.100
    path: "/exports/data"
```
- `capacity.storage` — how much storage this PV represents
- `accessModes` — what kind of concurrent access is possible (§4 below)
- `persistentVolumeReclaimPolicy` — what happens to the underlying storage when its claim is released (§13)
- `storageClassName: manual` — ties this PV to a specific class name for matching purposes (an empty/no class means it can only bind to PVCs that also request no class)
- `nfs.server`/`nfs.path` — the actual backend connection details, specific to this storage type (every backend — AWS EBS, GCP PD, NFS, etc. — has its own equivalent block here, or increasingly, this is left to a CSI driver instead of a hardcoded in-tree type)

## PersistentVolumeClaim (PVC)

A **namespace-scoped** request for storage, written by an application author, with no knowledge of *which* PV will fulfill it.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: manual
```
- `accessModes`/`resources.requests.storage`/`storageClassName` — must be **compatible** with (not necessarily identical to) an available PV for binding to succeed
- Once created, the PVC either binds to a pre-existing matching PV (static provisioning) or triggers a StorageClass's provisioner to create a brand-new one on demand (dynamic provisioning, §5)

## Binding

The **PersistentVolume controller** (yet another ordinary controller in `kube-controller-manager`, following the exact same watch/reconcile pattern as everything else in the Architecture masterclass) watches for unbound PVCs and unbound PVs, and matches them based on: sufficient capacity, compatible access modes, matching `storageClassName`, and (optionally) label selectors — once matched, both objects are updated to reference each other, and the binding is **exclusive**: one PVC binds to exactly one PV, and vice versa, for as long as that binding exists.

```bash
kubectl get pv
kubectl get pvc
# both show a STATUS column: Bound, Available, Pending, Released, Failed
```

---

# PART 4 — STORAGECLASS AND PROVISIONING

## StorageClass

Defines **how** to dynamically provision new PVs on demand, and with what parameters — this is the piece that lets a PVC request storage without any PV having been pre-created at all.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com          # WHICH CSI driver handles this class
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```
- `provisioner` — the CSI driver (§6) responsible for actually creating storage for this class — different clouds/backends have entirely different provisioner names
- `parameters` — backend-specific tuning (disk type, IOPS, filesystem type) passed straight through to the CSI driver
- `reclaimPolicy` — default policy applied to PVs dynamically created from this class (§13)
- `volumeBindingMode: WaitForFirstConsumer` — **delays** actual provisioning until a Pod using this PVC is actually scheduled — critical for correctness in multi-zone clusters (provisioning immediately, before scheduling, risks creating storage in a zone the Pod later gets scheduled *away* from, which then can't attach at all)
- `allowVolumeExpansion: true` — permits a bound PVC's size to be increased later (§17) without needing to recreate anything

## Dynamic provisioning
```
PVC created, referencing StorageClass "fast-ssd"
        │
        ▼
No matching pre-existing PV found
        │
        ▼
StorageClass's provisioner (the CSI driver) is invoked
        │
        ▼
CSI driver calls the actual cloud/storage backend's API
        │
        ▼
A brand-new PV object is created automatically, representing
the newly-provisioned real storage, and immediately bound to the PVC
```
This is, by far, the dominant pattern in modern Kubernetes — most clusters have a **default StorageClass** so a PVC with no `storageClassName` specified at all still gets dynamically provisioned automatically.

```bash
kubectl get storageclass
# NAME                 PROVISIONER          DEFAULT
# fast-ssd (default)    ebs.csi.aws.com       yes
```

## Static provisioning
The manual, older-style pattern: a cluster operator hand-creates PV objects ahead of time (§3's PV example), and PVCs bind to whichever pre-existing one matches — appropriate when storage already exists outside Kubernetes' control (a pre-existing NFS export, a SAN volume manually carved out) and needs to simply be *represented* to Kubernetes rather than *created* by it.

## CSI (Container Storage Interface)

A standardized plugin interface (analogous to CRI for container runtimes, per the Architecture masterclass) that lets any storage vendor write a driver Kubernetes can use, **without that vendor's code needing to live inside Kubernetes core**. Before CSI, storage backend support was hardcoded into Kubernetes itself ("in-tree" volume plugins) — every new backend required a core Kubernetes code change and release. CSI moved this entirely out-of-tree: AWS, GCP, Azure, Ceph, Portworx, and dozens of others each ship their own independently-versioned CSI driver, installed into any cluster that needs it, following one shared, stable interface.

```bash
kubectl get csidrivers
kubectl get pods -n kube-system -l app=ebs-csi-controller    # example: AWS's CSI driver Pods
```

---

# PART 5 — ACCESS MODES AND VOLUME MODES

## 8–11. Access modes

| Mode | Meaning |
|---|---|
| **ReadWriteOnce (RWO)** | Mountable read-write by a **single node** at a time (as of recent Kubernetes versions, multiple Pods on that *same* node can share it — this was historically single-Pod, now it's single-**node**) |
| **ReadOnlyMany (ROX)** | Mountable read-only by **many nodes** simultaneously |
| **ReadWriteMany (RWX)** | Mountable read-write by **many nodes** simultaneously |
| **ReadWriteOncePod** (newer) | Mountable read-write by exactly **one Pod**, cluster-wide — the strictest option, for workloads that must never have two writers under any circumstance |

**Practical consequence:** most cloud block storage (AWS EBS, GCP Persistent Disk, Azure Disk) **only supports RWO** — this is a fundamental backend limitation, not a Kubernetes restriction; you cannot attach a single EBS volume read-write to Pods on two different nodes simultaneously, full stop. Genuine `ReadWriteMany` requires a backend actually built for concurrent multi-node access — NFS, CephFS, Azure Files, and similar network-filesystem-style backends.

```
┌──────────┐         ┌──────────┐
│  Node A    │         │  Node B    │
│  ┌──────┐  │         │  ┌──────┐  │
│  │ Pod 1  │  │         │  │ Pod 2  │  │
│  └───┬──┘  │         │  └───┬──┘  │
└──────┼───┘         └──────┼───┘
       │                       │
       └─────────┬───────────┘
                   ▼
         RWX-capable storage (NFS, CephFS, etc.)
         — BOTH nodes can mount read-write simultaneously

         (an RWO EBS volume could NEVER do this — it can only
          ever be attached to ONE of these two nodes at a time)
```

## 12. Volume modes

```yaml
spec:
  volumeMode: Filesystem     # default — the volume is formatted with a filesystem, mounted as a directory
  # OR
  volumeMode: Block           # the volume is presented as a raw block device, no filesystem at all
```
`Block` mode is a specialized case — used when an application (often a database engine) wants to manage its own on-disk format directly rather than going through a general-purpose filesystem layer, trading Kubernetes-level simplicity for raw performance/control.

---

# PART 6 — RECLAIM POLICIES AND LIFECYCLE

## 13. Reclaim policies

What happens to the **underlying actual storage** when its PVC is deleted:

| Policy | Behavior |
|---|---|
| `Delete` (common default for dynamically provisioned PVs) | The PV **and** the actual backend storage (the real EBS volume, etc.) are deleted automatically |
| `Retain` | The PV object and underlying storage survive — the PV moves to `Released` status, but must be manually cleaned up or reclaimed by an administrator before it can be reused |

**Production implication:** `Delete` is convenient but genuinely destructive — deleting a PVC on a `Delete`-policy PV permanently destroys the actual data with no recovery path (short of a separate backup, §19). Many production setups deliberately override the default to `Retain` for anything holding genuinely critical, hard-to-regenerate data, accepting the extra manual cleanup step as the cost of that safety margin.

## 14–15. Storage lifecycle, the complete picture

```
1. PVC created (references a StorageClass, or matches an existing PV)
2. PV bound (dynamically provisioned, or matched to a pre-existing one)
3. Pod references the PVC by name; scheduler places the Pod (subject to
   the PV's own access-mode/zone constraints — a Pod needing an RWO
   volume already attached elsewhere CANNOT be scheduled onto a
   different node until that volume is detached)
4. kubelet, via CSI, ATTACHES the volume to the node, then MOUNTS it
   into the Pod's filesystem namespace
5. Pod runs, reads/writes data through the mount
6. Pod deleted — volume is UNMOUNTED and DETACHED (but the PVC/PV
   binding itself SURVIVES a Pod deletion — this is precisely the
   point: the data outlives the Pod)
7. A NEW Pod referencing the SAME PVC can be scheduled and will see
   the SAME data — this is the entire value proposition of the whole
   chain, made concrete
8. PVC deleted (a separate, explicit action) → PV's reclaimPolicy
   determines whether underlying storage is destroyed or retained
```

---

# PART 7 — STATEFULSET STORAGE

## Why StatefulSets need their own storage pattern
A Deployment's Pods are fully interchangeable — any Pod can be replaced by any other. A **StatefulSet** (databases, distributed systems with per-member identity) needs each replica to keep **its own, stable, dedicated storage** across restarts and rescheduling — Pod "database-0" must always come back to the same data it had before, never accidentally get "database-1"'s data.

## volumeClaimTemplates
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels: { app: postgres }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 20Gi
```
**Line-by-line:**
- `volumeClaimTemplates` — unlike a Deployment (which has no equivalent field at all), the StatefulSet controller automatically creates **one distinct PVC per replica**, from this template — `data-postgres-0`, `data-postgres-1`, `data-postgres-2` — each bound to its own separate PV
- Each Pod (`postgres-0`, `postgres-1`, `postgres-2`) always mounts **its own specific PVC**, by a stable, predictable naming convention (`<template-name>-<statefulset-name>-<ordinal>`) — if `postgres-1` is deleted and recreated (crash, rescheduling), the replacement Pod is guaranteed to bind to the **same** PVC (`data-postgres-1`), and therefore the same underlying data, every time

```
StatefulSet "postgres" (3 replicas)
        │
   ┌────┼────┬────────────┐
   ▼         ▼             ▼
postgres-0  postgres-1   postgres-2
   │          │             │
   ▼          ▼             ▼
data-        data-         data-
postgres-0   postgres-1    postgres-2
   │          │             │
   ▼          ▼             ▼
 PV A        PV B          PV C
(always stays paired with its OWN ordinal, even across Pod
 deletion/recreation — this is the entire point)
```
**Crucial cleanup note:** deleting a StatefulSet does **not**, by default, delete its `volumeClaimTemplates`-created PVCs — this is deliberate, since accidentally deleting a StatefulSet should never silently destroy a database's actual data; the PVCs (and their bound PVs, subject to reclaim policy) must be cleaned up as a separate, explicit step.

---

# PART 8 — VOLUME EXPANSION, SNAPSHOTS, AND BACKUP

## 17. Volume expansion
```bash
kubectl patch pvc my-app-data -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```
Requires `allowVolumeExpansion: true` on the originating StorageClass (§4) — most CSI drivers support **online** expansion (no Pod restart required), though some backends/filesystems still require a Pod restart (or even a manual filesystem resize step inside the container) to actually make the new space usable at the filesystem level, not just at the block-device level. **Shrinking a volume is generally not supported** at all — expansion is one-directional by design, since safely shrinking live data is a fundamentally harder problem most backends simply don't attempt.

## 18. Snapshots
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot
spec:
  volumeSnapshotClassName: csi-snapshot-class
  source:
    persistentVolumeClaimName: data-postgres-0
```
A **VolumeSnapshot** is a point-in-time copy of a PVC's data, taken via the CSI driver's own snapshot capability (if the backend supports it) — usable later to provision a **new** PVC pre-populated with that snapshot's data (`dataSource` field on a new PVC referencing the snapshot), which is the standard Kubernetes-native mechanism for both backups and cloning a dataset for a staging/test environment.

## 19. Backup considerations
Snapshots alone are **not** a complete backup strategy — a snapshot typically lives on the *same* underlying storage backend/region as the original, meaning a large-scale backend outage or account-level incident can take out both the original and its snapshots together. Production backup strategy generally layers: (1) CSI snapshots for fast, frequent, low-overhead recovery points, **plus** (2) application-level backups (e.g., `pg_dump`, exported to genuinely separate storage/region — this is exactly the Database Backup CronJob pattern from the CronJobs guide) for true disaster-recovery-grade durability, independent of the primary storage system entirely.

---

# PART 9 — TROUBLESHOOTING

## Pending PVC
```bash
kubectl get pvc
kubectl describe pvc <name>
```
Common causes: no StorageClass exists (and none is marked default), the requested access mode isn't supported by any available provisioner, or (for static provisioning) simply no matching pre-created PV exists yet. `describe`'s Events section states the exact reason.

## FailedMount
```bash
kubectl describe pod <pod>
```
The volume is attached at the node level but the actual mount into the Pod's filesystem failed — common causes: wrong filesystem type expectations, a corrupted filesystem on the volume, or (for network storage like NFS) the backend server being unreachable from that specific node.

## FailedAttachVolume
Occurs a layer earlier than FailedMount — the volume couldn't even be **attached** to the node at all. Extremely common cause for RWO cloud block storage: the volume is **still attached to a different node** from a previous Pod placement that hasn't fully detached yet (a stuck/crashed node, or a very fast reschedule racing the detach operation) — this is a frequent, specific pain point for StatefulSets rescheduled quickly after a node failure.
```bash
kubectl describe pod <pod> | grep -A5 FailedAttachVolume
# often names the conflicting node/volume directly
```

## Storage provisioning failure
```bash
kubectl describe pvc <name>
kubectl logs -n kube-system -l app=<csi-driver-controller-label>
```
Check the CSI driver's own controller Pod logs directly — provisioning failures are frequently backend-side (cloud API quota exhausted, insufficient IAM/cloud permissions for the CSI driver's own service account, requested zone has no capacity) rather than a Kubernetes-object-level misconfiguration.

## Permission problems
```yaml
spec:
  securityContext:
    fsGroup: 2000        # ensures the mounted volume's group ownership matches the container's user
```
A very common, specific failure mode: a container running as a non-root user (per the Pods masterclass' `securityContext`) can't actually write to a freshly-mounted volume because it's owned by `root:root` by default — `fsGroup` tells the kubelet to `chown`/set the group ownership of the volume's contents to match, resolving the large majority of "permission denied" errors on otherwise-correctly-mounted volumes.

---

# PART 10 — MINIKUBE LABS

### Lab 1: emptyDir shared between containers
```bash
minikube start
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: lab1 }
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh","-c","echo hello > /data/msg.txt; sleep 3600"]
      volumeMounts: [{ name: shared, mountPath: /data }]
    - name: reader
      image: busybox
      command: ["sh","-c","sleep 5; cat /data/msg.txt; sleep 3600"]
      volumeMounts: [{ name: shared, mountPath: /data }]
  volumes: [{ name: shared, emptyDir: {} }]
EOF
kubectl logs lab1 -c reader
# → prints "hello", proving the shared mount
```

### Lab 2: Dynamic provisioning end-to-end
```bash
kubectl get storageclass       # minikube ships a "standard" default class
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: lab2-pvc }
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: "1Gi" } }
EOF
kubectl get pvc lab2-pvc -w
kubectl get pv
# watch a PV get created and bound AUTOMATICALLY, with no manual PV YAML at all
```

### Lab 3: Prove data survives Pod deletion
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: lab3 }
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","echo persisted-data > /data/file.txt; sleep 3600"]
      volumeMounts: [{ name: storage, mountPath: /data }]
  volumes:
    - name: storage
      persistentVolumeClaim: { claimName: lab2-pvc }
EOF
kubectl delete pod lab3 --wait
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: lab3b }
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","cat /data/file.txt; sleep 3600"]
      volumeMounts: [{ name: storage, mountPath: /data }]
  volumes:
    - name: storage
      persistentVolumeClaim: { claimName: lab2-pvc }
EOF
kubectl logs lab3b
# → "persisted-data" — a DIFFERENT Pod, SAME data, via the SAME PVC
```

### Lab 4: StatefulSet per-replica storage
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: lab4 }
spec:
  serviceName: lab4
  replicas: 3
  selector: { matchLabels: { app: lab4 } }
  template:
    metadata: { labels: { app: lab4 } }
    spec:
      containers:
        - name: app
          image: busybox
          command: ["sh","-c","echo $HOSTNAME > /data/id.txt; sleep 3600"]
          volumeMounts: [{ name: data, mountPath: /data }]
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: "1Gi" } }
EOF
kubectl get pvc -l app=lab4
# → three SEPARATE PVCs: data-lab4-0, data-lab4-1, data-lab4-2
kubectl delete pod lab4-1
kubectl exec lab4-1 -- cat /data/id.txt
# → still shows "lab4-1" — the replacement Pod got its OWN original PVC back
```

---

# STORAGE CHEAT SHEET

```bash
kubectl get pv
kubectl get pvc
kubectl get storageclass
kubectl describe pvc <name>
kubectl describe pv <name>
kubectl get csidrivers
kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"<new-size>"}}}}'  # expand
```
```
Object            Scope        Created by
────────────      ─────────    ─────────────────────────────
PersistentVolume   Cluster       admin (static) OR provisioner (dynamic)
PVC                Namespace     application author
StorageClass       Cluster       admin, referenced by PVC

Access Mode         Concurrent nodes    Typical backend
──────────────      ────────────────    ───────────────────
ReadWriteOnce         1                   cloud block storage (EBS, PD, Azure Disk)
ReadOnlyMany           many (read-only)     shared read-only datasets
ReadWriteMany           many (read-write)     NFS, CephFS, Azure Files
ReadWriteOncePod         1 Pod, strictly        anything needing single-writer guarantee

Reclaim Policy        Underlying storage on PVC delete
──────────────        ────────────────────────────────
Delete                  destroyed automatically
Retain                   survives, needs manual cleanup
```

---

# INTERVIEW QUESTIONS

**Fundamentals**
1. Why are containers ephemeral by design, and what problem does that create for stateful data?
2. What's the difference between a Volume, a PersistentVolume, and a PersistentVolumeClaim?

**The chain**
3. Trace the full path from a Pod's volume reference down to actual physical storage.
4. Why is storage provisioning split across PVC, PV, StorageClass, and CSI driver instead of being one object?
5. What is CSI, and what problem did it solve compared to Kubernetes' old in-tree volume plugins?

**Provisioning**
6. What's the difference between static and dynamic provisioning?
7. Why does `volumeBindingMode: WaitForFirstConsumer` matter in a multi-zone cluster?

**Access modes**
8. Explain the difference between ReadWriteOnce, ReadOnlyMany, and ReadWriteMany.
9. Why can't a typical AWS EBS volume support ReadWriteMany?

**Lifecycle**
10. What's the difference between the `Delete` and `Retain` reclaim policies, and what's the production risk of using `Delete` carelessly?
11. Does deleting a Pod delete its PVC? Does deleting a PVC always delete the underlying storage?

**StatefulSet storage**
12. Why can't a Deployment provide the same storage guarantees a StatefulSet does?
13. What does `volumeClaimTemplates` actually create, and how many PVCs result from 3 replicas?
14. If a StatefulSet Pod is deleted and recreated, does it get a new PVC or the same one? Why does that matter?
15. Does deleting a StatefulSet delete its PVCs by default? Why is that the deliberate choice?

**Snapshots/backup**
16. What is a VolumeSnapshot, and why is it not by itself a complete backup strategy?

**Troubleshooting (scenario-based)**
17. A PVC is stuck `Pending` — what are your first three checks?
18. A Pod fails with `FailedAttachVolume` right after a node failure and fast rescheduling — what's the likely cause?
19. A Pod's volume mounts successfully but the application gets "permission denied" writing to it — what field fixes this, and why?
20. Volume provisioning fails with no obvious Kubernetes-side misconfiguration — where would you look next?

---

*This guide covers Kubernetes storage from ephemeral-container fundamentals through the full PVC → PV → StorageClass → CSI chain, access modes, StatefulSet per-replica storage, and production backup strategy — the complete arc from beginner to storage-architect-level understanding.*

# Kubernetes RBAC — Complete Mastery Guide
### From Absolute Beginner to Production Security Architecture

---

# PART 1 — AUTHENTICATION VS AUTHORIZATION

*(Both already introduced structurally in the Architecture masterclass, Part 3 §24 — this guide goes deep specifically on the authorization stage, since that's what RBAC actually is.)*

## Authentication (AuthN) — "Who are you?"
Kubernetes doesn't have its own built-in user database — identity comes from external mechanisms: client certificates, OIDC tokens (from an identity provider like Google/Okta/Azure AD), static tokens, or ServiceAccount tokens for in-cluster workloads. By the time a request reaches the authorization stage, it carries exactly one thing forward: an **identity** — a username and a set of group memberships.

## Authorization (AuthZ) — "Are you allowed to do THIS?"
Given that identity, is this **specific request** (this verb, on this resource, in this namespace) permitted? This is where **RBAC** (Role-Based Access Control) lives — it is Kubernetes' dominant, though not only, authorization mode (others exist — ABAC, Webhook, Node — but RBAC is what the overwhelming majority of real clusters use).

```
Request arrives
      │
      ▼
AUTHENTICATION → produces an identity: "user: alice, groups: [developers]"
                  or "serviceaccount: default:my-app-sa"
      │
      ▼
AUTHORIZATION (RBAC) → "is alice, or the developers group, permitted
                         to do THIS specific verb/resource/namespace
                         combination?"
      │
      ▼
   allowed → proceed to admission control (Architecture masterclass §24)
   denied  → 403 Forbidden, request stops here entirely
```

---

# PART 2 — THE RBAC MENTAL MODEL

## The one sentence that is all of RBAC
**Who → can perform what action → on which resource → in which namespace.**

```
   WHO                  WHAT ACTION           WHICH RESOURCE        WHICH SCOPE
 (subject)               (verb)                (resource type)      (namespace or cluster)
─────────────         ──────────────         ─────────────────    ───────────────────
 User: alice            get, list, watch        pods                 namespace: "dev"
 Group: developers       create, update           deployments           namespace: "staging"
 ServiceAccount: sa       delete, patch              secrets              cluster-wide
```
Every RBAC question you'll ever be asked reduces to filling in these four blanks. The entire rest of this guide is just the mechanics of how Kubernetes objects express that sentence.

## The four RBAC objects, and how they pair up

```
Role / ClusterRole          =  WHAT ACTION + WHICH RESOURCE
                                (a reusable permission set, no "who" yet)

RoleBinding / ClusterRoleBinding  =  WHO  +  attaches to a Role/ClusterRole
                                       (connects a subject to a permission set)
```

| | Namespace-scoped | Cluster-scoped |
|---|---|---|
| Permission set | **Role** | **ClusterRole** |
| Attaches subjects | **RoleBinding** | **ClusterRoleBinding** |

**Four combinations, each with a distinct real meaning** — this 2×2 is the single most important table in this guide, unpacked fully in Part 4.

---

# PART 3 — VERBS, RESOURCES, AND API GROUPS

## Verbs
The action being performed — mapped directly onto the REST HTTP methods from the Kubernetes API (Architecture masterclass §18):

| Verb | Roughly maps to |
|---|---|
| `get` | Read one specific object |
| `list` | Read a collection of objects |
| `watch` | Subscribe to changes (the mechanism behind Informers, Architecture masterclass §22–23) |
| `create` | POST a new object |
| `update` | Replace an existing object entirely |
| `patch` | Partially modify an existing object |
| `delete` | Remove an object |
| `deletecollection` | Remove a whole collection at once |

## Resources
The object **type** being acted on — `pods`, `deployments`, `secrets`, `configmaps` — always the **plural, lowercase** REST resource name, exactly as it appears in the API path (Architecture masterclass §18), not the `kind` field's capitalized singular form.

## API groups
Because different resource types live in different API groups (Architecture masterclass §18), a Role must specify **which group** each resource belongs to:

```yaml
rules:
  - apiGroups: [""]              # the "core" group — pods, services, configmaps, secrets
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]           # deployments, replicasets, statefulsets, daemonsets
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
```
**`apiGroups: [""]` (empty string) specifically means the core group** — a common point of confusion, since it looks like an omission rather than a deliberate value.

```bash
kubectl api-resources    # lists every resource type WITH its API group — the authoritative reference
```

---

# PART 4 — Role, ClusterRole, RoleBinding, ClusterRoleBinding

## Role — namespace-scoped permission set
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev                  # this Role ONLY has meaning within the "dev" namespace
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```
This Role, by itself, grants **nobody** anything — it's purely a named, reusable permission template, inert until bound to a subject.

## RoleBinding — attaches subjects to a Role, within one namespace
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader
  namespace: dev
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
- `subjects` — WHO (a User, Group, or ServiceAccount — §5)
- `roleRef` — WHICH permission set this binding grants
- **Namespace of the RoleBinding itself determines the scope** — this grant only applies within `dev`, even though "alice" as an identity is cluster-wide; alice has zero pod-reading permission in any other namespace unless a separate binding grants it there too

## ClusterRole — the same shape, but reusable cluster-wide (or for cluster-scoped resources)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader          # NOT namespaced — ClusterRoles never are
rules:
  - apiGroups: [""]
    resources: ["nodes"]      # Nodes are CLUSTER-scoped resources — a Role could never grant this at all
    verbs: ["get", "list"]
```
**Why ClusterRole is sometimes mandatory, not just a convenience:** some resources (`nodes`, `persistentvolumes`, `namespaces` themselves, `clusterroles`) have **no namespace at all** — a plain `Role` is structurally incapable of granting permission on them, since a Role's scope is always exactly one namespace. Any permission touching a genuinely cluster-scoped resource *must* go through a ClusterRole.

## ClusterRoleBinding — grants a ClusterRole cluster-wide
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: alice-node-reader
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

## The fourth, commonly-missed combination: ClusterRole + RoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader-in-dev
  namespace: dev              # <- RoleBinding, so scope is limited to "dev"
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole            # <- referencing a CLUSTER-scoped permission SET
  name: generic-pod-reader      #    (defined once, reusable everywhere)
  apiGroup: rbac.authorization.k8s.io
```
**This is an extremely common, powerful production pattern:** define a `ClusterRole` **once** (e.g., a generic "pod-reader" permission set), then grant it to different users in different namespaces via **separate RoleBindings**, each scoping that same reusable permission set to just one namespace — avoiding having to redefine an identical `Role` object by hand in every namespace that needs the same permissions.

```
┌─────────────────────────────────────────────────────────────┐
│  ClusterRole "generic-pod-reader" (defined ONCE, cluster-wide)  │
└───────────┬─────────────────────────┬─────────────────────┘
            │                           │
   RoleBinding (namespace: dev)   RoleBinding (namespace: staging)
            │                           │
            ▼                           ▼
   Grants pod-read ONLY in "dev"   Grants pod-read ONLY in "staging"
   (to whichever subjects each RoleBinding names — could be
    different people/teams per namespace, sharing the same
    underlying permission definition)
```

---

# PART 5 — SUBJECTS: USERS, GROUPS, SERVICEACCOUNTS

## Users and Groups
Kubernetes has **no User or Group API objects at all** — these are purely identities asserted by whatever authentication mechanism is in front of the API server (a client certificate's Common Name/Organization fields, an OIDC token's claims). RBAC references them by name, but there is nothing to `kubectl get users` — they exist only as strings that authentication produces and RBAC subsequently checks against.

## ServiceAccounts — the one subject type that IS a real Kubernetes object
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: dev
```
Every Pod runs **as** some ServiceAccount (a default one is auto-provisioned per namespace if none is specified explicitly) — this is the identity a **process inside a Pod** uses when it calls the Kubernetes API itself (a controller you wrote, a CI/CD tool running inside the cluster, `kubectl` invoked from within a Pod).

```yaml
# on the Pod itself:
spec:
  serviceAccountName: my-app-sa
```
```yaml
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: dev              # ServiceAccounts ALWAYS need their namespace specified in a binding,
                                  # since the same NAME could exist in multiple namespaces as
                                  # genuinely different ServiceAccount objects
```
**This directly connects to the Secrets guide's most important RBAC-adjacent security fact:** every Pod, by default, gets its ServiceAccount's token mounted automatically — so **any process running inside that Pod effectively inherits whatever RBAC permissions that ServiceAccount has been granted**, which is exactly why scoping ServiceAccount RBAC tightly (and disabling auto-mounting where a Pod never needs to call the API at all, via `automountServiceAccountToken: false`) is a genuine, high-leverage security control, not a nitpick.

---

# PART 6 — YAML EXAMPLES: FIVE REAL-WORLD ROLES

## 1. Read-only user
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-only
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["*"]              # every resource type in these groups
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bob-read-only
subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: read-only
  apiGroup: rbac.authorization.k8s.io
```
- `resources: ["*"]` — wildcard, matches every resource type within the listed API groups
- `verbs` limited to read-only (`get`/`list`/`watch`) — bob can inspect anything in these groups but create/modify/delete nothing at all

## 2. Developer (namespace-scoped, broader verbs, no Secrets access)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # NOTE: "secrets" deliberately OMITTED — per the Secrets guide's
  # RBAC-scoping principle, developers get broad workload permissions
  # but NOT blanket secret access
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-developer
  namespace: dev
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```
- `subjects: kind: Group` — binds an entire group at once, rather than naming individuals one by one — the standard, scalable pattern for team-based access, since adding a new team member only requires updating the identity provider's group membership, not touching any Kubernetes RBAC object at all

## 3. Namespace administrator
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: team-a
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]                  # full control...
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-admins
  namespace: team-a
subjects:
  - kind: Group
    name: team-a-leads
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io
```
- `verbs: ["*"]` within a **Role** (not ClusterRole) — this is full control, but **strictly bounded to `team-a`'s namespace** — this subject cannot touch anything in `team-b`, cannot see cluster-scoped resources (nodes, PVs, other namespaces) at all — this is precisely the safe way to grant "admin-like" power without it becoming actual cluster-admin

## 4. Pod reader (minimal, single-purpose)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-only
  namespace: monitoring
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["pods/log"]        # a SUBRESOURCE — logs require explicit, SEPARATE permission
    verbs: ["get"]
```
- `pods/log` — subresources (logs, exec, status, scale) require their **own** explicit RBAC entry, distinct from the parent resource — granting `get` on `pods` alone does **not** implicitly grant `kubectl logs` access; this is a very common real-world gotcha for anyone building a minimal monitoring/debugging role

## 5. Deployment manager (CI/CD pipeline pattern)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]           # read-only, for verifying rollout status
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-cd-deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-cd-deployment-manager
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ci-cd-deployer
    namespace: production
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```
- No `create`/`delete` on Deployments — deliberately scoped to **updating existing** Deployments only (the exact operation a rollout pipeline performs via `kubectl set image`/`kubectl apply` on an already-existing object) — this pipeline's credentials, if ever leaked, cannot create brand-new workloads or delete existing ones, only push updates to what's already there

---

# PART 7 — kubectl auth can-i

The direct, authoritative way to check what a permission set actually allows, without trial and error:

```bash
kubectl auth can-i create deployments --namespace=dev
kubectl auth can-i delete secrets --namespace=production --as=bob
kubectl auth can-i get pods --as=system:serviceaccount:dev:my-app-sa
kubectl auth can-i '*' '*' --as=alice          # "can alice do literally anything, anywhere?"
kubectl auth can-i --list --namespace=dev       # everything the CURRENT identity can do here
kubectl auth can-i --list --as=bob --namespace=dev
```
- `--as` — impersonates another identity for the check (requires the caller to itself have `impersonate` permission — worth noting this is itself an RBAC-gated capability, not a universal debugging bypass)
- `--list` — dumps every permission the given identity has in the specified scope, the single fastest way to audit "what can this actually do" without reading every Role/Binding by hand

```bash
# real-world debugging flow: a CI pipeline's deploy step is failing with 403
kubectl auth can-i update deployments --namespace=production \
  --as=system:serviceaccount:production:ci-cd-deployer
# → "no" confirms an RBAC gap; "yes" means the 403 has some OTHER cause
# (wrong ServiceAccount actually attached to the Pod, wrong namespace, etc.)
```

---

# PART 8 — DIAGRAMS

## The full decision, visually
```
             "Can alice DELETE a POD in the DEV namespace?"
                              │
                              ▼
        Find every RoleBinding/ClusterRoleBinding naming "alice"
        (directly, or via a Group alice belongs to) that is either:
          - a RoleBinding IN the "dev" namespace, OR
          - a ClusterRoleBinding (applies everywhere, including "dev")
                              │
                              ▼
        For each such binding, look up its referenced Role/ClusterRole's
        rules — does ANY rule match: apiGroup="", resource="pods", verb="delete"?
                              │
              ┌────────────────┴────────────────┐
              ▼                                    ▼
          YES, at least one rule matches      NO rule matches, anywhere
              │                                    │
              ▼                                    ▼
          ALLOWED (RBAC is purely ADDITIVE —     DENIED — 403 Forbidden
          if ANY applicable binding grants it,   (there is NO explicit "deny"
          it's allowed — there's no "deny"        rule type in RBAC at all;
          rule type to override this)              absence of a grant IS the deny)
```

## RBAC is purely additive — no explicit deny
**This is the single most important RBAC design fact to internalize:** you cannot write a rule that says "deny X." Permissions only ever accumulate across every applicable binding — the only way to prevent an action is to ensure **no** binding anywhere grants it. This is precisely why auditing "what can this identity NOT do" is fundamentally a process of enumerating everything they CAN do (across every Role/ClusterRoleBinding that applies to them) and confirming the dangerous thing isn't among them — there's no single rule you can point to that "blocks" something.

---

# PART 9 — SECURITY BEST PRACTICES

## Least privilege
Grant the minimum verb/resource/namespace combination that lets a subject do its actual job — not "give broad access and restrict later," which almost never actually happens in practice once something works. The Deployment Manager example (§6, item 5) is a concrete template: `update`/`patch` only, no `create`/`delete`, scoped to one resource type, in one namespace.

## Avoiding cluster-admin
```bash
kubectl get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name=="cluster-admin")'
```
`cluster-admin` is a built-in ClusterRole granting `verbs: ["*"]` on `resources: ["*"]` in `apiGroups: ["*"]`, cluster-wide — genuinely unrestricted control over everything, including RBAC itself (a cluster-admin can grant themselves or anyone else any other permission, trivially). It should be bound to as few identities as possible — ideally just a small number of humans for genuine break-glass emergencies, and audited on a recurring basis, never handed out routinely "to make something work."

## Namespace isolation
RBAC's namespace scoping (Role/RoleBinding) is one pillar of multi-tenancy — but remember it's **only** an API-level boundary; it says nothing about network reachability (that's NetworkPolicy, from the Networking guide) or resource fairness (that's ResourceQuota, from the Resource Limits guide). Genuine tenant isolation requires **all three** layered together — RBAC alone stops a team from reading/editing another team's *objects*, but doesn't stop their Pods from reaching each other over the network or consuming all shared node capacity.

## ServiceAccount permissions
- **Never use the `default` ServiceAccount** for anything with real permissions — it's auto-created in every namespace and easy to accidentally grant broad permissions to without realizing every unspecified Pod in that namespace inherits them
- **Set `automountServiceAccountToken: false`** on Pods (or the ServiceAccount itself) that never call the Kubernetes API at all — reduces the blast radius of a container compromise to "no API credentials available inside it whatsoever"
- **One ServiceAccount per application/purpose**, not one shared broad ServiceAccount reused across many unrelated workloads — mirrors the "least privilege" principle applied specifically to machine identities rather than human ones

---

# PART 10 — TROUBLESHOOTING

**"Forbidden" errors despite believing the right RBAC is in place**
```bash
kubectl auth can-i <verb> <resource> --namespace=<ns> --as=<identity>
```
Always start here — it's authoritative and instant, versus manually re-reading YAML and hoping you've traced every applicable binding correctly by eye.

**A ServiceAccount's Pod gets 403s calling the API**
Confirm the Pod is actually running as the ServiceAccount you think it is:
```bash
kubectl get pod <pod> -o jsonpath='{.spec.serviceAccountName}'
```
A very common mistake: creating a correctly-scoped RoleBinding for `my-app-sa`, while the Pod itself is still silently running as `default` because `serviceAccountName` was never actually set on the Pod spec.

**A RoleBinding "isn't working"**
Check for a **Role vs ClusterRole** mismatch in `roleRef.kind` — a RoleBinding referencing a `ClusterRole` that doesn't actually exist (typo'd name, or it's actually a `Role` in a different namespace) silently grants nothing, with no obvious error pointing at the exact cause beyond the eventual 403.

**Someone has more access than expected**
Remember RBAC is purely additive (Part 8) — check for **group membership** granting access via a completely different binding than the one you were looking at; a user can accumulate permissions from their own direct bindings, plus every group they belong to, plus any binding naming `system:authenticated` or similarly broad built-in groups.

---

# PART 11 — REAL-WORLD SCENARIOS

- **A compromised CI/CD ServiceAccount with `cluster-admin`** is one of the most common real-world Kubernetes security incidents — a leaked pipeline credential with unrestricted cluster access turns a single leaked secret into full cluster compromise; scoping that ServiceAccount to exactly the Deployment Manager pattern from §6 instead would have contained the blast radius to "can update Deployments in one namespace."
- **A support/on-call engineer granted `view` (a built-in, broad read-only ClusterRole) cluster-wide** so they can debug incidents anywhere, without ever being handed write access anywhere — a genuinely common and reasonable use of a broad-but-read-only ClusterRoleBinding.
- **Namespace-per-team with a shared `ClusterRole` + per-namespace `RoleBinding`s** (Part 4's pattern) is the standard way platform teams scale RBAC across dozens of teams without redefining the same permission set by hand in every namespace.

---

# PART 12 — MINIKUBE LABS

### Lab 1: Prove RBAC's additive-only nature
```bash
minikube start
kubectl create namespace dev
kubectl create serviceaccount limited-sa -n dev
kubectl auth can-i get pods -n dev --as=system:serviceaccount:dev:limited-sa
# → no
kubectl create role pod-reader -n dev --verb=get,list,watch --resource=pods
kubectl create rolebinding limited-sa-binding -n dev \
  --role=pod-reader --serviceaccount=dev:limited-sa
kubectl auth can-i get pods -n dev --as=system:serviceaccount:dev:limited-sa
# → yes
kubectl auth can-i delete pods -n dev --as=system:serviceaccount:dev:limited-sa
# → still no — get/list/watch were granted, delete never was
```

### Lab 2: Subresource gotcha
```bash
kubectl run test-pod --image=nginx -n dev
kubectl auth can-i get pods/log -n dev --as=system:serviceaccount:dev:limited-sa
# → no, even though "get pods" is allowed — logs are a separate subresource
```

### Lab 3: ClusterRole + RoleBinding pattern
```bash
kubectl create clusterrole generic-reader --verb=get,list,watch --resource=pods,deployments
kubectl create namespace staging
kubectl create rolebinding staging-reader -n staging \
  --clusterrole=generic-reader --serviceaccount=dev:limited-sa
kubectl auth can-i get deployments -n staging --as=system:serviceaccount:dev:limited-sa
# → yes, in staging
kubectl auth can-i get deployments -n dev --as=system:serviceaccount:dev:limited-sa
# → no — the SAME ClusterRole was never bound in "dev" for this permission
```

### Lab 4: Audit an identity's full permission set
```bash
kubectl auth can-i --list --as=system:serviceaccount:dev:limited-sa -n dev
kubectl auth can-i --list --as=system:serviceaccount:dev:limited-sa -n staging
```

---

# INTERVIEW QUESTIONS

**Fundamentals**
1. What's the difference between authentication and authorization?
2. State the "who → what → which resource → which scope" model in your own words.
3. Does Kubernetes have a User object? Where does user identity actually come from?

**Core objects**
4. What's the difference between a Role and a ClusterRole?
5. What's the difference between a RoleBinding and a ClusterRoleBinding?
6. Why must some resources always use a ClusterRole, never a plain Role?
7. Explain the ClusterRole + RoleBinding pattern and why it's useful at scale.

**Mechanics**
8. What does `apiGroups: [""]` mean, and why does it look confusing at first?
9. Why does granting `get` on `pods` not automatically grant `kubectl logs` access?
10. Is RBAC additive, subtractive, or both? Can you write an explicit "deny" rule?

**ServiceAccounts**
11. What's the difference between a User/Group subject and a ServiceAccount subject, architecturally?
12. Why does every Pod need its ServiceAccount specified explicitly in a RoleBinding's namespace field?
13. What security risk does the `default` ServiceAccount pose if left unmanaged?
14. What does `automountServiceAccountToken: false` protect against?

**Best practices**
15. Why is `cluster-admin` dangerous even when "only used occasionally"?
16. Why isn't RBAC alone sufficient for true multi-tenant namespace isolation?

**Troubleshooting (scenario-based)**
17. A user gets 403 Forbidden despite what looks like a correctly configured RoleBinding — what's your first diagnostic command?
18. A Pod's application code gets 403s calling the Kubernetes API — what are your first two checks?
19. You suspect a user has more access than intended — why can't you find this by reading just one RoleBinding?
20. How would you verify, with a single command, exactly what a given ServiceAccount can do across an entire namespace?

---

*This guide covers Kubernetes RBAC from the authentication/authorization split through Role/ClusterRole mechanics, ServiceAccount security, and production least-privilege design — the complete arc from beginner to security-architect-level authorization design.*


# Kubernetes ServiceAccounts — Complete Masterclass (Beginner → Production)

---

## PART 1 — CONCEPTS

### What a ServiceAccount is

A **ServiceAccount** is a Kubernetes-native identity for **processes running inside Pods** — as opposed to a human user identity. When your application code needs to call the Kubernetes API (list Pods, read a Secret, watch ConfigMaps, create Jobs), it authenticates as a ServiceAccount, not as "you."

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
```

That's the entire minimal object — a ServiceAccount by itself is just a named identity with no permissions. Permissions come entirely from **RBAC bindings** layered on top (covered below).

### Why Pods need identities

Every request to the Kubernetes API server must be **authenticated as someone** — there's no anonymous access by default in a properly configured cluster. Two categories of "someone" exist:

| | Users | ServiceAccounts |
|---|---|---|
| Who/what it represents | A human (or external system) | A process running inside a Pod |
| Managed by | External identity provider (OIDC, certs) — Kubernetes has **no** User object | Kubernetes-native object (`kind: ServiceAccount`) — created/managed via API |
| Typical use | `kubectl` from your laptop | An app Pod calling the K8s API, a controller, an operator |
| Credential | kubeconfig (cert or OIDC token) | Automatically mounted token inside the Pod |

If a Pod needs to introspect the cluster — e.g., a controller watching for CRDs, a CI job creating other Pods, an app doing service discovery via the API — **it must present some identity** to be authenticated and then authorized. That's what ServiceAccounts exist for.

### ServiceAccount vs User

This is a common interview trip-up: **Kubernetes has no `User` API object at all.** Users are entirely external — authenticated via client certificates, OIDC tokens, or a cloud IAM integration, and the cluster just trusts whatever identity that external mechanism asserts (a username + group list). ServiceAccounts, by contrast, are **first-class API objects** you can create, list, delete, and bind RBAC roles to, entirely within the cluster.

```
User:            external identity → cluster trusts assertion (cert CN, OIDC claim)
ServiceAccount:  kubectl get sa    → an actual etcd-stored object, k8s-native
```

---

## PART 2 — TOKENS & AUTHENTICATION

### Token

Every ServiceAccount has an associated **credential** — a signed JWT bearer token — that a Pod using that ServiceAccount presents to the API server on every request, in the `Authorization: Bearer <token>` HTTP header.

There have been **two generations** of this mechanism:

**Legacy (pre-1.24, still supported):** a long-lived token stored in a `Secret` object, auto-created for every ServiceAccount, auto-mounted into Pods. Never expires unless manually deleted.

**Modern (1.24+, default today): Projected Service Account Tokens** — short-lived, audience-scoped, auto-rotating tokens generated on-demand by the kubelet via the **TokenRequest API**, not stored as a long-lived Secret at all.

### Projected Service Account Tokens

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-sa
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: kube-api-access
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      readOnly: true
  volumes:
  - name: kube-api-access
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600     # token auto-rotates, valid 1hr at a time
          audience: api                # scopes token to a specific intended audience
      - configMap:
          name: kube-root-ca.crt
          items:
          - key: ca.crt
            path: ca.crt
      - downwardAPI:
          items:
          - path: namespace
            fieldRef:
              fieldPath: metadata.namespace
```

**This is auto-injected by kubelet for every Pod by default** — you rarely write this YAML by hand; it's shown here to demystify what's actually happening under `/var/run/secrets/kubernetes.io/serviceaccount/` inside every Pod.

**Why this is better than the old long-lived Secret token:**
- **Short-lived** (`expirationSeconds`, default 1hr) — kubelet transparently refreshes it before expiry, but a leaked/exfiltrated token becomes useless quickly
- **Audience-scoped** — a token requested for the `api` audience can't be replayed against a different service expecting a different audience
- **Bound to the Pod** — the token includes claims tying it to the specific Pod's UID and node; if the Pod is deleted, the token is immediately invalid, unlike the old Secret-based token which lived independently of any particular Pod

### API Server Authentication

When a request arrives with `Authorization: Bearer <token>`, the API server's **ServiceAccount token authenticator** verifies:

1. The token's signature (against the cluster's signing key)
2. The token hasn't expired
3. (For projected tokens) the token is bound to a Pod that **still exists** and hasn't been deleted

If valid, the API server extracts the identity as:
```
username: system:serviceaccount:<namespace>:<serviceaccount-name>
groups:   system:serviceaccounts
          system:serviceaccounts:<namespace>
          system:authenticated
```

This synthetic username (`system:serviceaccount:default:myapp-sa`) is exactly what RBAC `RoleBinding`/`ClusterRoleBinding` subjects reference.

---

## PART 3 — RBAC (AUTHORIZATION)

Authentication answers "who are you." **RBAC** answers "what are you allowed to do." A ServiceAccount with zero RoleBindings can authenticate successfully but will be authorized for **nothing** — every API call returns `403 Forbidden`.

### Role / ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                          # namespace-scoped permission set
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]                    # "" = core API group (Pods, Services, ConfigMaps...)
  resources: ["pods"]
  verbs: ["get", "list", "watch"]     # read-only access to Pods
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole                    # cluster-wide (or reusable across namespaces via binding)
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

- `apiGroups` — which API group the resource belongs to (`""` = core, `apps` = Deployments, `batch` = Jobs, etc.)
- `resources` — the resource type (`pods`, `secrets`, `configmaps`, `deployments`)
- `verbs` — allowed operations (`get`, `list`, `watch`, `create`, `update`, `patch`, `delete`)

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-sa-pod-reader
  namespace: default
subjects:
- kind: ServiceAccount
  name: myapp-sa                    # WHICH ServiceAccount gets this permission
  namespace: default
roleRef:
  kind: Role                         # WHICH Role is being granted
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

This is the **glue** — it says "the `myapp-sa` ServiceAccount in namespace `default` now has all permissions defined in the `pod-reader` Role." Without this binding, the `pod-reader` Role exists but grants nothing to anyone.

```
ServiceAccount "myapp-sa"  ──bound via RoleBinding──►  Role "pod-reader"
                                                          (get/list/watch pods)
```

`ClusterRoleBinding` works identically but grants a `ClusterRole`'s permissions **cluster-wide**, across all namespaces — used sparingly, for genuinely cluster-scoped needs (e.g., a monitoring agent reading nodes cluster-wide).

---

## PART 4 — THE COMPLETE FLOW

```
┌─────────┐
│   Pod    │  spec.serviceAccountName: myapp-sa
└────┬────┘
     │ kubelet auto-mounts projected token at
     │ /var/run/secrets/kubernetes.io/serviceaccount/token
     ▼
┌──────────────────┐
│  ServiceAccount    │  myapp-sa (namespace: default)
│  "myapp-sa"         │
└────────┬──────────┘
         │ token = signed JWT, claims include:
         │   sub: system:serviceaccount:default:myapp-sa
         │   aud: api
         │   exp: <1hr from issuance>
         ▼
┌──────────────────┐
│      Token         │  app code reads this file, sends as:
│  (JWT, mounted)     │  Authorization: Bearer <token>
└────────┬──────────┘
         │  HTTPS request to API server
         ▼
┌──────────────────┐
│   kube-apiserver    │
└────────┬──────────┘
         │
         ▼
┌──────────────────┐
│ Authentication      │  verify JWT signature, expiry, audience,
│ (ServiceAccount      │  and (for bound tokens) that the Pod still exists
│  token authenticator)│  → resolves identity:
└────────┬──────────┘     system:serviceaccount:default:myapp-sa
         │  (if invalid → 401 Unauthorized, stop here)
         ▼
┌──────────────────┐
│ RBAC Authorization  │  does this identity have a RoleBinding/
│                      │  ClusterRoleBinding granting the requested
│                      │  verb on the requested resource?
└────────┬──────────┘
         │  (if not allowed → 403 Forbidden, stop here)
         ▼
┌──────────────────┐
│   API Response      │  request proceeds — e.g., returns the Pod list
└──────────────────┘
```

### How a Pod actually accesses the API (in-cluster client code)

```python
# Most Kubernetes client libraries auto-detect this "in-cluster config":
import kubernetes
kubernetes.config.load_incluster_config()
# under the hood, this reads:
#   token:     /var/run/secrets/kubernetes.io/serviceaccount/token
#   ca.crt:    /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
#   namespace: /var/run/secrets/kubernetes.io/serviceaccount/namespace
#   apiserver: from KUBERNETES_SERVICE_HOST / KUBERNETES_SERVICE_PORT env vars
#              (automatically injected into every Pod by kubelet)
```

Or manually via `curl` from inside a Pod (for understanding, not typical production code):
```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
curl --cacert $CACERT \
     -H "Authorization: Bearer $TOKEN" \
     https://kubernetes.default.svc/api/v1/namespaces/default/pods
```

---

## PART 5 — DEFAULT SERVICEACCOUNT & AUTOMOUNT CONTROL

### The default ServiceAccount

Every namespace automatically gets a ServiceAccount named `default`. **Any Pod that doesn't explicitly specify `serviceAccountName` uses this one.**

```bash
kubectl get sa default -n default
kubectl get sa default -n default -o yaml
```

By itself, `default` has **no RBAC permissions** bound to it out of the box in a properly locked-down cluster — but if someone has previously bound it to a permissive Role/ClusterRole (a common misconfiguration), **every unlabeled Pod in that namespace silently inherits that access.** This is one of the most common real-world Kubernetes privilege-escalation footguns.

### `automountServiceAccountToken`

By default, every Pod gets the ServiceAccount token auto-mounted — **even if the app never calls the Kubernetes API.** This is unnecessary attack surface: if that Pod is compromised (e.g., via a code injection vulnerability), the attacker now has a live credential to probe the Kubernetes API with, for free.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
automountServiceAccountToken: false    # set at ServiceAccount level (applies to Pods using it)
---
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false   # can also override per-Pod
  containers:
  - name: app
    image: myapp:1.0
```

**Best practice:** disable this by default for any workload that doesn't genuinely need to talk to the Kubernetes API (most application Pods — web servers, workers, databases — never need it).

---

## PART 6 — WORKLOAD IDENTITY & CLOUD IAM INTEGRATION

### The problem this solves

A Pod's ServiceAccount token authenticates it **to the Kubernetes API** — it says nothing about **cloud provider permissions** (e.g., "can this Pod read from this S3 bucket / GCS bucket / Azure Blob container"). Historically, teams solved this by injecting long-lived cloud credentials (AWS access keys, GCP service account JSON keys) as Kubernetes Secrets — a significant, static, hard-to-rotate liability.

**Workload Identity** (the generic term; AWS calls it IRSA — "IAM Roles for Service Accounts", GCP calls it "Workload Identity", Azure calls it "Workload Identity Federation") lets a Kubernetes ServiceAccount's **token itself** be trusted directly by the cloud IAM system, via **OIDC federation** — no static cloud credentials stored in the cluster at all.

```
Kubernetes ServiceAccount token (OIDC-compliant JWT)
       │
       ▼
Cloud IAM trusts the cluster's OIDC issuer (configured once, cluster-wide)
       │
       ▼
Cloud IAM maps: "this specific ServiceAccount (namespace + name)"
                 → "this specific cloud IAM role"
       │
       ▼
Pod's SDK calls (boto3, google-cloud SDK, azure SDK) automatically
   exchange the K8s SA token for temporary cloud credentials
       │
       ▼
Pod can call cloud APIs (S3, GCS, etc.) with SHORT-LIVED, auto-rotated
   credentials — no static keys anywhere
```

### Example — AWS IRSA

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-reader-role
    # this annotation is what wires the K8s ServiceAccount to a specific AWS IAM role
---
apiVersion: v1
kind: Pod
metadata:
  name: s3-app
spec:
  serviceAccountName: s3-reader-sa
  containers:
  - name: app
    image: myapp:1.0
    # AWS SDK auto-detects AWS_WEB_IDENTITY_TOKEN_FILE + AWS_ROLE_ARN env vars,
    # which EKS's Pod Identity webhook injects automatically based on the
    # ServiceAccount annotation above — no code changes needed
```

**Internally:** an EKS-installed mutating webhook sees a Pod using a ServiceAccount with the `eks.amazonaws.com/role-arn` annotation, and injects `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` env vars plus a projected token volume pointing at that role. The AWS SDK inside the container then automatically calls STS `AssumeRoleWithWebIdentity`, presenting the Kubernetes-issued OIDC token, and receives short-lived AWS credentials in return.

This is the modern standard for cloud-integrated Kubernetes workloads — eliminate static cloud secrets entirely.

---

## PART 7 — SECURITY RISKS & BEST PRACTICES

### Security risks

1. **Overly broad `default` ServiceAccount permissions** — if `default` in a namespace is bound to a permissive Role/ClusterRole (even accidentally, or via a legacy Helm chart), every Pod that doesn't specify its own ServiceAccount inherits that access silently.
2. **Unnecessary token automount** — Pods that never call the K8s API still get a live credential mounted, expanding blast radius if compromised.
3. **Overly broad ClusterRoleBindings** — binding `cluster-admin` (or anything close to it) to an application ServiceAccount "to make an error go away" is a critical common mistake — it means a single Pod compromise can pivot to full cluster control.
4. **Legacy long-lived Secret-based tokens** — if still in use (older clusters, `automountServiceAccountToken` referencing an explicit long-lived Secret), a leaked token has no expiry and remains valid until manually revoked/rotated.
5. **Cross-namespace token confusion** — tokens are namespace-scoped identities, but RBAC misconfig (e.g., a `ClusterRoleBinding` instead of a `RoleBinding`) can inadvertently grant a namespace-local ServiceAccount cluster-wide reach.
6. **Wildcard RBAC rules** — `resources: ["*"]`, `verbs: ["*"]` grants used "temporarily" during debugging and never cleaned up.

### Best Practices

- **One ServiceAccount per workload/purpose** — never share a single ServiceAccount across unrelated apps; blast radius should be as narrow as possible
- **Never bind broad permissions to `default`** — leave it permission-less; require every workload needing API access to use an explicitly created, narrowly-scoped ServiceAccount
- **Set `automountServiceAccountToken: false`** on the ServiceAccount (or Pod) for any workload that doesn't call the Kubernetes API — which is most application workloads
- **Principle of least privilege in RBAC** — grant only the specific `verbs` on the specific `resources` actually needed; prefer `Role`+`RoleBinding` (namespace-scoped) over `ClusterRole`+`ClusterRoleBinding` unless truly cluster-wide access is required
- **Use projected, short-lived tokens** (the 1.24+ default) — avoid reintroducing long-lived Secret-based tokens unless you have a specific, justified need
- **Use Workload Identity / IRSA for cloud access** — never store static cloud credentials as Kubernetes Secrets when a federation mechanism is available
- **Audit RBAC regularly** — tools like `kubectl-who-can`, `rbac-lookup`, or `kubectl auth can-i --list --as=system:serviceaccount:ns:name` help catch drift and over-permissioning
- **Avoid wildcard verbs/resources in Roles** — enumerate explicitly

```bash
# Audit what a ServiceAccount can actually do:
kubectl auth can-i --list --as=system:serviceaccount:default:myapp-sa
kubectl auth can-i delete pods --as=system:serviceaccount:default:myapp-sa -n default
```

---

## PART 8 — HANDS-ON MINIKUBE LABS

```bash
minikube start
```

### Lab 1 — Create a ServiceAccount and inspect the mounted token
```bash
kubectl create serviceaccount demo-sa
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sa-demo-pod
spec:
  serviceAccountName: demo-sa
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
EOF
kubectl exec sa-demo-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/
kubectl exec sa-demo-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

### Lab 2 — 403 Forbidden with no RBAC bound
```bash
kubectl exec sa-demo-pod -- wget -q -O- \
  --header="Authorization: Bearer $(kubectl exec sa-demo-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  --no-check-certificate \
  https://kubernetes.default.svc/api/v1/namespaces/default/pods
# → 403 Forbidden — authenticated, but zero RBAC permissions
```

### Lab 3 — Grant read access via Role + RoleBinding
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: demo-sa-pod-reader
subjects:
- kind: ServiceAccount
  name: demo-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl auth can-i list pods --as=system:serviceaccount:default:demo-sa
# yes
kubectl auth can-i delete pods --as=system:serviceaccount:default:demo-sa
# no
```

### Lab 4 — Disable automount
```bash
kubectl patch serviceaccount demo-sa -p '{"automountServiceAccountToken": false}'
kubectl delete pod sa-demo-pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sa-demo-pod-2
spec:
  serviceAccountName: demo-sa
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
EOF
kubectl exec sa-demo-pod-2 -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# no such file or directory — no token mounted at all
```

### Lab 5 — Default ServiceAccount danger demo
```bash
kubectl run default-sa-pod --image=busybox -- sleep 3600
kubectl get pod default-sa-pod -o jsonpath='{.spec.serviceAccountName}'
# "default" — implicitly assigned
kubectl auth can-i list secrets --as=system:serviceaccount:default:default
# should be "no" in a properly locked-down cluster — verify it stays that way!
```

---

## PART 9 — TROUBLESHOOTING SCENARIOS

### "403 Forbidden" calling the API from inside a Pod

```bash
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa-name> -n <ns>
```
Check: is there a `RoleBinding`/`ClusterRoleBinding` at all? Does it reference the correct `Role`/`ClusterRole`? Is the `subjects[].namespace` correct (a common typo — binding to the wrong namespace silently does nothing)?

### Token file missing inside Pod

```bash
kubectl get pod <name> -o yaml | grep -A3 serviceAccount
```
Check `automountServiceAccountToken` at both Pod and ServiceAccount level — either can disable it, and Pod-level setting (if explicitly set) overrides the ServiceAccount's.

### "Unauthorized" (401) rather than "Forbidden" (403)

Distinct failure mode — means **authentication** failed, not authorization. Check token expiry, whether the Pod (and thus its bound token) still exists, and whether the token's `audience` claim matches what the API server expects.

```bash
kubectl exec <pod> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | cut -d. -f2 | base64 -d
# decode the JWT payload manually to inspect claims: exp, aud, sub
```

### App using cloud SDK gets "no credentials found" despite Workload Identity setup

Check the ServiceAccount annotation (`eks.amazonaws.com/role-arn` or GCP/Azure equivalent) is present and correctly formatted, the cluster's OIDC provider is registered in the cloud IAM trust policy, and the mutating webhook that injects the identity env vars/volume is actually running (`kubectl describe pod` should show the injected env vars if it worked).

---

## PART 10 — INTERVIEW QUESTIONS

1. What is a ServiceAccount, and what identity gap does it fill?
2. Why doesn't Kubernetes have a native `User` object, but does have `ServiceAccount`?
3. What username format does the API server derive from a ServiceAccount token?
4. What's the difference between legacy long-lived ServiceAccount tokens and projected service account tokens?
5. What is the TokenRequest API, and how does it relate to token rotation?
6. Where inside a Pod is the ServiceAccount token mounted by default?
7. What three files does kubelet typically project into a Pod for in-cluster API access?
8. What does `automountServiceAccountToken: false` do, and why is it a security best practice for most workloads?
9. What is the `default` ServiceAccount, and why is binding broad RBAC to it dangerous?
10. Walk through the full authentication + authorization flow when a Pod calls the Kubernetes API.
11. What's the difference between a 401 and a 403 response, in terms of which stage failed?
12. What's the difference between `Role` and `ClusterRole`?
13. What's the difference between `RoleBinding` and `ClusterRoleBinding`?
14. Can a `RoleBinding` reference a `ClusterRole`? What does that achieve?
15. What RBAC fields make up a `Role`'s `rules` entry?
16. How would you audit exactly what permissions a given ServiceAccount has?
17. What is Workload Identity (or IRSA on AWS), and what problem does it solve versus static cloud credentials?
18. Explain the OIDC federation trust chain in a cloud Workload Identity setup.
19. What cloud-side configuration links a specific ServiceAccount to a specific cloud IAM role?
20. Why is a JWT audience (`aud`) claim important for ServiceAccount token security?
21. What happens to a projected ServiceAccount token's validity if the Pod it's bound to is deleted?
22. What's a realistic attack scenario enabled by overly broad `cluster-admin` binding to an app ServiceAccount?
23. Why should you avoid sharing one ServiceAccount across multiple unrelated workloads?
24. What tool/command would you use to quickly check "can this ServiceAccount delete Secrets"?
25. What's the security risk of `resources: ["*"], verbs: ["*"]` in a Role, even scoped to one namespace?
26. How would you migrate a workload from static cloud access keys stored as Kubernetes Secrets to Workload Identity?
27. What KUBERNETES_SERVICE_HOST/PORT env vars are auto-injected into every Pod, and what are they for?
28. Why is namespace correctness in a RoleBinding's `subjects` field a common source of silent RBAC failures?
29. What's the practical difference in blast radius between a compromised Pod with no ServiceAccount token mounted vs. one with a `cluster-admin`-bound token?
30. Design a least-privilege ServiceAccount + RBAC setup for a CI/CD Pod that needs to create Jobs and read their logs, but nothing else.

---

That completes the ServiceAccounts chapter. Ready to continue to the next batch whenever you'd like — Services & Networking, ConfigMaps/Secrets, Ingress, or Network Policies would all logically follow in the security/identity thread we're now in.

# Kubernetes Troubleshooting & Debugging — Complete Masterclass (Beginner → Advanced SRE Level)

---

## PART 1 — THINKING LIKE AN SRE: THE METHODOLOGY BEFORE THE COMMANDS

Before any command, internalize this: **Kubernetes troubleshooting is not "run kubectl commands until something looks wrong."** It's a disciplined narrowing process through a stack of layers, each of which can fail independently, and each of which has a distinct signature of failure. An SRE's edge isn't knowing more commands — it's knowing **which layer to suspect first, given the symptom**, and how to gather evidence that either confirms or eliminates a hypothesis, fast.

### The Kubernetes Failure Stack (memorize this order)

```
1. Application layer     (code bug, bad config, unhandled exception)
2. Container layer        (image issue, entrypoint, resource limits)
3. Pod layer               (scheduling, probes, restarts)
4. Controller layer        (Deployment/ReplicaSet/StatefulSet reconciliation)
5. Node layer               (capacity, health, kubelet)
6. Cluster networking layer  (Service, DNS, CNI, Ingress)
7. Storage layer              (PV/PVC, CSI driver, mount failures)
8. Control plane layer         (API server, etcd, scheduler, controller-manager)
9. Identity/RBAC layer           (authentication, authorization)
```

A symptom like "my app is returning 500s" could originate at **any** of these 9 layers. The methodology exists to stop you from randomly guessing.

### The Decision Tree (the core loop)

```
                    ┌─────────────┐
                    │   Problem    │  (symptom reported: "site is down",
                    └──────┬──────┘   "pod won't start", "slow responses")
                           │
                           ▼
              ┌─────────────────────────┐
              │  1. IDENTIFY THE LAYER    │
              │  Is it Pod? Node? Network? │
              │  Storage? RBAC? Scaling?    │
              └───────────┬─────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │  2. GATHER EVIDENCE       │
              │  get → describe → logs →   │
              │  events → exec/debug        │
              └───────────┬─────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │  3. FORM & TEST A          │
              │     HYPOTHESIS              │
              │  "I think X because Y"      │
              │  Confirm with more evidence  │
              └───────────┬─────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │  4. FIND ROOT CAUSE        │
              │  Not just the symptom —     │
              │  WHY did it happen?          │
              └───────────┬─────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │  5. FIX                    │
              │  Smallest safe change that   │
              │  addresses ROOT CAUSE         │
              └───────────┬─────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │  6. VERIFY                 │
              │  Confirm symptom is gone AND │
              │  no new symptom introduced    │
              └───────────┬─────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │  7. PREVENT RECURRENCE     │
              │  Alerting, guardrails,       │
              │  postmortem, docs              │
              └─────────────────────────┘
```

**The SRE discipline is step 1 and step 3.** Junior engineers jump straight to `kubectl logs` on whatever Pod they can find. Senior engineers ask "what layer does this symptom actually point to?" *first* — because that determines which command sequence is even relevant.

### The Symptom → Layer Mapping (build this intuition)

| Symptom | Most likely layer(s) to check first |
|---|---|
| Pod stuck `Pending` | Scheduling, Node capacity, PVC binding |
| Pod `CrashLoopBackOff` | Application/container layer |
| Pod `Running` but not receiving traffic | Readiness probe, Service selector, Endpoints |
| "Service unreachable" from another Pod | Service, DNS, NetworkPolicy, CNI |
| "Works via port-forward, not via Service" | Service selector/labels mismatch |
| Slow responses, no errors | Resource throttling (CPU limits), autoscaling lag |
| Intermittent 502/503 from Ingress | Backend readiness flapping, Ingress controller config |
| Data missing after Pod restart | Storage (emptyDir vs PVC misunderstanding) |
| "403 Forbidden" calling K8s API from a Pod | RBAC |
| Deployment stuck mid-rollout | New ReplicaSet's Pods failing readiness/scheduling |
| Everything on one node is broken | Node health, kubelet, node-level resource pressure |
| Cluster-wide chaos, multiple layers failing at once | Control plane (API server/etcd) health |

---

## PART 2 — CORE COMMANDS, TAUGHT DEEPLY

### `kubectl get`

**What it does:** Lists objects and their current high-level status — the fastest "is anything obviously wrong" scan.

**When to use it:** Always your **first** command in any investigation — establishes the blast radius (one Pod? all Pods? one node? cluster-wide?).

**Useful options:**
```bash
kubectl get pods -o wide              # + node, IP — critical for spotting node-correlated failures
kubectl get pods --show-labels        # verify labels match what a Service/selector expects
kubectl get pods -w                    # watch in real time — essential for catching flapping/transient issues
kubectl get pods --field-selector=status.phase=Pending
kubectl get pods --sort-by=.status.containerStatuses[0].restartCount   # find the worst offenders fast
kubectl get all -n <namespace>         # broad namespace sweep
kubectl get pods -A                    # cluster-wide — is this isolated or everywhere?
```

**Example:**
```bash
kubectl get pods -o wide -n prod
```
```
NAME                READY   STATUS             RESTARTS   AGE   IP           NODE
api-7d9f8-x2k9p     0/1     CrashLoopBackOff   7          12m   10.244.1.5   node-a
api-7d9f8-m4j2q     1/1     Running            0          12m   10.244.2.9   node-b
api-7d9f8-p8n1r     0/1     Pending            0          2m    <none>       <none>
```

**Interpretation:** Three Pods, three different states — this alone tells you: (1) one Pod is genuinely crashing (app-layer problem, not node-wide), (2) one Pod is healthy (rules out "the image is universally broken"), (3) one Pod can't even schedule (separate investigation — scheduling/capacity, not related to the crash). **This single command already split what could look like "one problem" into three independent problems** — that's the value of `get` as a first move.

### `kubectl describe`

**What it does:** Shows the full object spec, current status, AND — critically — the **Events** section, a chronological log of everything the control plane has done/attempted/failed regarding this object.

**When to use it:** Immediately after `get` flags something as unhealthy. This is usually where 80% of root causes are visible directly.

**Useful options:**
```bash
kubectl describe pod <name>
kubectl describe node <name>
kubectl describe deployment <name>
kubectl describe pvc <name>
kubectl describe service <name>
```

**Example:**
```bash
kubectl describe pod api-7d9f8-p8n1r
```
```
...
Status:       Pending
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  2m    default-scheduler  0/3 nodes are available:
                                                        3 Insufficient cpu.
```

**Interpretation:** The **Events** section directly names the cause — insufficient CPU across all nodes. No log-diving needed. This is why `describe` (not `logs`) is almost always the **second** command in any Pod investigation, before you even think about container logs.

### `kubectl logs` / `kubectl logs --previous`

**What it does:** Streams stdout/stderr from a container. `--previous` retrieves logs from the **last terminated instance** of the container — essential for crash investigation, since a freshly-restarted container's *current* logs won't show why the *last* one died.

**When to use it:** Once `describe` tells you the container is actually starting and crashing (vs. never starting at all — in which case logs will be empty or unavailable, and `describe`'s Events are more informative).

**Useful options:**
```bash
kubectl logs <pod>                        # current container
kubectl logs <pod> -c <container>          # multi-container Pod — MUST specify which
kubectl logs <pod> --previous              # last crashed instance — THE crash debugging command
kubectl logs <pod> -f                       # follow/stream live
kubectl logs <pod> --since=10m               # recent window only, avoid noise
kubectl logs <pod> --tail=100                 # last N lines
kubectl logs -l app=myapp --all-containers     # aggregate across all matching Pods
```

**Example:**
```bash
kubectl logs api-7d9f8-x2k9p --previous
```
```
Traceback (most recent call last):
  ...
ConnectionRefusedError: [Errno 111] Connection refused: db-service:5432
```

**Interpretation:** The app is crashing because it can't reach its database dependency at startup — this reframes the whole investigation from "why is my app broken" to "why can't this Pod reach `db-service`," which is now a **Service/DNS/networking layer** question, not an application bug.

### `kubectl exec`

**What it does:** Runs a command inside a **running** container — your window into "what does the container actually see," as opposed to what Kubernetes *thinks* is true.

**When to use it:** When you need to verify something from **inside** the container's perspective — is a config file actually present? Can it resolve DNS? Does the mounted volume have the expected content?

**Useful options:**
```bash
kubectl exec -it <pod> -- /bin/sh
kubectl exec -it <pod> -c <container> -- bash
kubectl exec <pod> -- env                          # verify injected env vars
kubectl exec <pod> -- cat /etc/resolv.conf           # DNS config as the container sees it
kubectl exec <pod> -- nslookup db-service             # does DNS resolution work from inside?
kubectl exec <pod> -- curl -v http://db-service:5432    # can it actually connect?
```

**Example & interpretation:**
```bash
kubectl exec api-7d9f8-m4j2q -- nslookup db-service
```
```
** server can't find db-service: NXDOMAIN
```
This single result eliminates the app entirely as the culprit and **confirms** the hypothesis from the logs — DNS resolution for `db-service` is failing. Now you know to investigate the Service object and CoreDNS, not the application code.

### `kubectl run`

**What it does:** Quickly creates a throwaway Pod — invaluable for testing connectivity/behavior **from the cluster's perspective**, independent of any existing broken workload.

**When to use it:** To isolate "is this a cluster-wide networking/DNS issue, or specific to my broken Pod?"

**Useful options:**
```bash
kubectl run debug --image=busybox --rm -it --restart=Never -- sh
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl -v http://myservice
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
```

**Example & interpretation:**
```bash
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup db-service
```
If this **succeeds** from a brand-new Pod but failed from `api-7d9f8-m4j2q`, the problem is **specific to that Pod/node** (e.g., a NetworkPolicy scoped to certain labels, or a node-specific CoreDNS connectivity issue) — not a cluster-wide DNS outage. This is a critical differential diagnosis technique: **use a clean control Pod to isolate whether the problem is systemic or localized.**

### `kubectl debug`

**What it does:** Attaches an **ephemeral container** to a running Pod (or creates a copy, or debugs a Node directly) — for cases where the target container has no shell/debugging tools (common with minimal/distroless production images).

**When to use it:** When `kubectl exec` fails because there's no shell in the image, or you need heavier debugging tools than the app image contains, or you need to debug a Node itself.

**Useful options:**
```bash
kubectl debug -it <pod> --image=busybox --target=<container>
# adds a debug container sharing the target's process namespace — can see its processes,
# inspect /proc/<pid>/root for its filesystem, etc.

kubectl debug node/<nodename> -it --image=busybox
# creates a privileged Pod on that node with the node's filesystem mounted at /host —
# for debugging the NODE itself, not a specific Pod

kubectl debug <pod> -it --copy-to=<pod>-debug --container=<container> -- sh
# makes a COPY of the Pod (so you don't disturb the live one) with a shell added
```

**Example:**
```bash
kubectl debug -it api-7d9f8-m4j2q --image=nicolaka/netshoot --target=app
```
This attaches a full networking-toolkit container (`netshoot` — has `dig`, `tcpdump`, `curl`, `netstat`, etc.) sharing the same network namespace as `app`, letting you run real diagnostic tools against a Pod whose own image has none.

### `kubectl events` (or `kubectl get events`)

**What it does:** Shows the cluster-wide (or namespace-scoped) event stream — a chronological record of everything the control plane observed and acted on, across **all** objects, not just one Pod.

**When to use it:** When you don't yet know **which** object is the problem — a broad sweep to spot anomalies (repeated warnings, scheduling failures, eviction events) across the whole namespace/cluster.

**Useful options:**
```bash
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events -A --sort-by=.metadata.creationTimestamp | tail -30
kubectl get events --field-selector type=Warning
kubectl get events --field-selector involvedObject.name=<pod-name>
kubectl alpha events                    # newer structured event viewer (where available)
```

**Example & interpretation:**
```bash
kubectl get events -n prod --field-selector type=Warning --sort-by=.metadata.creationTimestamp
```
```
2m    Warning   Evicted            pod/cache-3        The node had condition: [MemoryPressure]
2m    Warning   FailedScheduling   pod/api-p8n1r        0/3 nodes are available: 3 Insufficient cpu
1m    Warning   BackOff             pod/worker-9x2       Back-off restarting failed container
```

**Interpretation:** Seeing `MemoryPressure` eviction **and** `Insufficient cpu` scheduling failures **together**, close in time, points strongly toward a **node capacity crisis**, not three unrelated problems — this correlation is only visible by scanning events broadly, not by looking at one Pod in isolation.

### `kubectl top`

**What it does:** Shows real-time CPU/memory usage (requires `metrics-server`) — actual consumption, not requests/limits from spec.

**When to use it:** To check if a resource problem is about actual usage exceeding capacity, vs. a scheduling/config problem unrelated to real load.

**Useful options:**
```bash
kubectl top nodes                        # per-node CPU/memory usage vs allocatable
kubectl top pods                          # per-Pod usage
kubectl top pods --containers               # per-container breakdown within multi-container Pods
kubectl top pods --sort-by=memory            # find the memory hogs fast
```

**Example:**
```bash
kubectl top nodes
```
```
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-a   1900m        95%    3200Mi          82%
node-b   400m         20%    1100Mi          28%
```

**Interpretation:** `node-a` is under real CPU pressure while `node-b` has headroom — this explains why new Pods might fail to schedule on `node-a` specifically, and hints the fix might be better pod anti-affinity/spread, not necessarily "add more nodes."

### `kubectl explain`

**What it does:** Shows the **schema documentation** for any API resource/field, straight from the cluster's OpenAPI spec — the fastest way to check exact field names, types, and nesting without leaving the terminal.

**When to use it:** When writing/debugging YAML and you're unsure of exact field names, required vs optional fields, or valid enum values.

**Useful options:**
```bash
kubectl explain pod.spec.containers.resources
kubectl explain deployment.spec.strategy.rollingUpdate
kubectl explain pod.spec --recursive          # full nested schema tree
kubectl explain hpa.spec.metrics --api-version=autoscaling/v2
```

**Example:**
```bash
kubectl explain pod.spec.containers.livenessProbe
```
```
FIELDS:
   exec         <Object>
   failureThreshold    <integer>
   httpGet       <Object>
   initialDelaySeconds  <integer>
   periodSeconds       <integer>
   ...
```
**Interpretation:** Confirms exact field names and types before you write/patch YAML — avoids the classic mistake of guessing a field name (e.g., `retries` instead of `failureThreshold`) and having it silently ignored by schema validation-adjacent defaults.

### `kubectl auth can-i`

**What it does:** Directly answers "is this identity authorized to do this action on this resource," without needing to actually attempt the action and parse a 403.

**When to use it:** Any time you suspect an RBAC problem — either your own kubectl access, or (critically) a ServiceAccount's access from inside a Pod.

**Useful options:**
```bash
kubectl auth can-i list pods
kubectl auth can-i delete secrets -n prod
kubectl auth can-i '*' '*'                                       # am I cluster-admin?
kubectl auth can-i list pods --as=system:serviceaccount:prod:myapp-sa
kubectl auth can-i --list --as=system:serviceaccount:prod:myapp-sa    # full permission dump
```

**Example & interpretation:**
```bash
kubectl auth can-i create jobs --as=system:serviceaccount:ci:ci-runner -n prod
```
```
no
```
Immediately confirms/denies an RBAC hypothesis without needing to reproduce the actual failing API call from inside a Pod — a massive time-saver during a live incident.

---

## PART 3 — LAYER-BY-LAYER INVESTIGATION PLAYBOOKS

### 1. Pods

```
get pods -o wide  →  describe pod  →  logs [--previous]  →  exec/debug
```
Look for: `STATUS` (Pending/CrashLoopBackOff/ImagePullBackOff/Error), `RESTARTS` count, `READY` ratio (`0/1` = not passing readiness), Events section causes.

### 2. Deployments

```
get deployment  →  describe deployment (Conditions, Events)  →  get rs -l <selector>  →  kubectl rollout status
```
Look for: `Progressing=False` condition, mismatched `DESIRED`/`CURRENT`/`AVAILABLE` replica counts, stuck rollout (new RS's Pods failing).

### 3. ReplicaSets

```
get rs -l app=<name>  →  describe rs  →  compare replica counts old vs new RS
```
Look for: is the **old** RS still holding replicas it shouldn't (rollout stuck)? Is a RS's `Events` showing repeated Pod creation failures?

### 4. Nodes

```
get nodes  →  describe node (Conditions, Allocated resources, Events)  →  kubectl top nodes
```
Look for: `Ready` condition status, `MemoryPressure`/`DiskPressure`/`PIDPressure` conditions, taints, `Allocated resources` section showing overcommitment.

### 5. Services

```
get svc  →  describe svc (check Endpoints!)  →  get endpoints <svc-name>
```
Look for: **empty Endpoints** — the #1 Service problem, almost always a **label selector mismatch** between the Service and the Pods.

```bash
kubectl get svc myapp -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels | grep <expected-labels>
```

### 6. Ingress

```
get ingress  →  describe ingress  →  check backend Service + Endpoints  →  check Ingress controller logs
```
Look for: `Backend` shown in describe output matching an actual healthy Service, TLS secret existing and valid, Ingress controller Pod logs for routing/cert errors.

### 7. DNS

```
exec into a Pod → nslookup/dig the target → check CoreDNS Pods are Running → check CoreDNS logs
```
```bash
kubectl exec -it <pod> -- nslookup kubernetes.default    # sanity check: does basic cluster DNS even work?
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### 8. Networking (Pod-to-Pod, NetworkPolicy)

```
kubectl run test Pods on both ends → curl/nc between them → check NetworkPolicy objects → check CNI Pod health
```
```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <name>     # verify podSelector/namespaceSelector match intent
kubectl get pods -n kube-system -l k8s-app=calico-node   # (or your CNI) — are agents healthy on every node?
```

### 9. Storage

```
get pvc → describe pvc (check Bound status + Events) → describe pod (FailedMount events) → check StorageClass
```
```bash
kubectl get pvc
kubectl describe pvc <name>       # Status: Pending? check Events for provisioning failures
kubectl get storageclass
kubectl get pv
```

### 10. ConfigMaps

```
get configmap → describe/get -o yaml → verify Pod's envFrom/volumeMounts reference the RIGHT name → exec + check actual values
```
```bash
kubectl exec <pod> -- env | grep <expected-var>
kubectl exec <pod> -- cat /path/to/mounted/config      # for volume-mounted ConfigMaps
```
Remember: ConfigMap data changes don't propagate to already-running Pod env vars — always check if a **stale Pod** is the real issue.

### 11. Secrets

```
get secret → describe secret (type, keys — NOT values) → verify reference name/key match in Pod spec
```
```bash
kubectl get secret <name> -o jsonpath='{.data}' | jq 'keys'   # see key NAMES without decoding values
kubectl exec <pod> -- env | grep DB_PASSWORD     # confirm it actually landed (in a safe, controlled context)
```

### 12. RBAC

```
kubectl auth can-i (as the relevant identity) → describe rolebinding/clusterrolebinding → check subjects namespace/name match exactly
```
```bash
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>
kubectl get rolebinding,clusterrolebinding -A -o wide | grep <sa-name>
```

### 13. Resource Problems

```
kubectl top pods/nodes → describe pod (Last State: OOMKilled?) → describe node (Allocated resources)
```
```bash
kubectl describe pod <name> | grep -A5 "Last State"
kubectl describe node <name> | grep -A10 "Allocated resources"
```

### 14. Scheduling

```
describe pod (FailedScheduling event, exact reason) → get nodes -o wide → check taints/affinity/resource fit
```
```bash
kubectl describe pod <name> | grep -A5 Events
kubectl describe nodes | grep Taints
kubectl get pod <name> -o yaml | grep -A10 affinity
```

### 15. Autoscaling

```
describe hpa (current/target metrics, Events) → kubectl top pods → check metrics-server/adapter health → check pending Pods for CA
```
```bash
kubectl describe hpa <name>
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl get pods --field-selector=status.phase=Pending
kubectl logs -n kube-system deploy/cluster-autoscaler | tail -50
```

---

## PART 4 — REAL INCIDENT-STYLE EXERCISES

### Incident 1: "Checkout service returning intermittent 502s"

**Symptom reported:** Users occasionally see 502 errors on checkout, roughly 1 in 10 requests, started 20 minutes ago.

**Think like an SRE — don't jump to logs. Start at layer identification:**

```bash
kubectl get pods -l app=checkout -o wide
```
```
NAME               READY   STATUS    RESTARTS   AGE   NODE
checkout-8f7-a1     1/1    Running   0          45m   node-a
checkout-8f7-b2     0/1    Running   0          18m   node-b
checkout-8f7-c3     1/1    Running   0          45m   node-c
```

**Observation:** One Pod (`b2`) is `Running` but `0/1` Ready — **exactly** matching the "intermittent" nature of the symptom (traffic sometimes routes there via the Service, sometimes to a healthy Pod).

```bash
kubectl describe pod checkout-8f7-b2
```
```
Readiness probe failed: HTTP probe failed with statuscode: 503
```

**Hypothesis:** This Pod's readiness probe is failing, but it's still receiving *some* traffic — wait, that shouldn't happen if readiness gates Endpoints correctly. Check:
```bash
kubectl get endpoints checkout
```
If `b2`'s IP **is** in the Endpoints list despite failing readiness, that's a real anomaly (stale Endpoint, propagation delay) — but more likely, the probe is **flapping** (intermittently passing/failing), which explains "intermittent" perfectly.

```bash
kubectl logs checkout-8f7-b2 --previous --since=20m
```
Reveals: the app occasionally times out on a slow downstream dependency call during its `/readyz` check, briefly failing readiness, then recovering — explaining the flap.

**Root cause:** readiness probe logic is coupled to a flaky downstream dependency check, causing the Pod to bounce in/out of the Service's Endpoints.

**Fix:** decouple `/readyz` from that downstream call, or add retry/timeout tuning to the health check logic itself; short-term mitigation: increase `failureThreshold` to tolerate brief blips without full removal from Endpoints.

**Verify:** watch `kubectl get endpoints checkout -w` — confirm the IP stays present.

**Prevent recurrence:** alert on readiness flapping (`restartCount` doesn't catch this — need a probe-failure-rate metric), document the health-check design principle ("readiness should reflect this Pod's own ability to serve, not blindly chain to dependency health").

---

### Incident 2: "New feature deploy — Pods stuck Pending, rollout frozen"

```bash
kubectl rollout status deployment/api
# Waiting for deployment "api" rollout to finish: 2 out of 5 new replicas have been updated...
kubectl get pods -l app=api -o wide
```
```
api-new-x1   0/1   Pending   0   5m   <none>   <none>
api-new-y2   0/1   Pending   0   5m   <none>   <none>
api-old-a1   1/1   Running   0   2h   node-a
api-old-b2   1/1   Running   0   2h   node-b
api-old-c3   1/1   Running   0   2h   node-c
```

```bash
kubectl describe pod api-new-x1
```
```
Warning  FailedScheduling  0/3 nodes are available: 3 Insufficient memory.
```

**Hypothesis:** new Pods need more memory than the old ones — check if the deploy changed `resources.requests`.

```bash
kubectl get deployment api -o jsonpath='{.spec.template.spec.containers[0].resources}'
```
Reveals `requests.memory` was bumped from `256Mi` to `1Gi` in this release — but the cluster's nodes don't have that much spare capacity while `maxSurge` Pods need to coexist with the old ones.

**Root cause:** resource request increase wasn't validated against actual cluster capacity before deploy; `RollingUpdate`'s `maxSurge` behavior requires **temporary extra headroom** the cluster doesn't have.

**Fix (immediate):** `kubectl rollout undo deployment/api` to unblock the incident. **Fix (real):** either provision more node capacity, right-size the new memory request based on actual observed usage (check with VPA recommender or historical `kubectl top`), or temporarily set `maxSurge: 0, maxUnavailable: 1` to avoid needing extra headroom during the rollout.

**Verify:** re-attempt rollout, watch `kubectl get pods -w` for successful scheduling.

**Prevent recurrence:** add a CI/CD gate that diffs resource requests and flags significant increases for capacity review before deploy; monitor cluster-wide allocatable headroom as a standing metric.

---

### Incident 3: "Pod can't reach an internal API — works from my laptop"

```bash
kubectl exec -it worker-abc -- curl -v http://internal-api:8080/health
# curl: (6) Could not resolve host: internal-api
```

```bash
kubectl exec -it worker-abc -- nslookup kubernetes.default
```
Succeeds — basic cluster DNS works, so this isn't a total CoreDNS outage.

```bash
kubectl get svc internal-api -n prod
```
```
Error from server (NotFound): services "internal-api" not found
```

**Root cause found immediately:** the Service simply doesn't exist in this namespace — likely the app team pointed to a bare hostname assuming same-namespace resolution, but the Service actually lives in a different namespace.

```bash
kubectl get svc -A | grep internal-api
```
```
backend   internal-api   ClusterIP   10.96.44.2   8080/TCP
```

**Fix:** correct the app config to use the fully-qualified DNS name `internal-api.backend.svc.cluster.local`, or create matching Service objects per-namespace if genuinely needed in both.

**Verify:** re-run the `curl` test from inside the Pod.

**Prevent recurrence:** document namespace-crossing service references explicitly, consider a naming/discovery convention across teams.

---

## PART 5 — TROUBLESHOOTING CHECKLIST (complete)

```
□ Identify blast radius: one Pod? one node? one namespace? cluster-wide?
□ kubectl get -o wide on the relevant object type — status, node, IP at a glance
□ kubectl describe on the unhealthy object — READ THE EVENTS SECTION FIRST
□ kubectl get events --sort-by=.metadata.creationTimestamp for broader correlation
□ kubectl logs [--previous] if container-level investigation is warranted
□ kubectl exec / kubectl debug to verify from the container's own perspective
□ kubectl top nodes/pods if resource pressure is suspected
□ kubectl auth can-i if RBAC/permission errors are involved
□ kubectl get endpoints if Service connectivity is the symptom
□ Isolate with kubectl run — is this Pod-specific or cluster-wide?
□ Form an explicit hypothesis ("I believe X because Y") before acting
□ Fix the ROOT CAUSE, not just the symptom (avoid "just restart it" without understanding why)
□ Verify the fix — confirm symptom gone AND no new symptom introduced
□ Add monitoring/alerting to catch this class of failure earlier next time
□ Document in a postmortem — what layer failed, why, how it was caught, how to prevent it
```

---

## PART 6 — INTERVIEW QUESTIONS

1. Describe your systematic approach to troubleshooting an unknown Kubernetes issue, from first symptom to resolution.
2. Why is `kubectl describe` often more valuable than `kubectl logs` as a first diagnostic step?
3. What's the difference between `kubectl logs` and `kubectl logs --previous`, and when is each appropriate?
4. How would you determine if a networking issue is Pod-specific or cluster-wide?
5. What does an empty `Endpoints` object for a Service almost always indicate?
6. How do you distinguish a scheduling problem from a container crash problem, just from `kubectl get pods` output?
7. What's your first command when investigating "Pod stuck Pending," and what are you looking for in its output?
8. How would you debug a Pod running a distroless image with no shell?
9. What's the difference between `kubectl exec` and `kubectl debug`, and when would you choose one over the other?
10. How do you check whether a ServiceAccount has permission to perform a specific action, without triggering an actual failing request?
11. Why might `kubectl logs` show nothing useful even though a Pod is clearly failing?
12. Explain the difference between a Pod's `phase` and its `Ready` condition, and why both matter during troubleshooting.
13. What does `OOMKilled` in a Pod's `Last State` tell you, and what's your next diagnostic step?
14. How would you determine if a Deployment rollout is stuck due to resource constraints vs. a bad readiness probe?
15. What's the significance of correlated events (e.g., MemoryPressure + FailedScheduling) appearing together in `kubectl get events`?
16. How would you troubleshoot "works via port-forward but not via the Service"?
17. What's your approach when DNS resolution fails for one specific Pod but works from others?
18. How would you verify a ConfigMap's changes have actually reached a running Pod?
19. What's the danger of just restarting a crashing Pod without investigating `--previous` logs first?
20. How would you approach a "cluster-wide chaos, everything is broken" scenario differently from a single-Pod issue?
21. What's the role of `kubectl top` in a resource-related investigation, and what does it NOT tell you (vs. `requests`/`limits`)?
22. How would you use `kubectl run` as an isolation/control technique during an investigation?
23. What's the value of `kubectl explain` during active incident YAML editing, versus during calm development?
24. Describe a real (or realistic) incident where the root cause was NOT where the initial symptom pointed.
25. What does "fix the root cause, not the symptom" mean in practice — give a concrete example from Kubernetes troubleshooting.
26. How would you design monitoring/alerting to catch the class of failure in a given incident before it recurs?
27. What's your process for writing a blameless postmortem after a Kubernetes production incident?
28. Walk through diagnosing an Ingress returning 502s when the backend Service appears healthy.
29. What's the difference in diagnostic approach between a node-level problem and an application-level problem, when the symptom looks identical to users?
30. How do you prioritize which of the 9 failure-stack layers to investigate first, given limited time during an active incident?

---

That completes the Troubleshooting & Debugging chapter. Ready for the next batch in the masterclass sequence whenever you'd like to continue.


# Kubernetes # Kubernetes Pod Troubleshooting — Complete Masterclass
## Part 1 of 4: CrashLoopBackOff, ImagePullBackOff, ErrImagePull, Pending, OOMKilled

*(This masterclass covers 15 failure modes at full depth — delivered in installments so each gets the complete 12-point treatment rather than being compressed. Part 1 covers the five most frequently encountered failures. Say "continue" for Part 2: CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted.)*

---

# THE UNIVERSAL TROUBLESHOOTING ENTRY POINT

Before diving into individual failures, every single one of them starts the same way:
```bash
kubectl get pods                    # spot the abnormal STATUS column
kubectl describe pod <pod>           # Events section — almost always names the exact problem
kubectl logs <pod>                    # what the CONTAINER itself said before dying
kubectl logs <pod> --previous          # logs from the PREVIOUS crashed instance (crucial for CrashLoopBackOff)
```
Every section below assumes you've already run these four commands — they're the universal first move, not repeated as "step 1" for every failure type below.

---

# 1. CrashLoopBackOff

## What it means
The container **starts, then exits** (crashes, or even exits cleanly with code 0 when it shouldn't), repeatedly — and Kubernetes is now waiting increasingly long **backoff** periods between each restart attempt rather than restarting instantly forever.

## What Kubernetes is doing internally
This is a **kubelet-local** behavior (Pods masterclass, Diagram D1) — no scheduler, no controller-manager involvement. The kubelet observes the container exit, and per `restartPolicy: Always` (the Deployment/ReplicaSet default), restarts it — but with **exponential backoff**: 10s, 20s, 40s, 80s... capped at 5 minutes between attempts, specifically to avoid hammering a fundamentally broken container in a tight, wasteful restart loop.
```
Container exits → kubelet waits (backoff) → restarts → exits again → wait LONGER → restart → ...
```

## Common causes
- Application crashes immediately on startup (missing config, unhandled exception, failed dependency connection)
- Wrong `command`/`args` — e.g., a command that runs once and exits, used on a Pod expecting a long-running process
- A missing environment variable or file the app requires to even initialize
- The main process legitimately finishing (exit 0) in a container Kubernetes expects to run forever

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     CrashLoopBackOff   7          12m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl logs my-app                 # current attempt (often empty if it crashed instantly)
kubectl logs my-app --previous       # THE crash's actual output — usually the real answer
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      ...
  Finished:     ...
Restart Count:  7
```
**Exit code matters a lot here:**
- `0` — the process exited cleanly, meaning your restartPolicy expectations and the app's actual behavior disagree (app thinks it's done; Kubernetes expects it to run forever)
- `1` (or other nonzero) — generic application error; the actual reason is almost always in `logs --previous`
- `137` — this is actually `OOMKilled` wearing a CrashLoopBackOff's clothes (§5) — always check `Reason` explicitly, not just the exit code

## Root-cause investigation
```bash
kubectl logs my-app --previous --timestamps
kubectl describe pod my-app | grep -A10 "Last State"
# reproduce locally if possible:
docker run --rm <same-image> <same-command>
```
The single highest-leverage move: **read `logs --previous` before doing anything else** — the overwhelming majority of CrashLoopBackOff cases are fully explained by the application's own final log lines before it died.

## Fixes
- Fix the actual application bug/misconfiguration revealed by the logs
- If it's a legitimately run-to-completion process, use a **Job** (Jobs guide), not a Deployment
- Add a `startupProbe` (§15) if the app just needs more time before health checks should even begin evaluating it

## Verification
```bash
kubectl get pod my-app -w
# RESTARTS stops climbing, STATUS settles to Running, READY reaches 1/1
```

## Prevention
- Fail fast and log clearly on startup — a silent crash with no log output is far harder to diagnose than one that prints exactly what config was missing
- Use `startupProbe` for slow-initializing apps so the kubelet doesn't prematurely judge them unhealthy during legitimate startup time

## Production example
A Node.js app crash-looped in production because a required environment variable (`DATABASE_URL`) was renamed in a ConfigMap update but the Deployment's `env` reference still pointed at the old key name — `logs --previous` showed `TypeError: Cannot read property 'connect' of undefined` within milliseconds of each restart, immediately pointing at the missing config rather than a code bug.

## Interview question
**Q: A Pod is in CrashLoopBackOff with exit code 0. Why is that unusual, and what does it suggest?**
A: Exit code 0 means the process believed it finished successfully — but `restartPolicy: Always` (typical under a Deployment) treats *any* exit as something to restart. This mismatch suggests either the workload should actually be a Job (finite, run-to-completion) rather than a Deployment, or the application has a bug causing it to terminate early when it should keep running.

---

# 2–3. ImagePullBackOff and ErrImagePull

## What they mean
`ErrImagePull` is the **first** failed attempt to pull a container image. `ImagePullBackOff` is what you see on **subsequent** attempts, once the kubelet starts backing off between retries — the same exponential-backoff relationship as CrashLoopBackOff, just applied to image pulls instead of container starts.

## What Kubernetes is doing internally
```
kubelet sees a Pod assigned to its node
    → calls the container runtime (via CRI) to pull the specified image
    → runtime attempts to contact the registry, authenticate, download layers
    → FAILS → kubelet reports ErrImagePull, schedules a retry with backoff
    → next failure → reports ImagePullBackOff, backoff continues to grow
```

## Common causes
- Typo in the image name or tag (`myapp:1.O` instead of `1.0` — a classic letter-O-vs-zero mistake)
- Image genuinely doesn't exist in the registry, or was deleted
- Private registry requiring authentication, with no (or wrong) `imagePullSecrets` configured
- Rate limiting from the registry (Docker Hub's anonymous pull limits are a very common real-world trigger)
- Network policy or firewall blocking egress from nodes to the registry

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     ImagePullBackOff   0          2m
```

## Commands to run
```bash
kubectl describe pod my-app
```

## How to interpret the output
```
Events:
  Warning  Failed     kubelet  Failed to pull image "myapp:1.O":
                                rpc error: code = NotFound desc = failed to pull and
                                unpack image "docker.io/library/myapp:1.O":
                                failed to resolve reference: myapp:1.O: not found
  Warning  Failed     kubelet  Error: ErrImagePull
  Normal   BackOff    kubelet  Back-off pulling image "myapp:1.O"
  Warning  Failed     kubelet  Error: ImagePullBackOff
```
The **exact error text after "Failed to pull image"** tells you precisely which failure mode you're in — `not found` (bad name/tag), `unauthorized`/`403` (auth problem), `connection refused`/`timeout` (network problem) all point in genuinely different directions.

## Root-cause investigation
```bash
# verify the image actually exists and is spelled correctly:
docker pull myapp:1.0          # try it manually, from a machine with equivalent network access
kubectl get pod my-app -o jsonpath='{.spec.containers[0].image}'   # confirm exactly what was requested
kubectl get pod my-app -o jsonpath='{.spec.imagePullSecrets}'       # confirm a pull secret is even attached
```

## Fixes
```bash
# for private registries:
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io --docker-username=me --docker-password=secret
```
```yaml
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: myregistry.io/myapp:1.0    # fix any typo here
```

## Verification
```bash
kubectl delete pod my-app        # if managed by a controller, a fresh Pod will retry immediately
kubectl get pod -w
```

## Prevention
- Pin exact image tags (never `:latest` in production — Pods masterclass-adjacent best practice) and verify them in CI before deployment
- Test `imagePullSecrets` in a staging namespace before relying on them in production
- Consider a pull-through cache/mirror to avoid public registry rate limits entirely

## Production example
A cluster's nodes suddenly couldn't pull any Docker Hub images after Docker Hub introduced anonymous pull rate limits — every new Pod scheduled that day hit `ErrImagePull` with a `429 Too Many Requests` message buried in the Events, resolved by adding authenticated `imagePullSecrets` (which have a much higher rate limit) cluster-wide via a default ServiceAccount patch.

## Interview question
**Q: What's the actual difference between ErrImagePull and ImagePullBackOff?**
A: They're the same underlying failure — ErrImagePull is the immediate, first-attempt failure state; ImagePullBackOff is what's reported once the kubelet starts throttling retry attempts with exponential backoff after repeated failures. Neither indicates a fundamentally different problem — they're two points on the same timeline.

---

# 4. Pending

## What it means
The Pod object exists in etcd, but **has not been scheduled to a node at all** (or, less commonly, is scheduled but can't start for some pre-container reason) — this is the Pod phase from the Pods masterclass §6–9, at its most literal.

## What Kubernetes is doing internally
```
Pod created → scheduler watches for Pods with empty .spec.nodeName
            → filtering phase eliminates unsuitable nodes
            → IF NO NODE SURVIVES FILTERING: Pod stays Pending, scheduler
              retries on backoff and whenever cluster state changes
              (Architecture masterclass, Part 4 §28)
```

## Common causes
- **Insufficient resources** cluster-wide for the Pod's requests (Resource Requests guide)
- **Taints without matching tolerations** — every node the Pod could otherwise fit on is tainted against it
- **Node selector/affinity rules** that no current node satisfies
- **No available PersistentVolume** matching a required PVC (Storage guide) — scheduling waits on binding
- Zero worker nodes exist yet (a brand-new cluster, or all nodes cordoned/draining)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS    RESTARTS   AGE
# my-app    0/1     Pending   0          5m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl get nodes
kubectl describe nodes | grep -A5 "Allocated resources"
```

## How to interpret the output
```
Events:
  Warning  FailedScheduling  0/5 nodes are available: 3 Insufficient cpu,
           2 node(s) had taint {dedicated: gpu}, that the pod didn't tolerate.
```
This message is **exhaustive and literal** — it tells you exactly how many nodes were rejected and for which specific reason, across every node in the cluster. Read it fully before guessing.

## Root-cause investigation
```bash
kubectl get pod my-app -o jsonpath='{.spec.tolerations}'
kubectl get pod my-app -o jsonpath='{.spec.nodeSelector}'
kubectl get pod my-app -o jsonpath='{.spec.containers[*].resources}'
kubectl describe nodes | grep -B2 Taints
```

## Fixes
- Reduce over-inflated resource requests, or add cluster capacity (more nodes, or trigger a cluster autoscaler)
- Add the required `tolerations` if the workload genuinely belongs on a tainted node pool
- Correct a `nodeSelector`/`affinity` rule that's impossible to satisfy (e.g., referencing a label that no node actually has, often a typo)

## Verification
```bash
kubectl get pod my-app -w
# STATUS moves from Pending to ContainerCreating to Running
```

## Prevention
- Right-size resource requests based on actual observed usage (Resource Requests guide) rather than guessing high "to be safe," which wastes schedulable capacity across the whole cluster
- Keep taints/tolerations and nodeSelectors documented and reviewed — these are easy to silently break during node pool migrations

## Production example
A team migrated from one node pool to another with different labels; their Deployment's `nodeSelector` still referenced the old pool's label, and every new Pod sat `Pending` indefinitely after the old pool was fully drained — `FailedScheduling`'s message explicitly listed "0/12 nodes match node selector," which immediately pointed at the stale selector.

## Interview question
**Q: Is a Pending Pod always a resource-capacity problem?**
A: No — resource insufficiency is one cause among several (taints/tolerations mismatches, node selector/affinity that no node satisfies, an unbound PVC, or literally zero nodes existing). The `FailedScheduling` event message always states the specific reason(s); assuming it's always "not enough CPU/memory" without reading the actual message is a common, avoidable diagnostic mistake.

---

# 5. OOMKilled

## What it means
*(Full internal mechanics already covered exhaustively in the Resource Limits guide, Part 3 — this section is the troubleshooting-flow-focused summary.)* A container exceeded its memory limit and was forcibly killed by the kernel's OOM killer, scoped to that container's own cgroup.

## What Kubernetes is doing internally
```
Container's memory usage → approaches cgroup memory.max
    → kernel reclaims what it can (silent, no kill yet)
    → still over → kernel OOM killer sends SIGKILL to the container's process
    → container exits with code 137 (128 + SIGKILL's signal number 9)
    → kubelet observes the exit, reports Reason: OOMKilled
    → restartPolicy governs what happens next (same restart mechanics as
      any container exit, per Pods masterclass Diagram D1)
```

## Common causes
- Memory limit set genuinely too low for real peak usage
- A memory leak — usage climbs steadily over the container's lifetime rather than stabilizing
- A sudden legitimate spike (large request payload, batch processing burst) exceeding a limit sized for steady-state only
- No limit set at all, but the **node itself** ran out of memory (node-level OOM, per the Resource Limits guide — more disruptive, can affect unrelated Pods)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS      RESTARTS   AGE
# my-app    0/1     OOMKilled   3          8m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl top pod my-app --containers      # if it's currently running, see live usage
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Started:      ...
  Finished:     ...
```
**Exit code 137 plus `Reason: OOMKilled` together are unambiguous** — this is never a coincidence or a generic crash; it's specifically the kernel's memory enforcement, distinct from every other failure in this guide.

## Root-cause investigation
```bash
kubectl describe pod my-app | grep -A3 "Limits\|Requests"
kubectl top pod my-app --containers        # compare against the limit, if still running long enough
# check historical usage if you have metrics retention (Prometheus, etc.) —
# was this a SLOW climb (leak) or a SUDDEN spike (load-driven)?
```

## Fixes
- Raise the memory limit to a realistic ceiling based on observed peak usage, with headroom
- Fix an actual memory leak if usage climbs without bound over time rather than stabilizing
- For genuinely bursty workloads, consider whether the request should be raised too (affects scheduling and QoS, per the Resource Requests/Limits guides), not just the limit

## Verification
```bash
kubectl get pod my-app -w
kubectl top pod my-app --containers -w
# usage should stabilize comfortably under the new limit, RESTARTS stops climbing
```

## Prevention
- Load-test with realistic peak traffic before setting production memory limits
- Alert on Pods approaching their memory limit (e.g., >80% sustained) **before** they get OOMKilled, not just after
- Use Guaranteed QoS (Resource Limits guide) for workloads where an OOMKill is especially costly/disruptive

## Production example
A Java application was OOMKilled repeatedly after a Kubernetes migration because its JVM heap flags were set assuming the *node's* total memory, not the container's cgroup limit — the JVM allocated far more heap than the container was actually permitted, guaranteeing an eventual OOMKill under any real load; the fix was explicit `-Xmx` flags sized against the container's own memory limit, not the host's.

## Interview question
**Q: A container is OOMKilled even though `kubectl top pod` showed usage well under its limit most of the time. What's the likely explanation?**
A: `kubectl top` reports periodic snapshots, not continuous monitoring — a brief, sharp memory spike between polling intervals can cross the limit and trigger an instant SIGKILL without ever being visible in average or snapshot-based metrics. This is exactly why memory limit violations have zero grace period, unlike CPU throttling: there's no "average was fine" defense once a single instant crosses the cgroup ceiling.

---

## What's coming in Part 2

Part 2 covers: **CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted** — each with the full 12-point treatment, realistic error messages, and production examples.

Say **"continue"** whenever you're ready. — Complete Masterclass
## Part 1 of 4: CrashLoopBackOff, ImagePullBackOff, ErrImagePull, Pending, OOMKilled

*(This masterclass covers 15 failure modes at full depth — delivered in installments so each gets the complete 12-point treatment rather than being compressed. Part 1 covers the five most frequently encountered failures. Say "continue" for Part 2: CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted.)*

---

# THE UNIVERSAL TROUBLESHOOTING ENTRY POINT

Before diving into individual failures, every single one of them starts the same way:
```bash
kubectl get pods                    # spot the abnormal STATUS column
kubectl describe pod <pod>           # Events section — almost always names the exact problem
kubectl logs <pod>                    # what the CONTAINER itself said before dying
kubectl logs <pod> --previous          # logs from the PREVIOUS crashed instance (crucial for CrashLoopBackOff)
```
Every section below assumes you've already run these four commands — they're the universal first move, not repeated as "step 1" for every failure type below.

---

# 1. CrashLoopBackOff

## What it means
The container **starts, then exits** (crashes, or even exits cleanly with code 0 when it shouldn't), repeatedly — and Kubernetes is now waiting increasingly long **backoff** periods between each restart attempt rather than restarting instantly forever.

## What Kubernetes is doing internally
This is a **kubelet-local** behavior (Pods masterclass, Diagram D1) — no scheduler, no controller-manager involvement. The kubelet observes the container exit, and per `restartPolicy: Always` (the Deployment/ReplicaSet default), restarts it — but with **exponential backoff**: 10s, 20s, 40s, 80s... capped at 5 minutes between attempts, specifically to avoid hammering a fundamentally broken container in a tight, wasteful restart loop.
```
Container exits → kubelet waits (backoff) → restarts → exits again → wait LONGER → restart → ...
```

## Common causes
- Application crashes immediately on startup (missing config, unhandled exception, failed dependency connection)
- Wrong `command`/`args` — e.g., a command that runs once and exits, used on a Pod expecting a long-running process
- A missing environment variable or file the app requires to even initialize
- The main process legitimately finishing (exit 0) in a container Kubernetes expects to run forever

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     CrashLoopBackOff   7          12m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl logs my-app                 # current attempt (often empty if it crashed instantly)
kubectl logs my-app --previous       # THE crash's actual output — usually the real answer
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      ...
  Finished:     ...
Restart Count:  7
```
**Exit code matters a lot here:**
- `0` — the process exited cleanly, meaning your restartPolicy expectations and the app's actual behavior disagree (app thinks it's done; Kubernetes expects it to run forever)
- `1` (or other nonzero) — generic application error; the actual reason is almost always in `logs --previous`
- `137` — this is actually `OOMKilled` wearing a CrashLoopBackOff's clothes (§5) — always check `Reason` explicitly, not just the exit code

## Root-cause investigation
```bash
kubectl logs my-app --previous --timestamps
kubectl describe pod my-app | grep -A10 "Last State"
# reproduce locally if possible:
docker run --rm <same-image> <same-command>
```
The single highest-leverage move: **read `logs --previous` before doing anything else** — the overwhelming majority of CrashLoopBackOff cases are fully explained by the application's own final log lines before it died.

## Fixes
- Fix the actual application bug/misconfiguration revealed by the logs
- If it's a legitimately run-to-completion process, use a **Job** (Jobs guide), not a Deployment
- Add a `startupProbe` (§15) if the app just needs more time before health checks should even begin evaluating it

## Verification
```bash
kubectl get pod my-app -w
# RESTARTS stops climbing, STATUS settles to Running, READY reaches 1/1
```

## Prevention
- Fail fast and log clearly on startup — a silent crash with no log output is far harder to diagnose than one that prints exactly what config was missing
- Use `startupProbe` for slow-initializing apps so the kubelet doesn't prematurely judge them unhealthy during legitimate startup time

## Production example
A Node.js app crash-looped in production because a required environment variable (`DATABASE_URL`) was renamed in a ConfigMap update but the Deployment's `env` reference still pointed at the old key name — `logs --previous` showed `TypeError: Cannot read property 'connect' of undefined` within milliseconds of each restart, immediately pointing at the missing config rather than a code bug.

## Interview question
**Q: A Pod is in CrashLoopBackOff with exit code 0. Why is that unusual, and what does it suggest?**
A: Exit code 0 means the process believed it finished successfully — but `restartPolicy: Always` (typical under a Deployment) treats *any* exit as something to restart. This mismatch suggests either the workload should actually be a Job (finite, run-to-completion) rather than a Deployment, or the application has a bug causing it to terminate early when it should keep running.

---

# 2–3. ImagePullBackOff and ErrImagePull

## What they mean
`ErrImagePull` is the **first** failed attempt to pull a container image. `ImagePullBackOff` is what you see on **subsequent** attempts, once the kubelet starts backing off between retries — the same exponential-backoff relationship as CrashLoopBackOff, just applied to image pulls instead of container starts.

## What Kubernetes is doing internally
```
kubelet sees a Pod assigned to its node
    → calls the container runtime (via CRI) to pull the specified image
    → runtime attempts to contact the registry, authenticate, download layers
    → FAILS → kubelet reports ErrImagePull, schedules a retry with backoff
    → next failure → reports ImagePullBackOff, backoff continues to grow
```

## Common causes
- Typo in the image name or tag (`myapp:1.O` instead of `1.0` — a classic letter-O-vs-zero mistake)
- Image genuinely doesn't exist in the registry, or was deleted
- Private registry requiring authentication, with no (or wrong) `imagePullSecrets` configured
- Rate limiting from the registry (Docker Hub's anonymous pull limits are a very common real-world trigger)
- Network policy or firewall blocking egress from nodes to the registry

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     ImagePullBackOff   0          2m
```

## Commands to run
```bash
kubectl describe pod my-app
```

## How to interpret the output
```
Events:
  Warning  Failed     kubelet  Failed to pull image "myapp:1.O":
                                rpc error: code = NotFound desc = failed to pull and
                                unpack image "docker.io/library/myapp:1.O":
                                failed to resolve reference: myapp:1.O: not found
  Warning  Failed     kubelet  Error: ErrImagePull
  Normal   BackOff    kubelet  Back-off pulling image "myapp:1.O"
  Warning  Failed     kubelet  Error: ImagePullBackOff
```
The **exact error text after "Failed to pull image"** tells you precisely which failure mode you're in — `not found` (bad name/tag), `unauthorized`/`403` (auth problem), `connection refused`/`timeout` (network problem) all point in genuinely different directions.

## Root-cause investigation
```bash
# verify the image actually exists and is spelled correctly:
docker pull myapp:1.0          # try it manually, from a machine with equivalent network access
kubectl get pod my-app -o jsonpath='{.spec.containers[0].image}'   # confirm exactly what was requested
kubectl get pod my-app -o jsonpath='{.spec.imagePullSecrets}'       # confirm a pull secret is even attached
```

## Fixes
```bash
# for private registries:
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io --docker-username=me --docker-password=secret
```
```yaml
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: myregistry.io/myapp:1.0    # fix any typo here
```

## Verification
```bash
kubectl delete pod my-app        # if managed by a controller, a fresh Pod will retry immediately
kubectl get pod -w
```

## Prevention
- Pin exact image tags (never `:latest` in production — Pods masterclass-adjacent best practice) and verify them in CI before deployment
- Test `imagePullSecrets` in a staging namespace before relying on them in production
- Consider a pull-through cache/mirror to avoid public registry rate limits entirely

## Production example
A cluster's nodes suddenly couldn't pull any Docker Hub images after Docker Hub introduced anonymous pull rate limits — every new Pod scheduled that day hit `ErrImagePull` with a `429 Too Many Requests` message buried in the Events, resolved by adding authenticated `imagePullSecrets` (which have a much higher rate limit) cluster-wide via a default ServiceAccount patch.

## Interview question
**Q: What's the actual difference between ErrImagePull and ImagePullBackOff?**
A: They're the same underlying failure — ErrImagePull is the immediate, first-attempt failure state; ImagePullBackOff is what's reported once the kubelet starts throttling retry attempts with exponential backoff after repeated failures. Neither indicates a fundamentally different problem — they're two points on the same timeline.

---

# 4. Pending

## What it means
The Pod object exists in etcd, but **has not been scheduled to a node at all** (or, less commonly, is scheduled but can't start for some pre-container reason) — this is the Pod phase from the Pods masterclass §6–9, at its most literal.

## What Kubernetes is doing internally
```
Pod created → scheduler watches for Pods with empty .spec.nodeName
            → filtering phase eliminates unsuitable nodes
            → IF NO NODE SURVIVES FILTERING: Pod stays Pending, scheduler
              retries on backoff and whenever cluster state changes
              (Architecture masterclass, Part 4 §28)
```

## Common causes
- **Insufficient resources** cluster-wide for the Pod's requests (Resource Requests guide)
- **Taints without matching tolerations** — every node the Pod could otherwise fit on is tainted against it
- **Node selector/affinity rules** that no current node satisfies
- **No available PersistentVolume** matching a required PVC (Storage guide) — scheduling waits on binding
- Zero worker nodes exist yet (a brand-new cluster, or all nodes cordoned/draining)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS    RESTARTS   AGE
# my-app    0/1     Pending   0          5m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl get nodes
kubectl describe nodes | grep -A5 "Allocated resources"
```

## How to interpret the output
```
Events:
  Warning  FailedScheduling  0/5 nodes are available: 3 Insufficient cpu,
           2 node(s) had taint {dedicated: gpu}, that the pod didn't tolerate.
```
This message is **exhaustive and literal** — it tells you exactly how many nodes were rejected and for which specific reason, across every node in the cluster. Read it fully before guessing.

## Root-cause investigation
```bash
kubectl get pod my-app -o jsonpath='{.spec.tolerations}'
kubectl get pod my-app -o jsonpath='{.spec.nodeSelector}'
kubectl get pod my-app -o jsonpath='{.spec.containers[*].resources}'
kubectl describe nodes | grep -B2 Taints
```

## Fixes
- Reduce over-inflated resource requests, or add cluster capacity (more nodes, or trigger a cluster autoscaler)
- Add the required `tolerations` if the workload genuinely belongs on a tainted node pool
- Correct a `nodeSelector`/`affinity` rule that's impossible to satisfy (e.g., referencing a label that no node actually has, often a typo)

## Verification
```bash
kubectl get pod my-app -w
# STATUS moves from Pending to ContainerCreating to Running
```

## Prevention
- Right-size resource requests based on actual observed usage (Resource Requests guide) rather than guessing high "to be safe," which wastes schedulable capacity across the whole cluster
- Keep taints/tolerations and nodeSelectors documented and reviewed — these are easy to silently break during node pool migrations

## Production example
A team migrated from one node pool to another with different labels; their Deployment's `nodeSelector` still referenced the old pool's label, and every new Pod sat `Pending` indefinitely after the old pool was fully drained — `FailedScheduling`'s message explicitly listed "0/12 nodes match node selector," which immediately pointed at the stale selector.

## Interview question
**Q: Is a Pending Pod always a resource-capacity problem?**
A: No — resource insufficiency is one cause among several (taints/tolerations mismatches, node selector/affinity that no node satisfies, an unbound PVC, or literally zero nodes existing). The `FailedScheduling` event message always states the specific reason(s); assuming it's always "not enough CPU/memory" without reading the actual message is a common, avoidable diagnostic mistake.

---

# 5. OOMKilled

## What it means
*(Full internal mechanics already covered exhaustively in the Resource Limits guide, Part 3 — this section is the troubleshooting-flow-focused summary.)* A container exceeded its memory limit and was forcibly killed by the kernel's OOM killer, scoped to that container's own cgroup.

## What Kubernetes is doing internally
```
Container's memory usage → approaches cgroup memory.max
    → kernel reclaims what it can (silent, no kill yet)
    → still over → kernel OOM killer sends SIGKILL to the container's process
    → container exits with code 137 (128 + SIGKILL's signal number 9)
    → kubelet observes the exit, reports Reason: OOMKilled
    → restartPolicy governs what happens next (same restart mechanics as
      any container exit, per Pods masterclass Diagram D1)
```

## Common causes
- Memory limit set genuinely too low for real peak usage
- A memory leak — usage climbs steadily over the container's lifetime rather than stabilizing
- A sudden legitimate spike (large request payload, batch processing burst) exceeding a limit sized for steady-state only
- No limit set at all, but the **node itself** ran out of memory (node-level OOM, per the Resource Limits guide — more disruptive, can affect unrelated Pods)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS      RESTARTS   AGE
# my-app    0/1     OOMKilled   3          8m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl top pod my-app --containers      # if it's currently running, see live usage
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Started:      ...
  Finished:     ...
```
**Exit code 137 plus `Reason: OOMKilled` together are unambiguous** — this is never a coincidence or a generic crash; it's specifically the kernel's memory enforcement, distinct from every other failure in this guide.

## Root-cause investigation
```bash
kubectl describe pod my-app | grep -A3 "Limits\|Requests"
kubectl top pod my-app --containers        # compare against the limit, if still running long enough
# check historical usage if you have metrics retention (Prometheus, etc.) —
# was this a SLOW climb (leak) or a SUDDEN spike (load-driven)?
```

## Fixes
- Raise the memory limit to a realistic ceiling based on observed peak usage, with headroom
- Fix an actual memory leak if usage climbs without bound over time rather than stabilizing
- For genuinely bursty workloads, consider whether the request should be raised too (affects scheduling and QoS, per the Resource Requests/Limits guides), not just the limit

## Verification
```bash
kubectl get pod my-app -w
kubectl top pod my-app --containers -w
# usage should stabilize comfortably under the new limit, RESTARTS stops climbing
```

## Prevention
- Load-test with realistic peak traffic before setting production memory limits
- Alert on Pods approaching their memory limit (e.g., >80% sustained) **before** they get OOMKilled, not just after
- Use Guaranteed QoS (Resource Limits guide) for workloads where an OOMKill is especially costly/disruptive

## Production example
A Java application was OOMKilled repeatedly after a Kubernetes migration because its JVM heap flags were set assuming the *node's* total memory, not the container's cgroup limit — the JVM allocated far more heap than the container was actually permitted, guaranteeing an eventual OOMKill under any real load; the fix was explicit `-Xmx` flags sized against the container's own memory limit, not the host's.

## Interview question
**Q: A container is OOMKilled even though `kubectl top pod` showed usage well under its limit most of the time. What's the likely explanation?**
A: `kubectl top` reports periodic snapshots, not continuous monitoring — a brief, sharp memory spike between polling intervals can cross the limit and trigger an instant SIGKILL without ever being visible in average or snapshot-based metrics. This is exactly why memory limit violations have zero grace period, unlike CPU throttling: there's no "average was fine" defense once a single instant crosses the cgroup ceiling.

---

## What's coming in Part 2

Part 2 covers: **CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted** — each with the full 12-point treatment, realistic error messages, and production examples.

Say **"continue"** whenever you're ready.


Understood — you're pointing me back to the **Pod Troubleshooting Masterclass**, continuing with Part 2 (CreateContainerError, CreateContainerConfigError, FailedMount, stuck ContainerCreating, and Evicted). I'll apply the new first-principles teaching standard you just set going forward, including here.

---

# Kubernetes Pod Troubleshooting — Complete Masterclass
## Part 2 of 4: CreateContainerError, CreateContainerConfigError, FailedMount, Stuck ContainerCreating, Evicted

*(Continuing from Part 1, now taught at first-principles depth: every term defined before use, WHAT vs WHY kept explicit, diagrams for every internal mechanism.)*

---

# 6. CreateContainerConfigError

## 1. What it means

Before defining this, you need one term: **"admission" / "creating" a container** here refers to the phase where the kubelet has already decided this Pod belongs on this node, and is now trying to actually assemble everything the container needs *before* the container's first line of code ever runs — pulling in environment variables, mounting files, checking security settings.

`CreateContainerConfigError` means: **the kubelet tried to assemble the container's configuration, and something it needed simply doesn't exist or is invalid — so the container was never even started.**

This is the failure state Incident 1's teardown mentioned as the sibling case to CrashLoopBackOff — worth restating precisely now: CrashLoopBackOff means the container **started and then died**. `CreateContainerConfigError` means the container **never started at all**, because the kubelet couldn't even finish preparing it.

## 2. What Kubernetes is doing internally

```
┌───────────────────────────────────────────────────────────────┐
│  BEFORE the container process can start, the kubelet must:         │
│                                                                       │
│  1. Resolve every env/envFrom reference                                │
│     → fetch each referenced ConfigMap/Secret from the API server         │
│     → does the referenced object actually EXIST? does the KEY exist?      │
│                                                                              │
│  2. Resolve every volume that needs a ConfigMap/Secret/PVC                    │
│     → same existence checks                                                    │
│                                                                                    │
│  3. Validate the securityContext against what the image allows                       │
│     → e.g., if runAsNonRoot: true is set, does the image's default user            │
│       actually satisfy that?                                                          │
│                                                                                           │
│  IF ANY of these checks fail → the container is NEVER started.                             │
│  The kubelet reports: CreateContainerConfigError                                             │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**WHY this is a distinct state from CrashLoopBackOff (the WHY, not just the WHAT):** Kubernetes deliberately separates "I couldn't even build this container's configuration" from "the container ran and then failed on its own." This distinction exists because the *fix* is completely different — a `CreateContainerConfigError` points you at your **Kubernetes YAML** (a missing reference, an impossible security setting). A `CrashLoopBackOff` points you at your **application's own behavior** once it's actually running. Collapsing these into one generic "broken" state would waste your time investigating the wrong layer.

## 3. Common causes

- A `env`/`envFrom` reference to a ConfigMap or Secret that **doesn't exist** in that namespace at all
- A `configMapKeyRef`/`secretKeyRef` referencing a **key that doesn't exist** within an object that *does* exist
- `securityContext.runAsNonRoot: true` combined with an image whose default user is root, and no `runAsUser` specified to override it
- A namespace mismatch — the ConfigMap exists, but in a *different* namespace than the Pod (Kubernetes objects generally can't be referenced across namespace boundaries this way)

## 4. How to identify it

```bash
kubectl get pods -n prod
```
```
NAME                       READY   STATUS                       RESTARTS   AGE
myapp-6b9f7c8d9f-x4k2p     0/1     CreateContainerConfigError   0          2m
```
**Notice `RESTARTS: 0`** — this is a meaningful, diagnostic detail (per your rule 23, don't skip edge cases): the container was never running in the first place, so there's nothing to "restart." A restart count of 0 alongside a config error status is itself a clue confirming you're in this failure mode, not CrashLoopBackOff.

## 5. Commands to run

```bash
kubectl describe pod myapp-6b9f7c8d9f-x4k2p -n prod
```

## 6. How to interpret the output

```
Events:
  Type     Reason     Age   From     Message
  ----     ------     ----  ----     -------
  Warning  Failed     30s   kubelet  Error: configmap "app-config" not found
```
or
```
  Warning  Failed     30s   kubelet  Error: couldn't find key DATABASE_URL in ConfigMap prod/app-config
```
or
```
  Warning  Failed     30s   kubelet  Error: container has runAsNonRoot and image will run as root
```
**These three messages map to three entirely different fixes** — this is why reading the exact text matters more than pattern-matching on the status name alone.

## 7. Root-cause investigation

```bash
# does the referenced object exist AT ALL, in this namespace?
kubectl get configmap,secret -n prod

# if it exists, does the specific KEY exist inside it?
kubectl get configmap app-config -n prod -o yaml

# what does the Pod spec ACTUALLY reference?
kubectl get pod myapp-6b9f7c8d9f-x4k2p -n prod -o jsonpath='{.spec.containers[0].env}'
kubectl get pod myapp-6b9f7c8d9f-x4k2p -n prod -o jsonpath='{.spec.containers[0].envFrom}'
```

## 8. Fixes

- Create the missing ConfigMap/Secret, or correct a typo'd name in the Pod spec
- Add the missing key to the existing object, or correct a typo'd key reference
- For the `runAsNonRoot` case: either set an explicit `runAsUser` matching a non-root UID the image supports, or use a base image built to run as non-root by default

## 9. Verification

```bash
kubectl delete pod myapp-6b9f7c8d9f-x4k2p -n prod
kubectl get pods -n prod -w
```
(If managed by a Deployment, a fresh Pod is created automatically once you delete the broken one — deleting it forces a fresh attempt using whatever you just fixed.)

## 10. Prevention

- **Production practice, not a Kubernetes requirement (rule 17):** validate that every ConfigMap/Secret a Deployment references actually exists, as part of your CI/CD pipeline, before the manifest ever reaches the cluster.
- Prefer explicit `env`/`valueFrom` over `envFrom` specifically because a typo'd key name in explicit form is easier to spot in a code review than a silent bulk-import mismatch (same reasoning as Incident 1).

## 11. Production example

A team's CI/CD pipeline deployed a new Deployment referencing `app-config-v2`, but a separate pipeline step responsible for creating that ConfigMap ran *after* the Deployment step due to a race condition in pipeline ordering — every Pod hit `CreateContainerConfigError` for about 45 seconds until the ConfigMap-creation step caught up, then self-resolved once the Deployment's ReplicaSet controller's next Pod-creation attempt succeeded.

## 12. Interview question

**Q: What's the fundamental difference between CreateContainerConfigError and CrashLoopBackOff, and why does Kubernetes distinguish them?**
A: `CreateContainerConfigError` means the container's configuration (env vars, volumes, security settings) couldn't even be assembled — the container process never starts. `CrashLoopBackOff` means the container *did* start, and then the application itself exited, repeatedly. Kubernetes distinguishes them because the fix lives in different places: the former points at your Kubernetes manifest's references; the latter points at your application's runtime behavior.

---

# 7. CreateContainerError

## 1. What it means

A closely related but distinct state from the one above: the kubelet successfully **built** the container's configuration (all references resolved fine), but the **container runtime itself failed to actually create the container** when asked.

**Term to define: "container runtime."** This is the software actually responsible for creating and running containers on a node — commonly `containerd`. The kubelet doesn't create containers itself; it *asks* the container runtime to do so, using a standardized interface. `CreateContainerConfigError` is a kubelet-side failure (before it even asks); `CreateContainerError` is a failure in that ask-and-execute step itself.

## 2. What Kubernetes is doing internally

```
kubelet: "config looks good, container runtime, please CREATE this container"
                        │
                        ▼
container runtime attempts to:
  - set up the container's filesystem from the image layers
  - apply cgroup limits (CPU/memory constraints)
  - apply namespace isolation
  - mount any volumes into the container's filesystem
                        │
                        ▼
              SOMETHING FAILS HERE
                        │
                        ▼
kubelet reports: CreateContainerError
```

## 3. Common causes

- A container name collision or a stuck, orphaned container left behind from a previous failed attempt on that node
- Invalid or conflicting mount paths (e.g., trying to mount two different volumes at the exact same path)
- The node's container runtime itself is unhealthy or resource-exhausted
- An OCI runtime spec violation — a request the runtime fundamentally can't satisfy (a nonsensical resource limit value, for example)

## 4. How to identify it

```bash
kubectl get pods -n prod
```
```
NAME                     READY   STATUS                RESTARTS   AGE
myapp-6b9f7c8d9f-x4k2p   0/1     CreateContainerError   0          1m
```

## 5. Commands to run

```bash
kubectl describe pod myapp-6b9f7c8d9f-x4k2p -n prod
```

## 6. How to interpret the output

```
Events:
  Warning  Failed  20s   kubelet  Error: failed to create containerd task:
                                    OCI runtime create failed: ... duplicate mount
                                    destination "/data"
```

## 7. Root-cause investigation

```bash
kubectl get pod myapp-6b9f7c8d9f-x4k2p -n prod -o yaml | grep -A5 volumeMounts
# look for two entries mounting to the same mountPath
```

## 8. Fixes

Correct the conflicting configuration (e.g., unique mount paths per volume). If the cause is node-level runtime health, that becomes a node-layer investigation rather than a Pod-layer one — check `kubectl describe node` and the container runtime's own logs on that node.

## 9. Verification

Same as above: delete the Pod, let its controller recreate it, watch for `Running`.

## 10. Prevention

Validate volume mount paths for uniqueness at manifest-authoring time — this is exactly the kind of mistake a schema-linting step in CI can catch before deployment.

## 11. Production example

A hand-edited manifest had two `volumeMounts` entries both targeting `/etc/config` — one intended for a ConfigMap, one for a Secret — a copy-paste error where the second entry's `mountPath` wasn't updated. Every replica failed identically with `CreateContainerError`, since the mistake was baked into the shared Pod template.

## 12. Interview question

**Q: A Pod shows CreateContainerError. Is this an application bug?**
A: No — by definition, the application's code never got a chance to run at all in this state. The failure is in the container runtime's attempt to construct the container's execution environment (filesystem, mounts, cgroups) — a Kubernetes/infrastructure-layer problem, not an application-layer one.

---

# 8. FailedMount

## 1. What it means

**Term to define first: "mount."** As established in the ConfigMaps chapter, mounting means making some storage appear at a path inside a container's filesystem. `FailedMount` means: the kubelet tried to attach and/or mount a volume for this Pod, and that operation failed.

## 2. What Kubernetes is doing internally

```
┌─────────────────────────────────────────────────────────────────┐
│  For a volume backed by real external storage (not ConfigMap/Secret,  │
│  which are simpler), TWO separate steps happen:                          │
│                                                                              │
│  STEP 1: ATTACH                                                              │
│    The node's operating system connects to the actual storage device          │
│    (e.g., a cloud block storage volume) — this makes the raw device            │
│    available to the node, but it's not yet usable by any container yet          │
│                                                                                     │
│  STEP 2: MOUNT                                                                       │
│    The attached device is formatted/recognized with a filesystem, and                  │
│    made to appear at the specific path inside the Pod's container                        │
└─────────────────────────────────────────────────────────────────────────────────┘
```
`FailedMount` can happen at either step, though the term covers both loosely in practice — `FailedAttachVolume` is the more specific event name for step 1 failures specifically.

## 3. Common causes

- The referenced PVC (**Persistent Volume Claim** — a request for storage) doesn't exist, or isn't yet bound to actual storage
- The storage is still attached to a **different node** from a previous Pod placement (very common right after a fast reschedule — the old attachment hasn't finished detaching yet)
- Wrong filesystem type expectations, or actual filesystem corruption on the volume
- Network-based storage (like NFS) where the backend server is unreachable from this specific node
- Permission/ownership mismatches once mounted (covered under "permission problems" generally)

## 4. How to identify it

```bash
kubectl get pods -n prod
```
```
NAME                     READY   STATUS              RESTARTS   AGE
myapp-6b9f7c8d9f-x4k2p   0/1     ContainerCreating   0          6m
```
**Notice: the STATUS column shows `ContainerCreating`, not literally "FailedMount"** — this is an important, easy-to-miss detail. `FailedMount` is the **Event reason**, not a Pod phase or status you'll see in `kubectl get pods` directly. The Pod just looks "stuck" in `ContainerCreating` from the `get` view alone — you only see `FailedMount` by looking at Events specifically. This is exactly why the next command is non-optional here, more so than in most other failures.

## 5. Commands to run

```bash
kubectl describe pod myapp-6b9f7c8d9f-x4k2p -n prod
```

## 6. How to interpret the output

```
Events:
  Warning  FailedMount  90s (x3 over 3m)  kubelet  Unable to attach or mount volumes:
           unmounted volumes=[data], unattached volumes=[data]: timed out waiting
           for the condition
```
The `(x3 over 3m)` notation matters — it tells you this has been **retried multiple times already**, not a one-off blip. Kubernetes automatically retries mount attempts; seeing this repeat is a signal the underlying cause is persistent, not transient.

## 7. Root-cause investigation

```bash
kubectl get pvc -n prod
# is the PVC Bound, or still Pending?

kubectl describe pvc <claim-name> -n prod
# Events here often show the DEEPER cause — e.g., a provisioning failure

kubectl get pod myapp-6b9f7c8d9f-x4k2p -n prod -o jsonpath='{.spec.nodeName}'
# which node is this Pod trying to run on?

# check if the volume is still attached elsewhere (cloud-provider-specific,
# but conceptually: was this Pod just rescheduled from a different node?)
kubectl get events -n prod --field-selector involvedObject.name=<pod-name>
```

## 8. Fixes

- If the PVC itself is unbound: fix the underlying provisioning issue (see the dedicated Storage guide's troubleshooting section for the full StorageClass/CSI-driver chain)
- If it's a stuck attachment from a fast reschedule: often self-resolves within a couple of minutes as the old node's detach completes; if persistent, may require manual intervention at the cloud-provider level
- If it's a genuine permission issue: add `fsGroup` under the Pod's `securityContext` so the volume's ownership matches the container's user

## 9. Verification

```bash
kubectl get pods -n prod -w
```
Watch for `ContainerCreating` to resolve into `Running` once the mount actually succeeds.

## 10. Prevention

For workloads using `ReadWriteOnce`-type storage (attachable to only one node at a time), avoid aggressive, fast Pod rescheduling patterns without accounting for detach/attach latency — a production-practice consideration, not a Kubernetes rule.

## 11. Production example

A StatefulSet Pod was evicted from a failing node and rapidly rescheduled to a healthy one by the StatefulSet controller. The new Pod's volume mount failed with `FailedMount` for about 90 seconds because the cloud block storage volume was still in the process of detaching from the original (failing) node — this resolved itself automatically once the detach completed, with no manual action needed, but caused a brief, confusing "why is this stuck" moment for the on-call engineer.

## 12. Interview question

**Q: You see a Pod stuck in ContainerCreating for several minutes. Where do you look, and why isn't the STATUS column itself informative here?**
A: `kubectl describe pod` — specifically the Events section. `ContainerCreating` is a broad, generic status covering many possible in-progress or stuck states (image pulling, volume mounting, sandbox creation); it does not by itself tell you *which* step is stuck. The specific reason (like `FailedMount`) only appears in Events, which is why `describe`, not `get`, is the required next step for this particular status.

---

# 9. ContainerCreating (Stuck)

## 1. What it means

Worth treating as its own entry, since "stuck ContainerCreating" is a **symptom that can be caused by several of the other failures in this guide**, not a distinct root cause of its own. `ContainerCreating` is simply the status shown while the kubelet is in the middle of the multi-step process of preparing and starting a container — it becomes a *problem* specifically when it never progresses past this point.

## 2. What Kubernetes is doing internally — the full sequence this status covers

```
Pod bound to a node
       │
       ▼
kubelet begins:
  1. Create the Pod "sandbox" (the shared network/IPC namespace — this
     is exactly the "pause container" concept covered in the Pods
     masterclass)
       │
  2. Pull required container image(s), if not already cached locally
       │
  3. Resolve and mount all volumes
       │
  4. Actually start the container process
       │
       ▼
   Pod phase → Running
```
**`ContainerCreating` covers ALL of steps 1 through 3.** If the Pod is stuck here, the actual stuck step could be any one of them — this is precisely why this "failure" doesn't get its own separate diagnostic path; instead, it's a signal to go check Events for which *specific* sub-step is actually failing.

## 3. Common causes (mapped to which step they belong to)

| Stuck at step | Likely cause |
|---|---|
| Sandbox creation | CNI (networking plugin) misconfiguration or failure on this node |
| Image pull | Same causes as ImagePullBackOff (Part 1) — just caught here before backoff kicks in |
| Volume mount | FailedMount (section 8, above) |

## 4. How to identify it

```bash
kubectl get pods -n prod
```
```
NAME                     READY   STATUS              RESTARTS   AGE
myapp-6b9f7c8d9f-x4k2p   0/1     ContainerCreating   0          8m
```
**A Pod sitting at `ContainerCreating` for more than roughly a minute or two** (image pulls of reasonably-sized images should complete faster than that under normal conditions) is worth investigating — it's not necessarily broken immediately, but it's abnormal enough to check.

## 5. Commands to run

```bash
kubectl describe pod myapp-6b9f7c8d9f-x4k2p -n prod
```
This single command's Events section is the entire investigation — it will show you whichever of the sub-steps above is actually the bottleneck, in plain text.

## 6. How to interpret the output

```
Events:
  Normal   Scheduled  8m    default-scheduler  Successfully assigned prod/myapp... to node-3
  Warning  FailedCreatePodSandBox  7m (x5 over 8m)  kubelet  Failed to create pod
           sandbox: rpc error: code = Unknown desc = failed to setup network for
           sandbox: plugin type="calico" failed
```
This particular message tells you the problem is at the **CNI / networking plugin level**, on this specific node — a node-layer investigation, not something fixable by changing the Pod's own YAML at all.

## 7. Root-cause investigation

```bash
# is this specific to ONE node, or happening cluster-wide?
kubectl get pods -A -o wide | grep ContainerCreating

# check the CNI plugin's own Pods (they typically run as a DaemonSet —
# one Pod per node, covered in the DaemonSets masterclass)
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
```
**Differential technique worth calling out explicitly:** if only Pods scheduled to `node-3` are stuck, and Pods on other nodes are fine, this strongly localizes the problem to `node-3` specifically — check that node's own CNI agent Pod and its logs directly, rather than assuming a cluster-wide networking outage.

## 8. Fixes

Entirely dependent on which sub-step is actually stuck (per the table in section 3) — this "failure" is a **routing signal to the correct dedicated section**, not a fix in itself.

## 9. Verification

```bash
kubectl get pods -n prod -w
```

## 10. Prevention

Monitor node-level CNI agent health as a standing signal (per the DaemonSets masterclass's production guidance) — catching a broken networking agent on one node before it causes a stuck-Pod incident is far better than diagnosing it after the fact.

## 11. Production example

A single node's Calico agent crashed after a kernel update changed an expected `iptables` behavior slightly. Every *new* Pod scheduled to that node afterward got stuck in `ContainerCreating` with `FailedCreatePodSandBox`, while Pods already running on that node before the crash kept working undisturbed — a clear signal, once noticed, that this was a **new-Pod-only, node-specific** problem, not a broad outage.

## 12. Interview question

**Q: Why doesn't "ContainerCreating" have its own dedicated fix, the way CrashLoopBackOff does?**
A: Because `ContainerCreating` isn't a specific failure — it's the general in-progress status covering several completely different underlying steps (sandbox creation, image pull, volume mounting). Diagnosing it always means first identifying *which* of those steps is actually stuck, via `kubectl describe`'s Events, and then following that specific sub-problem's own troubleshooting path.

---

# 10. Evicted

## 1. What it means

**Term to define: "eviction."** Eviction is when the kubelet (or, in a different but related mechanism, the API server directly) deliberately terminates a Pod — not because the Pod's own container crashed, but because something *external* to the Pod required it to be removed, usually to protect the health of the node it was running on.

This is fundamentally different from every failure covered so far: CrashLoopBackOff, ImagePullBackOff, etc. are all about the Pod *itself* being broken. Eviction can happen to a **perfectly healthy** Pod, entirely because of conditions on the node it happens to be sitting on.

## 2. What Kubernetes is doing internally

```
┌────────────────────────────────────────────────────────────────┐
│  The kubelet continuously monitors its own node's resource health:  │
│    - available memory                                                 │
│    - available disk space                                              │
│    - available process IDs (PIDs)                                        │
│                                                                              │
│  If any of these drops below a configured threshold, the node enters         │
│  a "pressure" condition — e.g., MemoryPressure: True                          │
│                                                                                    │
│  Once under pressure, the kubelet's EVICTION MANAGER picks Pods to REMOVE,          │
│  ranked by a specific priority order (poorest-protected first), to free up            │
│  enough of the pressured resource to bring the node back to a healthy state             │
└────────────────────────────────────────────────────────────────────────────────┘
```

**The ranking order, stated precisely (this connects directly to QoS classes covered elsewhere — defined briefly here since it's essential to this specific failure):** Pods with no resource requests/limits declared at all are evicted first; Pods that are using more than they *requested* are evicted next; Pods using no more than they requested, especially ones where requests exactly equal limits, are evicted last, and only as a genuine last resort.

## 3. Common causes

- Node running low on memory, and this Pod was using more memory than it requested (even if under its *limit*)
- Node running low on disk space (often from accumulated logs or container image layers)
- The Pod had no resource requests declared at all, making it the very first candidate under any pressure, regardless of how little it was actually using at that moment
- A **manual, deliberate** eviction — e.g., `kubectl drain` was run against the node as part of planned maintenance

## 4. How to identify it

```bash
kubectl get pods -n prod
```
```
NAME                     READY   STATUS    RESTARTS   AGE
myapp-6b9f7c8d9f-x4k2p   0/1     Evicted   0          15m
```
**Note: an Evicted Pod is NOT automatically deleted** — it lingers in your `kubectl get pods` output, taking up a visible slot, until something cleans it up. This surprises people the first time they see it: the Pod object still exists (for post-mortem inspection), even though it's not running anything and never will again.

## 5. Commands to run

```bash
kubectl describe pod myapp-6b9f7c8d9f-x4k2p -n prod
```

## 6. How to interpret the output

```
Status:       Failed
Reason:       Evicted
Message:      The node was low on resource: memory. Container app was using
              612Mi, which exceeds its request of 256Mi.
```
**This message is unusually informative compared to most failures in this guide** — it directly states the resource under pressure (`memory`), the exact usage at the moment of eviction (`612Mi`), and the declared request (`256Mi`) — you can often diagnose the entire root cause from this one line alone, without any further investigation.

## 7. Root-cause investigation

```bash
# was this ONE Pod evicted, or many Pods on the SAME node, at the SAME time?
kubectl get events -n prod --field-selector reason=Evicted --sort-by=.metadata.creationTimestamp

# check the node's overall condition
kubectl describe node <node-name> | grep -A5 Conditions
```
**If many unrelated Pods were evicted from the same node around the same time**, this points to genuine node-wide resource exhaustion (possibly from one specific noisy-neighbor Pod, or simply an under-provisioned node) rather than a problem specific to any single Pod.

## 8. Fixes

- Raise the evicted Pod's memory **request** to more accurately reflect its real usage (this affects both scheduling and its priority in future eviction decisions)
- If it's a genuine leak (usage climbing without bound rather than a legitimate steady-state need), fix the leak in the application itself
- If the node itself is simply under-provisioned for what's scheduled on it, add cluster capacity or reduce what's packed onto that node

## 9. Verification

Since the ReplicaSet/Deployment controller behind this Pod will have already created a replacement automatically (per the standard "actual count dropped below desired" reconciliation covered in the ReplicaSets masterclass), verify the *replacement* is healthy, not the evicted Pod itself:
```bash
kubectl get pods -n prod -l app=myapp
```

## 10. Prevention

- Always set resource requests close to real observed steady-state usage — a production practice, not a Kubernetes mandate, but the single most effective prevention here
- Monitor node-level memory/disk pressure conditions as a standing dashboard signal, catching brewing pressure before it triggers actual evictions

## 11. Production example

A batch-processing Pod had no memory limit set at all and gradually consumed more and more memory as it processed an unusually large input file. It was never OOMKilled (it never hit a *limit*, since none was set) — instead, the *node itself* ran low on available memory as a direct consequence, and the eviction manager picked this Pod specifically because it was using far more than its declared (default, minimal) request — a clean illustration of why "no limit" doesn't mean "no consequence," just a different consequence (node-level eviction instead of container-level OOMKill).

## 12. Interview question

**Q: A container was never OOMKilled, but its Pod was Evicted for memory reasons. How is that possible?**
A: OOMKill happens when a container exceeds its own declared memory *limit* — a per-container, kernel-enforced ceiling. Eviction happens when the *node as a whole* runs low on memory, regardless of any individual container's limit. A container with no limit set (or a very high one) can keep growing without ever triggering an OOMKill, while still contributing to genuine node-wide memory exhaustion — at which point the kubelet's eviction manager, not the kernel's per-container OOM killer, steps in and removes it.

---

## Beginner summary (Part 2)

- `CreateContainerConfigError`: the container never started because something it referenced (a ConfigMap key, a Secret, a security setting) was missing or invalid.
- `CreateContainerError`: configuration was fine, but the container runtime itself failed to actually construct the container.
- `FailedMount`: a volume couldn't be attached or mounted — shows up as a stuck `ContainerCreating` status, with the real reason only visible in Events.
- `ContainerCreating` (stuck): a generic in-progress status covering sandbox creation, image pulling, and volume mounting — always requires `describe` to find which specific sub-step is actually stuck.
- `Evicted`: the Pod was deliberately removed by the kubelet to protect the node's own health, independent of whether the Pod's container was itself broken.

## Advanced summary (Part 2)

- The Config-vs-Runtime distinction (`CreateContainerConfigError` vs `CreateContainerError`) maps directly onto the kubelet/container-runtime boundary — one is the kubelet's own preparation work, the other is the container runtime's execution of that plan.
- `FailedMount` genuinely covers two distinct sub-operations (attach, then mount) that are worth distinguishing when investigating cloud block storage specifically, since attach failures often relate to prior node placement, not the current one.
- Eviction ranks Pods by a request-relative usage metric, not absolute usage — this is why identical real memory consumption can be "fine" for one Pod and "the first to go" for another, purely based on what each declared as its request.

---

# Kubernetes Pod Troubleshooting — Complete Masterclass
## Part 3 of 4: RunContainerError, Terminating Stuck Pods, and the Three Probe Failures

*(Continuing at first-principles depth. This part covers the last container-startup failure type, the deletion-side counterpart to "stuck creating," and the three probe types — which are less about crashes and more about Kubernetes correctly detecting that something is subtly wrong.)*

---

# 11. RunContainerError

## 1. What it means

You now have two other failure types that sound similar and need to be told apart clearly, since this is exactly the kind of "commonly confused" case your rules ask me to compare directly:

```
CreateContainerConfigError → the kubelet couldn't even ASSEMBLE the
                              container's configuration (missing
                              ConfigMap/Secret reference, invalid
                              security setting)

CreateContainerError       → configuration was fine, but the container
                              RUNTIME failed to CONSTRUCT the container
                              (filesystem setup, mounts, cgroups)

RunContainerError          → the container was successfully CONSTRUCTED,
                              but failed the moment the runtime tried to
                              actually START running its process
```

So `RunContainerError` is the **third and final stage** in this progression — everything before "start the process" succeeded, but the act of starting it itself failed.

## 2. What Kubernetes is doing internally

```
kubelet: "container is built, now RUN it"
                    │
                    ▼
container runtime attempts to execute the container's entrypoint
process — this means: take the `command`/`args` (or the image's
built-in default entrypoint if none is specified) and actually
invoke it as a new process on the node
                    │
                    ▼
        THIS INVOCATION ITSELF FAILS
        (not "the process ran and then crashed" — the OPERATING
         SYSTEM couldn't even successfully launch the process)
                    │
                    ▼
kubelet reports: RunContainerError
```

**Why is this a meaningfully different failure from CrashLoopBackOff, even though both eventually mean "no working container"?** In CrashLoopBackOff, the process genuinely starts — it exists, briefly, as a running thing on the operating system, and then it exits (perhaps in milliseconds). In `RunContainerError`, the operating system's attempt to even *create the process in the first place* fails — there was never a running process at all, not even for a moment. The distinction matters because the causes live in completely different places: one is almost always your application's own logic; the other is almost always something wrong with the executable itself, or the environment it's being asked to run in.

## 3. Common causes

- The specified `command` doesn't actually exist inside the container's filesystem (a classic case: an image built from a minimal/distroless base that has no shell, but the Pod spec's `command` tries to run `sh -c "..."`)
- The binary being executed doesn't have execute permissions set
- The binary was built for a different CPU architecture than the node it's running on (e.g., an image built for `arm64` scheduled onto an `amd64` node — increasingly common as mixed-architecture clusters become more normal)
- A resource limit is so restrictive that even the initial process launch cannot succeed (extremely low memory limits, for instance)

## 4. How to identify it

```bash
kubectl get pods -n prod
```
```
NAME                     READY   STATUS              RESTARTS   AGE
myapp-6b9f7c8d9f-x4k2p   0/1     RunContainerError    0          1m
```

## 5. Commands to run

```bash
kubectl describe pod myapp-6b9f7c8d9f-x4k2p -n prod
```

## 6. How to interpret the output

```
Events:
  Warning  Failed  15s   kubelet  Error: failed to start container "app":
                                    Error response from daemon: OCI runtime
                                    create failed: exec: "/app/start.sh":
                                    stat /app/start.sh: no such file or
                                    directory: unknown
```
This exact wording — `no such file or directory` for the *command itself*, not a file the command tries to open — is the signature of this specific failure. Compare this deliberately against a CrashLoopBackOff's typical message, which would instead show output from *inside* your application (a Python traceback, a stack trace) — here, the operating system never got far enough to hand control to your application code at all.

## 7. Root-cause investigation

```bash
# what command/entrypoint does the Pod spec actually specify?
kubectl get pod myapp-6b9f7c8d9f-x4k2p -n prod -o jsonpath='{.spec.containers[0].command}{"\n"}{.spec.containers[0].args}'

# inspect the image itself, independent of Kubernetes entirely, to
# confirm the file/binary actually exists where expected:
docker run --rm --entrypoint sh myregistry.io/myapp:1.0 -c "ls -la /app/"

# check image architecture vs node architecture
kubectl get node <node-name> -o jsonpath='{.status.nodeInfo.architecture}'
docker inspect myregistry.io/myapp:1.0 --format '{{.Architecture}}'
```

## 8. Fixes

- Correct the `command`/`args` to point at something that genuinely exists inside the image
- Rebuild the image to include the missing file, or fix its permissions (`chmod +x`) at build time
- For architecture mismatches: build and push a multi-architecture image, or use node affinity/selectors to ensure this Pod only schedules onto nodes matching the image's actual architecture

## 9. Verification

```bash
kubectl delete pod myapp-6b9f7c8d9f-x4k2p -n prod
kubectl get pods -n prod -w
```

## 10. Prevention

- Test images locally (`docker run`) before deploying to Kubernetes at all — this single habit catches the large majority of `RunContainerError` causes before they ever reach a cluster
- In mixed-architecture clusters, build multi-architecture images as standard practice, and consider it a required CI step rather than an afterthought

## 11. Production example

A team switched their CI runners from `amd64` to `arm64` machines to save cost, and their image-build step silently produced `arm64` images from that point on, while their production cluster's nodes were still all `amd64`. Every new deploy failed with `RunContainerError`, with an `exec format error` buried in the Events message — a classic, easy-to-miss consequence of infrastructure changing one layer removed from where the actual failure surfaced.

## 12. Interview question

**Q: How would you tell CrashLoopBackOff and RunContainerError apart just from the Events message, without knowing the status name?**
A: A CrashLoopBackOff's relevant evidence lives in `kubectl logs --previous` and shows output *from the application itself* — a stack trace, an error message the app printed. A RunContainerError's evidence lives entirely in the `describe`'s Events, phrased in terms of the operating system or container runtime failing to launch the process at all (e.g., "no such file or directory" referring to the command itself, or "exec format error") — there is no application output to inspect, because the application never got the chance to run even once.

---

# 12. Terminating Stuck Pods

## 1. What it means

First, define the term precisely, since "stuck Terminating" is really two separate ideas that need separating: **"Terminating" is not a Pod phase** (recall from Part 1: the only phases are `Pending`, `Running`, `Succeeded`, `Failed`, `Unknown`). "Terminating" is a **status `kubectl get pods` displays** when a Pod has been marked for deletion but hasn't yet actually finished shutting down.

A Pod is "stuck" in this state when that shutdown process takes far longer than expected, or never completes on its own at all.

## 2. What Kubernetes is doing internally — the full termination sequence, defined term by term

```
┌────────────────────────────────────────────────────────────────────┐
│  STEP 1: Someone (a user, or a controller during a rollout) requests    │
│  deletion — e.g., `kubectl delete pod myapp-x4k2p`                        │
│           │                                                                  │
│           ▼                                                                   │
│  STEP 2: The API server does NOT immediately remove the Pod object from       │
│  etcd. Instead, it sets a field called `deletionTimestamp` on the object —      │
│  this is a marker saying "this object is scheduled for removal," while           │
│  the object itself STILL EXISTS and is still visible to `kubectl get`              │
│           │                                                                          │
│           ▼                                                                           │
│  STEP 3: If this Pod is being routed to by a Service, it is immediately                 │
│  removed from that Service's list of valid targets — this happens right                   │
│  away, specifically so no NEW traffic gets sent to a Pod that's about to                    │
│  shut down                                                                                     │
│           │                                                                                       │
│           ▼                                                                                          │
│  STEP 4: The kubelet, which is independently watching for Pods with a                                   │
│  deletionTimestamp set, notices this Pod needs to shut down                                                │
│           │                                                                                                   │
│           ▼                                                                                                      │
│  STEP 5: If a `preStop` hook is defined (a command you can configure to run                                        │
│  right before shutdown — commonly used to let in-flight requests finish),                                            │
│  the kubelet runs it now, and WAITS for it to finish before continuing                                                   │
│           │                                                                                                                 │
│           ▼                                                                                                                    │
│  STEP 6: The kubelet sends a SIGTERM signal to the container's main process               │
│  "SIGTERM" is a standard Unix signal meaning, roughly, "please shut yourself         │
│  down gracefully" — a polite request, not a forceful kill                              │
│           │                                                                               │
│           ▼                                                                                  │
│  STEP 7: The kubelet waits up to `terminationGracePeriodSeconds` (a Pod spec                    │
│  field, defaulting to 30 seconds if not set) for the process to actually exit                      │
│  on its own                                                                                            │
│           │                                                                                               │
│           ▼                                                                                                  │
│  STEP 8: If the process is STILL running once that grace period expires, the                                   │
│  kubelet sends SIGKILL — an unconditional, immediate termination signal that                                       │
│  cannot be caught, ignored, or gracefully handled by the process at all                                                │
│           │                                                                                                               │
│           ▼                                                                                                                   │
│  STEP 9: Once the container is confirmed stopped, the kubelet reports this,                                                     │
│  and the API server finally removes the Pod object from etcd entirely                                                             │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

**"Stuck" Terminating means the Pod is stalled somewhere in steps 5 through 9** — most commonly, the process simply never responds to SIGTERM at all (step 6), forcing the full grace period to elapse every single time before SIGKILL can even be attempted.

## 3. Common causes

- The application doesn't handle SIGTERM at all — many programming language runtimes and process supervisors, by default, simply ignore this signal unless the application explicitly registers a handler for it
- A `preStop` hook that itself hangs or takes a very long time to complete (recall: the kubelet waits for this to fully finish before even sending SIGTERM)
- A **finalizer** — define this term: a finalizer is a marker on an object that says "don't fully delete this until some external controller confirms it's safe to." If a controller responsible for clearing a finalizer is itself broken or absent, the object can remain in a perpetual "Terminating" limbo indefinitely, since Kubernetes is deliberately waiting for a signal that will never come
- The node the Pod was running on is itself unreachable — the kubelet that would report "yes, this container is stopped" is simply not answering at all

## 4. How to identify it

```bash
kubectl get pods -n prod
```
```
NAME                     READY   STATUS        RESTARTS   AGE
myapp-6b9f7c8d9f-x4k2p   1/1     Terminating   0          45m
```
A Pod visibly stuck at `Terminating` for well beyond its `terminationGracePeriodSeconds` (which, remember, defaults to 30 seconds) is the signal something is wrong here — a Pod that's been "Terminating" for 45 minutes is unambiguously stuck, not just slow.

## 5. Commands to run

```bash
kubectl describe pod myapp-6b9f7c8d9f-x4k2p -n prod
# check for finalizers, in particular:
kubectl get pod myapp-6b9f7c8d9f-x4k2p -n prod -o jsonpath='{.metadata.finalizers}'
```

## 6. How to interpret the output

If `metadata.finalizers` returns a non-empty list, like:
```
["example.com/my-custom-finalizer"]
```
this immediately tells you the Pod is waiting on some **external controller** to acknowledge it's safe to fully delete — and that controller either doesn't exist, has crashed, or has a bug preventing it from ever clearing this finalizer.

If `describe` instead shows a normal-looking Pod with no finalizers, but it's still stuck, the more likely explanation is an unresponsive node — check:
```bash
kubectl get node <node-name>
```
and look at whether that node itself shows `Ready` or `NotReady`/`Unknown`.

## 7. Root-cause investigation

```bash
# is the underlying NODE actually healthy and reachable?
kubectl describe node <node-name> | grep -A5 Conditions

# does this Pod have a preStop hook that might be hanging?
kubectl get pod myapp-6b9f7c8d9f-x4k2p -n prod -o jsonpath='{.spec.containers[0].lifecycle}'
```

## 8. Fixes

- If a finalizer is the cause: identify and fix (or restart) whatever controller is supposed to clear it; as a last-resort, manual escape hatch, you can forcibly remove the finalizer yourself via `kubectl patch`, but this should be treated as a deliberate override of a safety mechanism, not a routine fix — whatever cleanup that finalizer was supposed to guarantee will now simply not happen
- If the node is unreachable: this becomes a node-level incident (per the failure-scenario material from the Architecture masterclass) — the Pod will typically resolve itself once the node either recovers or is confirmed dead and its Pods are forcibly removed by the control plane
- As a genuinely last-resort, immediate fix for an urgently stuck Pod:
```bash
kubectl delete pod myapp-6b9f7c8d9f-x4k2p -n prod --grace-period=0 --force
```
**This command deserves an explicit warning, not just a mention (per your rules on not glossing over risk):** `--force` does not actually confirm the container has stopped running anywhere — it only removes the Pod *object* from Kubernetes's own records. If the underlying node and container are actually still alive and functioning (as opposed to genuinely dead), you can end up in a state where the *real* container keeps running, completely invisible to Kubernetes, while a *replacement* Pod is simultaneously created elsewhere — for a stateful workload writing to shared storage, this is exactly the "split brain" scenario the StatefulSet masterclass warned about, where two processes could end up writing to the same data concurrently.

## 9. Verification

```bash
kubectl get pods -n prod -w
```
Confirm the Pod object is genuinely gone, and if it's managed by a controller, that a healthy replacement has appeared.

## 10. Prevention

- Ensure your application explicitly handles SIGTERM and exits promptly and cleanly when it receives it — a production coding practice, not a Kubernetes requirement, but one with direct operational consequences
- Set a `terminationGracePeriodSeconds` that realistically matches how long your application actually needs to shut down cleanly — neither so short that in-flight work gets cut off, nor so long that a genuinely stuck Pod takes an excessive amount of time to be forcefully removed
- Any custom controller that adds a finalizer must be built with real reliability guarantees, since a broken finalizer-owning controller can permanently strand objects

## 11. Production example

An application used a language runtime whose default behavior was to ignore SIGTERM entirely unless the developer explicitly registered a signal handler — nobody had done so. Every single Pod deletion, including completely routine rolling updates, took the full default 30-second grace period before being forcefully SIGKILLed, every time, needlessly slowing down every deployment. Adding a five-line signal handler that called the runtime's own graceful-shutdown function reduced typical Pod termination time from 30 seconds to under 2 seconds.

## 12. Interview question

**Q: A Pod has been stuck in "Terminating" for over an hour. Walk through your diagnostic priorities.**
A: First, check `metadata.finalizers` — a non-empty list points directly at an external controller failing to release the object, which is a very different problem from anything happening inside the Pod itself. Second, check the health of the node the Pod is running on — an unreachable node means the kubelet that would report successful shutdown simply isn't communicating at all. Only after ruling both of these out would I suspect the application itself is ignoring SIGTERM, which I'd confirm by checking whether `terminationGracePeriodSeconds` is being fully exhausted every time (a symptom of SIGTERM being ignored) versus the Pod hanging indefinitely with no grace-period pattern at all (more consistent with the finalizer or node explanations).

---

# 13–15. THE THREE PROBE FAILURES

Before covering each individually, you need the shared foundation, since all three probe types are variations on one mechanism, and understanding that mechanism first makes each individual failure make sense immediately rather than needing to be separately memorized.

## What is a "probe," fundamentally?

**Definition:** A probe is a periodic health check that the kubelet performs *against a container it's already running* — separate entirely from anything covered so far in this masterclass, which has all been about a container failing to even reach a running state. Probes exist to answer an ongoing question about a container that **is** running: *"is it actually healthy and doing its job correctly, right now?"*

**Why does Kubernetes need this at all?** A container process can be technically "running" — the operating system considers it an alive process — while being **completely useless**: stuck in an infinite loop, deadlocked waiting on a resource that will never arrive, or simply not finished starting up yet even though the process itself launched successfully. The operating system has no way to know any of this; it only knows "this process exists and hasn't exited." Probes are Kubernetes's way of asking a more meaningful question than the operating system can answer on its own.

## The three probe types, defined and distinguished — this is exactly a "commonly confused" set requiring a direct comparison table (rule 21)

| | Startup probe | Liveness probe | Readiness probe |
|---|---|---|---|
| Question it answers | "Has this container finished starting up yet?" | "Is this container still working correctly, ongoing?" | "Is this container currently able to handle traffic?" |
| When it runs | Only during startup, until it succeeds once | Continuously, for the container's entire life | Continuously, for the container's entire life |
| What happens on failure | Kubernetes keeps waiting (suppresses liveness/readiness checks until this succeeds) | The kubelet kills and restarts the container | The Pod is removed from Service traffic — **the container is NOT restarted** |
| Effect on RESTARTS count | None | Increases it | None at all |
| Effect on Service traffic | None directly (readiness gates this separately) | Brief interruption during the restart | Immediate — traffic stops going to this Pod |

**The single most important distinction in this whole table, worth restating on its own:** liveness probe failure **restarts** the container. Readiness probe failure **only removes it from traffic** — the container keeps running, completely undisturbed, just quietly ignored by Services until it starts passing readiness checks again. These are fundamentally different remedies for fundamentally different problems, and conflating them is one of the most common real mistakes people make when configuring probes.

## The full internal mechanism — WHAT is happening, mechanically, for any probe

```
┌───────────────────────────────────────────────────────────────┐
│  On a repeating timer (controlled by `periodSeconds`), the         │
│  KUBELET ITSELF — not a separate agent, not something running        │
│  inside the container — directly performs a check against the         │
│  container, using one of three possible methods:                        │
│                                                                             │
│    httpGet:  make an HTTP request to a path/port on the container's         │
│              own IP; a response in the 200-399 range counts as success        │
│                                                                                    │
│    exec:     run a specific command INSIDE the container's own filesystem/          │
│              namespace; an exit code of 0 counts as success                            │
│                                                                                             │
│    tcpSocket: attempt to open a TCP connection to a specific port; a               │
│               successful connection counts as success, regardless of                     │
│               what (if anything) is sent or received                                        │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**WHY does the kubelet do this directly, rather than some separate monitoring system?** Because the kubelet is already the component responsible for that container's entire lifecycle on that node — it's the natural, already-present owner of "should this container keep running." Introducing a separate system for this would mean two different components needing to coordinate about the same container's fate, which is exactly the kind of unnecessary indirection Kubernetes's architecture generally avoids (the same "no component talks directly to another unless it has to" principle from the Architecture masterclass, applied here as "don't introduce an extra component when the existing owner can do the job").

---

## 13. Readiness Probe Failures

## 1. What it means

The kubelet's readiness check against this container is failing — meaning the container itself may be perfectly alive and running, but Kubernetes has determined it should **not receive any traffic right now**.

## 2. What Kubernetes is doing internally

```
readinessProbe fails (per its own failureThreshold — the number of
consecutive failures required before it's considered truly failed,
not just a one-off blip)
                    │
                    ▼
kubelet marks this container's `ready` status as false
                    │
                    ▼
This Pod's overall `Ready` CONDITION becomes false
                    │
                    ▼
The Endpoint controller (a separate control loop, watching Pod
readiness across the cluster) notices this and removes this
Pod's IP from every Service's list of valid backend targets
                    │
                    ▼
From this moment on, NO Service will route traffic to this Pod —
but the container process itself is COMPLETELY UNTOUCHED; nothing
about it is killed, restarted, or otherwise disturbed
```

## 3. Common causes

- The application genuinely can't currently serve requests correctly (e.g., its own database connection pool is exhausted, or a downstream dependency it needs is temporarily unavailable)
- A misconfigured probe path — checking `/health` when the application actually serves its health endpoint at `/healthz`
- The probe's `timeoutSeconds` is too short for an application that's simply slow to respond under load, even though it's fundamentally fine

## 4. How to identify it

```bash
kubectl get pods -n prod
```
```
NAME                     READY   STATUS    RESTARTS   AGE
myapp-6b9f7c8d9f-x4k2p   0/1     Running   0          10m
```
**This is the exact signature to recognize immediately: `STATUS: Running`, but `READY: 0/1`, with `RESTARTS: 0`.** This combination is uniquely diagnostic of a readiness (not liveness) problem — the container is running and has never restarted, yet isn't marked ready.

## 5. Commands to run

```bash
kubectl describe pod myapp-6b9f7c8d9f-x4k2p -n prod
kubectl logs myapp-6b9f7c8d9f-x4k2p -n prod
```

## 6. How to interpret the output

```
Events:
  Warning  Unhealthy  30s (x8 over 4m)  kubelet  Readiness probe failed:
           HTTP probe failed with statuscode: 503
```
A `503` status code specifically often means the application itself is *reporting* that it's not ready (many frameworks return 503 deliberately from a health endpoint when an internal dependency check fails) — this is your application telling you something, not a Kubernetes-side misconfiguration, in this particular case. Contrast with a `404`, which would instead suggest the probe is checking the *wrong path entirely* — a Kubernetes-manifest-side configuration mistake.

## 7. Root-cause investigation

```bash
# what path/port is the probe actually configured to check?
kubectl get pod myapp-6b9f7c8d9f-x4k2p -n prod -o jsonpath='{.spec.containers[0].readinessProbe}'

# manually try the SAME check yourself, from inside the cluster
kubectl exec -it myapp-6b9f7c8d9f-x4k2p -n prod -- curl -v http://localhost:8080/readyz
```
Running the exact same check yourself, manually, is one of the most reliable ways to distinguish "the probe configuration is wrong" (you get a different result than expected) from "the application genuinely isn't ready" (you get the same failure the probe reported, and now need to investigate *why* the app itself is unhealthy).

## 8. Fixes

- Correct a misconfigured probe path/port to match what the application actually serves
- Fix the underlying application-level issue causing a legitimate readiness failure (e.g., its database connection pool problem)
- Loosen an overly aggressive `timeoutSeconds`/`failureThreshold` if the application is fundamentally healthy but occasionally slower than the probe currently tolerates

## 9. Verification

```bash
kubectl get pods -n prod -l app=myapp -w
```
Watch for `READY` to reach `1/1`.

## 10. Prevention

- Readiness checks should reflect *this specific Pod's* own ability to serve requests — a common, genuinely important design mistake (worth flagging explicitly as a misconception, rule 18) is writing a readiness check that transitively checks the health of *every downstream dependency* the app calls. **Misconception:** "my readiness probe should fail if any dependency is down, so Kubernetes stops sending it traffic." **Why this is often wrong in practice:** if one shared downstream dependency has a brief blip, this design causes **every single replica** to simultaneously fail readiness at once — removing 100% of your capacity from service, for a problem that a well-designed application might have handled with a retry or a fallback instead. A readiness probe reflecting only *this Pod's own* immediate capability to accept a request, not the transitive health of everything it talks to, is generally the safer design.

## 11. Production example

A team's readiness probe checked the health of a caching layer as part of its own `/readyz` logic. When that shared cache had a brief network blip, **every replica** of the affected service failed readiness simultaneously, at the same moment — taking the entire service offline from the Service's point of view, even though every individual application process was otherwise perfectly capable of handling requests (it would have simply had cache misses, a far less severe outcome than total unavailability).

## 12. Interview question

**Q: Why does a failing readiness probe never increase a Pod's RESTARTS count?**
A: Because readiness and liveness serve entirely different purposes, with entirely different remedies. A liveness failure means "this container is broken and needs to be restarted to have any chance of recovering" — restarting is the appropriate fix. A readiness failure means "this container is temporarily unable to serve traffic, but nothing suggests restarting it would help" — for a readiness failure genuinely caused by an external dependency being briefly unavailable, restarting the container would accomplish nothing at all, since the container itself was never broken in the first place; only removing it from traffic is the correct, proportionate response.

---

## 14. Liveness Probe Failures

## 1. What it means

The kubelet has determined this container is broken badly enough that restarting it is the appropriate remedy.

## 2. What Kubernetes is doing internally

```
livenessProbe fails repeatedly, reaching its configured
failureThreshold
                    │
                    ▼
kubelet concludes this container is unhealthy in a way that
restarting might fix
                    │
                    ▼
kubelet sends SIGTERM (then SIGKILL if needed, per the exact
same graceful-termination sequence covered in section 12 above
— a liveness-triggered restart uses the SAME shutdown mechanism
as any other Pod termination)
                    │
                    ▼
kubelet starts a NEW container process within the SAME Pod object
(same Pod name, same IP — this is the kubelet-local restart
mechanism, not a brand-new Pod being scheduled)
                    │
                    ▼
RESTARTS count increments by 1
```

## 3. Common causes

- A genuine application deadlock or hang — the process is technically alive but will never respond to anything again without outside intervention
- A `livenessProbe` configured too aggressively for an application with occasional, legitimate slow responses under real load (this causes healthy Pods to be needlessly restarted, which is itself a production problem, not a fix)
- No `startupProbe` configured for a genuinely slow-starting application, so the liveness probe begins checking (and failing, and triggering restarts) before the application has even finished its normal startup sequence

## 4. How to identify it

```bash
kubectl get pods -n prod
```
```
NAME                     READY   STATUS    RESTARTS   AGE
myapp-6b9f7c8d9f-x4k2p   1/1     Running   4          20m
```
**Compare this signature directly against the readiness failure's signature above:** here, `READY` shows `1/1` (it's currently passing readiness, at this exact moment you happened to check) but `RESTARTS` is climbing — this is the tell that liveness, not readiness, is the active problem, and it's likely cycling: failing liveness → restarting → briefly healthy → failing again.

## 5. Commands to run

```bash
kubectl describe pod myapp-6b9f7c8d9f-x4k2p -n prod
kubectl logs myapp-6b9f7c8d9f-x4k2p -n prod --previous
```

## 6. How to interpret the output

```
Events:
  Warning  Unhealthy  2m (x4 over 20m)  kubelet  Liveness probe failed:
           Get "http://10.244.1.5:8080/healthz": context deadline exceeded
  Normal   Killing    2m                kubelet  Container app failed
           liveness probe, will be restarted
```
`context deadline exceeded` specifically means the probe's own request **timed out** — the application didn't refuse the connection or return an error; it simply never responded within the configured `timeoutSeconds`. This is a distinct signature from an HTTP error status code, and points more toward "the app is hung or overloaded" than "the app is actively reporting a problem."

## 7. Root-cause investigation

```bash
# is this a GENUINE hang, or is the probe just too strict for real
# (but acceptable) response-time variance?
kubectl get pod myapp-6b9f7c8d9f-x4k2p -n prod -o jsonpath='{.spec.containers[0].livenessProbe}'

# check actual resource usage — is CPU throttling (a separate mechanism,
# covered in the Resource Limits guide) slowing the app down enough to
# miss the probe's timeout, without the app being genuinely broken?
kubectl top pod myapp-6b9f7c8d9f-x4k2p -n prod --containers
```

## 8. Fixes

- If it's a genuine application hang: this is fundamentally an application bug to fix in code — the liveness probe correctly did its job by forcing periodic recovery in the meantime, but recovery-via-restart is a mitigation, not a real fix
- If the probe is simply too strict: loosen `timeoutSeconds`/`periodSeconds`/`failureThreshold` to realistically match the application's real (acceptable) response time variance
- If startup time is the real issue: add a `startupProbe`, so liveness checks don't even begin until startup has genuinely completed

## 9. Verification

```bash
kubectl get pods -n prod -l app=myapp -w
```
Watch for `RESTARTS` to stop climbing.

## 10. Prevention

- A `livenessProbe` should check the **narrowest possible thing**: "is this process fundamentally still able to function at all," not "is every feature of this application currently working perfectly." An overly broad liveness check causes unnecessary restarts for problems a restart can't actually fix (a broken downstream dependency, for instance, is a readiness concern, not a liveness one — restarting the container does nothing to fix a *dependency's* problem).
- Always pair a `livenessProbe` with an appropriate `startupProbe` for anything with meaningfully slow startup, rather than trying to tune the liveness probe's own initial delay to accommodate both concerns at once.

## 11. Production example

An application's liveness probe checked a path that internally performed a full database query as part of its health check logic. Under a period of genuinely heavy (but otherwise normal) load, database queries slowed down just enough to occasionally exceed the probe's `timeoutSeconds` — causing the kubelet to restart otherwise-perfectly-healthy application processes during exactly the moments they were most needed (peak load), making a busy-but-functioning system worse, not better. The fix was making the liveness check a trivial, near-instant "is the process alive" check with no database dependency at all, while moving the database-health concern to the *readiness* probe instead, where a failure correctly meant "stop sending traffic here" rather than "restart this process."

## 12. Interview question

**Q: Why is checking a downstream dependency inside a liveness probe considered a design mistake?**
A: Because restarting the container does absolutely nothing to fix a downstream dependency's problem — if the database is down, restarting the application won't bring the database back up. Worse, if every replica shares the same liveness check logic and that same downstream dependency has a shared outage, every replica could be restarted simultaneously and repeatedly, for a problem restarting can never solve, potentially making an already-bad situation actively worse through unnecessary restart churn.

---

## 15. Startup Probe Failures

## 1. What it means

The container has not yet successfully passed its startup probe — meaning, from Kubernetes's perspective, the application has not yet finished its own initial startup sequence.

## 2. What Kubernetes is doing internally

```
Container process has started (the OS process exists)
                    │
                    ▼
Kubernetes does NOT yet begin running livenessProbe or
readinessProbe checks at all — they are entirely SUPPRESSED
                    │
                    ▼
Instead, ONLY the startupProbe runs, repeatedly, per its own
periodSeconds and failureThreshold
                    │
          ┌─────────┴─────────┐
          ▼                     ▼
   startupProbe SUCCEEDS   startupProbe keeps FAILING,
          │                 up to its OWN failureThreshold
          ▼                     │
liveness/readiness probes        ▼
now BEGIN running normally   kubelet treats this as a startup
for the first time             FAILURE — the container is killed
                                and restarted (same underlying
                                mechanism as a liveness failure)
```

**WHY does this probe type exist at all, given liveness and readiness already existed first?** Before `startupProbe` was introduced, people faced a genuine dilemma: a `livenessProbe`'s own `initialDelaySeconds` (a fixed wait before checks begin at all) had to be set long enough to cover the *worst-case* startup time — but that same long delay would then also apply to detecting a genuine hang *after* startup, meaning a real deadlock might go undetected for that same long duration, unnecessarily. `startupProbe` decouples these two concerns entirely: it can be configured with a very patient, generous allowance specifically for one-time startup, while the ongoing `livenessProbe` can then be configured with a much tighter, more responsive `periodSeconds`/`failureThreshold` for detecting genuine problems *after* startup completes — since it only starts counting once startup is confirmed done.

## 3. Common causes

- An application with a genuinely long, unavoidable startup sequence (loading a large in-memory model, running startup migrations, warming a cache) that exceeds the startup probe's total allowed time (`failureThreshold × periodSeconds`)
- A startup probe configured to check the *wrong* endpoint — one that doesn't exist yet until much later in the startup sequence than intended
- An application that's actually failing to start at all (an unrelated CrashLoopBackOff-style problem), which will also show up as repeated startup probe failures, since there's nothing yet to successfully check

## 4. How to identify it

```bash
kubectl get pods -n prod
```
```
NAME                     READY   STATUS    RESTARTS   AGE
myapp-6b9f7c8d9f-x4k2p   0/1     Running   2          3m
```
This can look superficially similar to a liveness-failure pattern (climbing restarts) — the key distinguishing detail is in the Events themselves, specifically mentioning "Startup" rather than "Liveness."

## 5. Commands to run

```bash
kubectl describe pod myapp-6b9f7c8d9f-x4k2p -n prod
```

## 6. How to interpret the output

```
Events:
  Warning  Unhealthy  10s (x15 over 90s)  kubelet  Startup probe failed:
           HTTP probe failed with statuscode: 000
  Normal   Killing    5s                   kubelet  Container app failed
           startup probe, will be restarted
```
A status code of `000` here specifically means the connection couldn't be established at all — nothing was listening on that port yet — which is entirely expected and normal for the *early* part of a startup sequence; the concerning part is if this repeats all the way to `failureThreshold`, meaning the application never got to the point of listening on that port within the total allowed time.

## 7. Root-cause investigation

```bash
# how long is the startup probe actually allowing, in total?
# (failureThreshold × periodSeconds gives you the effective total window)
kubectl get pod myapp-6b9f7c8d9f-x4k2p -n prod -o jsonpath='{.spec.containers[0].startupProbe}'

# how long does the app ACTUALLY take to become ready, observed directly?
kubectl logs myapp-6b9f7c8d9f-x4k2p -n prod --previous --timestamps
# compare the timestamp of the container's first log line against the
# timestamp of whatever log line indicates "I am now ready to serve requests"
```

## 8. Fixes

- Increase the startup probe's total allowed window (`failureThreshold`, `periodSeconds`, or both) to comfortably exceed the application's genuine, real-world startup time, with reasonable headroom
- Correct the probe's target path/port if it's checking something that doesn't exist until later in the startup sequence than currently configured
- If the app is genuinely failing to start (not just being slow), this reduces to the same investigation as CrashLoopBackOff (Part 1) — the startup probe failure is a symptom pointing at that underlying, separate problem

## 9. Verification

```bash
kubectl get pods -n prod -l app=myapp -w
```
Watch for the Pod to transition cleanly to `READY 1/1` without repeated restarts along the way.

## 10. Prevention

- Set the startup probe's allowed window based on **measured, real startup time under realistic conditions** (including a cold cache, not just a warm developer machine), with meaningful headroom — a production practice, not a Kubernetes-enforced rule
- Whenever an application's startup time changes meaningfully (a new migration step is added, a larger model is now loaded at boot), revisit the startup probe's configuration as part of that same change, rather than treating it as a one-time setting from months or years earlier

## 11. Production example

An application began loading a significantly larger machine-learning model into memory at startup as part of a feature update, increasing real startup time from roughly 10 seconds to over 90 seconds. The existing `startupProbe` was still configured for the old, shorter startup profile, so every single replica was killed and restarted right as it was on the verge of actually finishing startup — an endless cycle where the application never once got the chance to fully start, because it was always killed moments before crossing the finish line. The fix was simply widening the startup probe's total allowed window to match the new, larger, measured startup time.

## 12. Interview question

**Q: Why does a startupProbe suppress the liveness probe, rather than the two running side by side from the very beginning?**
A: Because if both ran from the very first moment the container process existed, the liveness probe would immediately start failing during the application's own normal startup sequence (before it's listening on any port at all) — and the kubelet would restart the container over and over, with the application never getting a real chance to finish starting, since it keeps getting killed mid-startup. Suppressing liveness (and readiness) checks entirely until the startup probe has confirmed startup is genuinely complete is what allows a patient, generous startup allowance to coexist with a tight, responsive liveness check for detecting problems *after* that point.

---

## Beginner summary (Part 3)

- `RunContainerError` means the operating system itself failed to launch the container's process — distinct from CrashLoopBackOff, where the process does launch and then exits.
- A "stuck Terminating" Pod is one that's been marked for deletion but hasn't finished the multi-step shutdown sequence — check for finalizers and node health as the two leading suspects.
- Readiness, liveness, and startup probes each answer a different question: "can this take traffic right now," "is this fundamentally still alive," and "has this finished starting up," respectively — and each failure has a genuinely different consequence (traffic removal, restart, or suppressed checking).

## Advanced summary (Part 3)

- The Config/Runtime/Run progression (`CreateContainerConfigError` → `CreateContainerError` → `RunContainerError`) maps precisely onto three sequential kubelet/runtime responsibilities: assembling configuration, constructing the container, and launching its process — each failure isolates exactly which of these three stages broke.
- Finalizers introduce genuine, deliberate coupling between an object's deletion and an external controller's own logic — a stuck Terminating Pod with a populated `finalizers` field is never a Pod-layer problem at all; it's always a controller-layer one.
- `startupProbe` exists specifically to decouple "how long is acceptable for one-time startup" from "how quickly should an ongoing hang be detected" — two genuinely different tuning concerns that, before this probe type existed, had to be awkwardly compromised into a single `initialDelaySeconds` setting.

---

Say **"continue"** for Part 4, the final part: the master troubleshooting decision tree tying all fifteen failures together, broken YAML diagnostic exercises across the full set, consolidated Minikube labs, and the closing reference material.

# Kubernetes Pod Troubleshooting — Complete Masterclass
## Part 4 of 4 (Final): Master Decision Tree, Broken YAML Diagnostics, Consolidated Labs & Closing Reference

*(This closes out all 15 failure modes from Parts 1–3. Before the decision tree itself, one foundational idea ties everything in this masterclass together — worth stating explicitly, since every individual failure you've just learned is actually a special case of it.)*

---

# THE ONE IDEA UNDERNEATH ALL 15 FAILURES

**Every single failure in this masterclass is Kubernetes reporting: "I got exactly this far, and no further."**

```
Pod accepted by API server
        │
        ▼
Scheduled to a node ──────────────────── FAILS HERE → Pending
        │
        ▼
Container configuration assembled ────── FAILS HERE → CreateContainerConfigError
        │
        ▼
Container image available ────────────── FAILS HERE → ImagePullBackOff / ErrImagePull
        │
        ▼
Volumes attached and mounted ─────────── FAILS HERE → FailedMount
        │
        ▼
Container constructed by the runtime ─── FAILS HERE → CreateContainerError
        │
        ▼
Container process launched ───────────── FAILS HERE → RunContainerError
        │
        ▼
Container process stays running ──────── FAILS HERE → CrashLoopBackOff / OOMKilled
        │
        ▼
Startup probe passes ──────────────────── FAILS HERE → Startup probe failure
        │
        ▼
Liveness probe keeps passing, ongoing ─── FAILS HERE → Liveness probe failure
        │
        ▼
Readiness probe passes, ongoing ───────── FAILS HERE → Readiness probe failure
        │
        ▼
              Pod fully healthy and serving traffic
```

**This is why "which command do I run first" always has the same answer regardless of which failure you're facing:** `kubectl describe pod` tells you exactly how far down this ladder the Pod actually got, because each stage produces a distinctly different Events message. You are never guessing which stage failed — Kubernetes always tells you, in the Events section, precisely.

**Two failures don't fit neatly into this single ladder, because they're not about a Pod failing to *start* at all — worth calling this out explicitly as an edge case (rule 23):** `Evicted` and stuck `Terminating` both happen to Pods that may have been **perfectly healthy**, all the way at the bottom of this ladder — they're triggered by forces external to the Pod's own startup sequence (node resource pressure, or a deletion request), not by the Pod failing to progress through it.

---

# THE MASTER TROUBLESHOOTING DECISION TREE

```
                          ┌─────────────────────────┐
                          │   kubectl get pods         │
                          │   what does STATUS show?    │
                          └────────────┬────────────┘
                                        │
      ┌──────────────┬─────────────────┼──────────────────┬─────────────────┐
      ▼              ▼                 ▼                  ▼                 ▼
  "Pending"    "ImagePullBackOff"  "CreateContainer*"  "ContainerCreating" "Running"
      │          or "ErrImagePull"       │             (stuck)               │
      ▼                 │                │                  │            (but READY
Check Events:            ▼                ▼                  ▼             shows 0/1,
"FailedScheduling"   Check Events for   Config error?      Check Events    or RESTARTS
names the EXACT      exact image name/  → missing/wrong    for WHICH        climbing)
reason: insufficient  registry error     ConfigMap/Secret   sub-step:            │
resources? taint      → fix name, add    key                - image pull         ▼
mismatch? unbound     imagePullSecrets,  Runtime error?     - sandbox/CNI    Check READY
PVC?                  or fix registry    → check volume     - volume mount   column vs
                       auth               mount path         (→ FailedMount   RESTARTS:
                                          conflicts, node     branch)          │
                                          runtime health                 ┌────┴────┐
                                                                          ▼          ▼
                                                                    READY 0/1,   READY 1/1,
                                                                    RESTARTS 0   RESTARTS
                                                                    → READINESS  climbing
                                                                    probe issue  → LIVENESS
                                                                    (traffic      probe issue
                                                                     removed,     (being
                                                                     container    restarted)
                                                                     untouched)        │
                                                                                       ▼
                                                                                 Check if this
                                                                                 happened during
                                                                                 STARTUP window
                                                                                 → if so, likely
                                                                                 STARTUP probe,
                                                                                 not liveness

      ┌──────────────────────┐          ┌──────────────────────┐
      ▼                      ▼          ▼                      ▼
 "CrashLoopBackOff"      "OOMKilled"  "Evicted"           "Terminating"
      │                      │          │                 (stuck, past
      ▼                      ▼          ▼                  grace period)
kubectl logs           Reason:      Reason: Evicted             │
--previous is your      OOMKilled,   in describe output           ▼
FIRST move, always      exit 137     names the EXACT          Check
                                     resource (memory/         metadata.
                                     disk) and usage vs         finalizers
                                     request                    first, THEN
                                                                 node health
```

**How to use this tree in practice:** start at the top with `kubectl get pods`, follow the branch matching what you see, and at every branch point, the next command is always the same: `kubectl describe pod`. This tree isn't something to memorize as a flowchart — it's a compressed map of the ladder diagram above, and once the ladder itself makes sense, this tree is just "where on the ladder did it stop."

---

# BROKEN YAML DIAGNOSTIC EXERCISES

Six broken manifests follow. For each, don't apply it yet — first tell me: (1) which specific failure from this masterclass you predict will occur, (2) exactly which line is wrong and why, and (3) what `kubectl describe pod` Events message you expect to see. Then apply it in Minikube and check yourself.

## Exercise A

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-a
spec:
  containers:
    - name: app
      image: nginx:1.25
      env:
        - name: GREETING
          valueFrom:
            configMapKeyRef:
              name: greeting-config
              key: MESSAGE
```
*(No ConfigMap named `greeting-config` exists anywhere in the cluster.)*

## Exercise B

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-b
spec:
  containers:
    - name: app
      image: myregistry.internal/myapp:2.5.1
```
*(This image genuinely does not exist at that tag in that registry — a typo'd version bump.)*

## Exercise C

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-c
spec:
  containers:
    - name: app
      image: busybox
      command: ["/nonexistent-script.sh"]
```

## Exercise D

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-d
spec:
  containers:
    - name: hog
      image: polinux/stress
      resources:
        limits:
          memory: "50Mi"
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
```

## Exercise E

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-e
spec:
  containers:
    - name: app
      image: nginx:1.25
      readinessProbe:
        httpGet:
          path: /this-path-returns-404
          port: 80
        periodSeconds: 5
        failureThreshold: 2
```

## Exercise F

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-f
spec:
  containers:
    - name: app
      image: nginx:1.25
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 0
        periodSeconds: 3
        timeoutSeconds: 1
        failureThreshold: 1
      resources:
        limits:
          cpu: "10m"
```
*(Hint for F specifically: an extremely tight CPU limit combined with `failureThreshold: 1` and a very short `timeoutSeconds` — think about CPU throttling, covered in the Resource Limits guide, interacting with this probe's tolerance.)*

---

# CONSOLIDATED MINIKUBE LAB SEQUENCE

Run these in order — each intentionally creates one of the fifteen failures, so you experience the actual Events output firsthand rather than only reading about it.

```bash
minikube start --cpus=4 --memory=4096
kubectl create namespace troubleshoot-lab
kubectl config set-context --current --namespace=troubleshoot-lab
```

### Lab 1 — CrashLoopBackOff
```bash
kubectl run lab1 --image=busybox --restart=Always -- sh -c "echo dying now; exit 1"
kubectl get pods -w
kubectl logs lab1 --previous
```
**What to notice:** the exit code, and how quickly `--previous` gives you the answer versus staring at the current (likely empty) logs.

### Lab 2 — ImagePullBackOff
```bash
kubectl run lab2 --image=busybox:this-tag-does-not-exist
kubectl describe pod lab2 | grep -A3 Events
```

### Lab 3 — Pending (unschedulable)
```bash
kubectl run lab3 --image=nginx --requests='cpu=999'
kubectl describe pod lab3 | grep -A3 Events
```

### Lab 4 — OOMKilled
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: lab4 }
spec:
  containers:
    - name: hog
      image: polinux/stress
      resources: { limits: { memory: "50Mi" } }
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
EOF
kubectl get pod lab4 -w
kubectl describe pod lab4 | grep -A5 "Last State"
```

### Lab 5 — CreateContainerConfigError
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: lab5 }
spec:
  containers:
    - name: app
      image: nginx
      envFrom:
        - secretRef: { name: does-not-exist }
EOF
kubectl describe pod lab5 | grep -A3 Events
```

### Lab 6 — Readiness probe failure (vs liveness — deliberately compare both)
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: lab6-readiness }
spec:
  containers:
    - name: app
      image: nginx
      readinessProbe:
        httpGet: { path: /nonexistent, port: 80 }
        periodSeconds: 3
EOF
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: lab6-liveness }
spec:
  containers:
    - name: app
      image: nginx
      livenessProbe:
        httpGet: { path: /nonexistent, port: 80 }
        periodSeconds: 3
        failureThreshold: 2
EOF
kubectl get pods -w
```
**What to notice, directly comparing the two:** `lab6-readiness` sits at `READY 0/1`, `RESTARTS 0`, forever. `lab6-liveness` shows `RESTARTS` climbing steadily. Same broken probe path, genuinely different consequence — this is the single clearest hands-on demonstration of the whole Part 3 comparison table.

### Lab 7 — Stuck Terminating (simulated via finalizer)
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: lab7
  finalizers:
    - troubleshoot-lab.example.com/never-cleared
spec:
  containers:
    - name: app
      image: nginx
EOF
kubectl delete pod lab7 --wait=false
kubectl get pod lab7
# STAYS "Terminating" indefinitely — nothing will ever clear this finalizer
kubectl get pod lab7 -o jsonpath='{.metadata.finalizers}'
# manual escape hatch, since this masterclass created the stuck state deliberately:
kubectl patch pod lab7 -p '{"metadata":{"finalizers":[]}}' --type=merge
```

### Lab 8 — Evicted (simulated, conceptual on Minikube's single node)
```bash
# genuine eviction requires real node memory pressure, harder to force safely
# on a shared Minikube VM — instead, inspect a REAL example's anatomy:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: { name: lab8 }
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests: { memory: "8Gi" }   # deliberately larger than minikube's node
EOF
kubectl describe pod lab8 | grep -A3 Events
# this actually produces Pending/FailedScheduling rather than Evicted,
# which is itself a useful lesson: eviction happens to ALREADY-RUNNING
# Pods under pressure; a Pod that never got scheduled in the first place
# can't be evicted — it was never running to begin with
```

---

# TROUBLESHOOTING CHECKLIST (complete, all 15 failures)

```
□ kubectl get pods -o wide  — establish STATUS, READY ratio, RESTARTS, and NODE
□ kubectl describe pod <name>  — read the ENTIRE Events section, oldest to newest
□ Match the STATUS/Events combination against the ladder diagram at the top
  of this Part — which stage did it fail at?
□ If CrashLoopBackOff: kubectl logs --previous BEFORE anything else
□ If READY shows 0/1 with RESTARTS 0: this is READINESS, not liveness —
  the container is untouched, only traffic is affected
□ If RESTARTS is climbing with READY showing 1/1 intermittently: this is
  LIVENESS — the container is being killed and restarted
□ If STATUS is "Terminating" past its terminationGracePeriodSeconds:
  check metadata.finalizers FIRST, node health SECOND
□ If STATUS is "Evicted": read the Message field directly — it names the
  exact resource and usage vs request; this is your root cause already
□ If ContainerCreating is stuck: describe's Events tells you WHICH sub-step
  (sandbox/CNI, image pull, or volume mount) — never guess
□ Always distinguish: did the container ever actually START (CrashLoopBackOff,
  probes) vs did it NEVER start at all (everything from CreateContainerConfigError
  through RunContainerError)? This single question routes you to completely
  different investigation paths.
```

---

# COMMON MISTAKES (across the whole masterclass)

- Running `kubectl logs` before `kubectl describe pod` — Events almost always narrows the problem faster, and for anything before "container running," logs will simply be empty or unavailable
- Assuming `RESTARTS: 0` means "nothing is wrong" — a Pod stuck at `READY 0/1` with zero restarts is very much broken, just via the readiness path, not the crash path
- Treating "Terminating" as a Pod-level problem by default, when a populated `finalizers` field means the real problem is an entirely separate, external controller
- Writing a liveness probe that checks downstream dependencies — the single most repeated design mistake across real production incidents in this space
- Force-deleting a stuck Pod (`--grace-period=0 --force`) as a first resort rather than a last one, without first confirming the underlying node/container is actually dead — risking two copies of a stateful workload running simultaneously against the same data

---

# 20 RAPID-FIRE QUESTIONS

1. What's the first command you run on any unhealthy Pod? — `kubectl describe pod`
2. What's the second, for a CrashLoopBackOff specifically? — `kubectl logs --previous`
3. Exit code 137 means what? — OOMKilled (128 + SIGKILL's signal number 9)
4. `READY 0/1`, `RESTARTS 0` — which probe? — Readiness
5. `RESTARTS` climbing, `READY` intermittently `1/1` — which probe? — Liveness
6. Does a failing readiness probe restart the container? — No, never
7. What field suppresses liveness/readiness checks during startup? — `startupProbe`
8. What's the difference between CreateContainerConfigError and CreateContainerError? — one is the kubelet's own config assembly failing; the other is the container runtime failing to construct the container
9. What Pod field, if populated, explains a Pod stuck Terminating indefinitely? — `metadata.finalizers`
10. What's sent first in a graceful termination — SIGTERM or SIGKILL? — SIGTERM
11. Default `terminationGracePeriodSeconds`? — 30
12. Does OOMKill affect the whole node, or just one container? — Just that container's own cgroup
13. Does Eviction affect one container, or can it remove an entire Pod? — The entire Pod
14. What Pod status literally never appears in `kubectl get pods` output, only in `describe`'s Events? — FailedMount (it's an event reason, not a status)
15. What does `envFrom` silently skip? — Keys that aren't valid environment variable names
16. Why do rolling updates never proceed past a stuck new ReplicaSet's Pods? — Readiness gating — an old Pod is never removed until its replacement passes readiness
17. What command lets you manually reproduce a readiness probe's exact check? — `kubectl exec ... -- curl <path>` (or equivalent)
18. Base architecture mismatch (arm64 image, amd64 node) produces which failure? — RunContainerError
19. Is an Evicted Pod automatically deleted from `kubectl get pods`? — No, it lingers until manually cleaned up
20. What single diagram idea underlies literally all 15 failures in this masterclass? — Kubernetes reporting exactly how far a Pod progressed through its startup/health ladder before something stopped it

---

# MENTAL MODEL — THE ONE PARAGRAPH TO CARRY FORWARD

Every Pod failure in Kubernetes is a report of *where on the ladder* things stopped — scheduling, config assembly, image pull, volume mount, container construction, process launch, staying alive, or staying ready — and `kubectl describe pod`'s Events section always tells you which rung you're stuck on, in plain language, without you needing to guess. The only genuinely separate branch is anything triggered *after* a Pod is already healthy and running: Eviction (the node needed the resources back) and stuck Terminating (something external — a finalizer or an unreachable node — is blocking a deletion that was otherwise already in motion). Once you can place any symptom onto this one ladder-plus-two-branches picture, you've replaced fifteen things to memorize with one thing to reason from.

---

That closes the full Pod Troubleshooting Masterclass — all 15 failure modes, the master decision tree, six broken YAMLs for you to diagnose, eight hands-on labs, and the closing reference material.

Whenever you're ready, we can go back to the **ConfigMaps & Secrets Masterclass** (Part 2: Secrets from first principles was next), or continue wherever else you'd like.


# Kubernetes Deployment Strategies — Complete Mastery Guide
### From Beginner Rolling Updates to Production Progressive Delivery

---

# PART 1 — THE TWO NATIVE STRATEGIES

## 1–2. Recreate vs RollingUpdate

Every Deployment has exactly one of two native `.spec.strategy.type` values:

```yaml
spec:
  strategy:
    type: Recreate           # OR: RollingUpdate (the default)
```

| | Recreate | RollingUpdate |
|---|---|---|
| Old Pods | ALL terminated first | Terminated gradually, interleaved with new ones |
| New Pods | Created only after old ones are fully gone | Created alongside remaining old ones |
| Downtime | Yes — a gap with **zero** running Pods | No — some version is always serving, if configured correctly |
| Use case | Workloads that CANNOT have two versions running simultaneously (e.g., a schema-incompatible singleton, some legacy apps holding an exclusive lock/port) | The overwhelming majority of stateless workloads |

## 3. Rolling updates — mechanics recap

*(Full internal sequence diagram already covered in the Architecture masterclass, Part 5, Diagram F — this guide builds forward from that mechanical foundation into the fields and failure handling around it.)*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels: { app: my-app }
  template:
    metadata: { labels: { app: my-app } }
    spec:
      containers:
        - name: app
          image: myapp:v1
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 5
```

---

# PART 2 — THE EXACT v1 → v2 WALKTHROUGH, 5 REPLICAS

## Setup
```
maxSurge: 1          → at most 1 EXTRA Pod above desired count (6 total max, momentarily)
maxUnavailable: 0     → NEVER fewer than 5 Pods available at once
```

```
Step 0 (before update):
  ReplicaSet-v1: [v1][v1][v1][v1][v1]   (5/5 Ready)

Step 1: user runs `kubectl set image deployment/my-app app=myapp:v2`
  Deployment controller creates ReplicaSet-v2 (0 replicas so far)

Step 2: maxSurge allows +1 → scale ReplicaSet-v2 to 1
  ReplicaSet-v1: [v1][v1][v1][v1][v1]   (5 Ready)
  ReplicaSet-v2: [v2:starting]           (0 Ready yet — total: 6 Pods, 5 Ready)

Step 3: new v2 Pod passes its readinessProbe → becomes Ready
  ReplicaSet-v1: [v1][v1][v1][v1][v1]   (5 Ready)
  ReplicaSet-v2: [v2:Ready]              (1 Ready — total: 6 Pods, 6 Ready)

Step 4: NOW it's safe to remove an old Pod (maxUnavailable: 0 requires
        staying at/above 5 Ready at all times — we're at 6, so removing
        1 keeps us at 5)
  ReplicaSet-v1: [v1][v1][v1][v1]        (4 Ready)
  ReplicaSet-v2: [v2:Ready]               (1 Ready — total: 5 Pods, 5 Ready)

Step 5: maxSurge allows +1 again → scale ReplicaSet-v2 up again
  ReplicaSet-v1: [v1][v1][v1][v1]        (4 Ready)
  ReplicaSet-v2: [v2:Ready][v2:starting]  (1 Ready, 1 starting — total: 6 Pods, 5 Ready)

... this cycle repeats: wait for new Pod Ready → remove one old Pod →
    surge one more new Pod → wait for Ready → remove another old Pod ...

Step N (final):
  ReplicaSet-v1: []                        (0 — scaled to zero, KEPT for rollback)
  ReplicaSet-v2: [v2][v2][v2][v2][v2]       (5/5 Ready)
```

## The single gating rule, stated once, precisely
**A step that removes an old Pod never happens until the replacement new Pod has passed its `readinessProbe`** — not merely started, not merely `Running`. This is the entire mechanism behind "zero-downtime" rolling updates: total *available* capacity (Ready Pods, old + new combined) never drops below `desired - maxUnavailable`, at any single instant during the whole rollout.

```bash
kubectl rollout status deployment/my-app
# Waiting for deployment "my-app" rollout to finish: 2 out of 5 new replicas have been updated...
kubectl get replicasets -l app=my-app
# NAME             DESIRED   CURRENT   READY   AGE
# my-app-7d4f9c     0         0         0       10m    (old, v1, kept for rollback)
# my-app-8b2e1a     5         5         5       2m     (new, v2, active)
```

---

# PART 3 — maxSurge AND maxUnavailable IN DEPTH

## What each controls, precisely

```yaml
rollingUpdate:
  maxSurge: 25%          # can also be a percentage — rounds UP
  maxUnavailable: 25%     # rounds DOWN
```
With 5 replicas: `maxSurge: 25%` → `ceil(5 × 0.25) = 2` extra Pods allowed; `maxUnavailable: 25%` → `floor(5 × 0.25) = 1` Pod allowed to be missing at once. **The rounding direction is deliberate and asymmetric** — surge rounds up (favor having more capacity available during rollout), unavailable rounds down (favor not sacrificing more capacity than strictly necessary) — both choices lean toward preserving availability during the transition.

## Four meaningfully different configurations

| maxSurge | maxUnavailable | Behavior |
|---|---|---|
| `1` | `0` | Slowest, safest — always at full capacity or above, one extra Pod at a time (the v1→v2 walkthrough above) |
| `0` | `1` | Never exceeds desired count, but capacity dips below full during rollout — useful when the cluster has zero spare resource headroom to surge into |
| `25%` | `25%` | Faster rollout, some transient over-capacity AND some transient under-capacity simultaneously |
| `100%` | `0` | Full blue-green-like behavior WITHIN a single Deployment — doubles capacity momentarily, then cuts over entirely once all new Pods are Ready |

```
maxSurge: 0, maxUnavailable: 1  — the OPPOSITE shape from Part 2's walkthrough:

Step 0: [v1][v1][v1][v1][v1]           (5 Ready)
Step 1: REMOVE one old Pod FIRST (no room to surge — maxSurge: 0)
        [v1][v1][v1][v1]                (4 Ready — capacity DIPPED below 5)
Step 2: create a v2 Pod to replace it, wait for Ready
        [v1][v1][v1][v1][v2:Ready]       (5 Ready again)
Step 3: repeat — remove another old Pod BEFORE creating its replacement
```
This variant is appropriate specifically when the cluster genuinely cannot spare the resources for even one surged Pod — the trade-off is accepting brief reduced capacity instead.

---

# PART 4 — REVISIONS, ROLLOUT HISTORY, AND ROLLBACK

## 7–8. Deployment revisions and rollout history

Every time `.spec.template` changes, the Deployment controller creates (or reuses, if it matches an existing one exactly) a ReplicaSet and increments a revision counter, tracked via an annotation.

```bash
kubectl rollout history deployment/my-app
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         kubectl set image deployment/my-app app=myapp:v2
# 3         kubectl set image deployment/my-app app=myapp:v3
```
```bash
# to get meaningful CHANGE-CAUSE entries, annotate deliberately:
kubectl apply -f deployment.yaml --record          # deprecated but still common
# OR set it explicitly:
kubectl annotate deployment/my-app kubernetes.io/change-cause="Bump to v3 for security patch"
```

## 9. rollout status
```bash
kubectl rollout status deployment/my-app
# blocks and streams progress until the rollout completes or fails —
# exactly what a CI/CD pipeline step should wait on before proceeding
kubectl rollout status deployment/my-app --timeout=120s
```

## 10. rollout undo
```bash
kubectl rollout undo deployment/my-app                     # back to the PREVIOUS revision
kubectl rollout undo deployment/my-app --to-revision=1       # back to a SPECIFIC revision
```
**Mechanically, this is nothing new** — per the ReplicaSets guide, the old ReplicaSet was never deleted, only scaled to 0. `rollout undo` simply runs the exact same rolling-update machinery from Part 2, **in reverse**: scale the old (target) ReplicaSet back up, scale the current one down, gated by the same readiness checks — which is precisely why rollbacks are typically much faster than the original rollout: no new image pull is needed if nodes still have it cached, and the ReplicaSet's Pod template is already fully formed.

```
Before undo:  RS-v2 (5 Ready) ── RS-v1 (0, kept)
During undo:  RS-v2 scaling DOWN ── RS-v1 scaling UP (same maxSurge/maxUnavailable rules apply)
After undo:   RS-v2 (0, now kept for a POSSIBLE redo) ── RS-v1 (5 Ready)
```

## 11. Pause and resume — controlled, inspectable rollouts
```bash
kubectl rollout pause deployment/my-app
kubectl set image deployment/my-app app=myapp:v2      # change is recorded but NOT rolled out yet
# ... inspect, run manual checks, wait for a maintenance window ...
kubectl rollout resume deployment/my-app                # NOW the rolling update actually proceeds
```
Pausing is also a legitimate way to batch multiple changes (image + env vars + resource limits) into a **single** rollout instead of triggering a separate rolling update for each individual `kubectl set`/`kubectl patch` call.

---

# PART 5 — FAILURE HANDLING DURING ROLLOUTS

## 13–15. Failed deployments, bad images, and failed probes

**The critical safety property:** a rollout that produces Pods failing their readiness probe **simply stalls** — it does not proceed to remove more old Pods, and it does not roll back automatically either. It just... stops making progress, holding whatever mix of old/new Pods it had reached.

```bash
kubectl rollout status deployment/my-app
# Waiting for deployment "my-app" rollout to finish: 2 out of 5 new replicas
# have been updated...
# (this can hang here INDEFINITELY if the new Pods never become Ready)
```
```yaml
spec:
  progressDeadlineSeconds: 600    # after 10 minutes with no progress, mark the
                                    # Deployment condition as "ProgressDeadlineExceeded"
```
**`progressDeadlineSeconds` does not trigger an automatic rollback** — it only flips a `.status.conditions` entry to signal failure, for monitoring/alerting to catch. Kubernetes deliberately does not auto-rollback on its own; a human (or a CI/CD pipeline's own explicit logic) must decide to run `kubectl rollout undo`.

```bash
kubectl get deployment my-app -o jsonpath='{.status.conditions}'
# type: Progressing, status: "False", reason: ProgressDeadlineExceeded
```

## Diagram: a stuck rollout, frozen mid-transition
```
ReplicaSet-v1: [v1][v1][v1][v1]         (4 Ready — old version, still serving real traffic)
ReplicaSet-v2: [v2:CrashLoopBackOff]     (0 Ready — broken new version, stuck forever)

Total available: 4/5 desired — BELOW full capacity, but the Service
still routes ONLY to the 4 Ready v1 Pods (readiness-gated, per the
Services guide) — users experience reduced capacity, NOT v2's bugs,
because v2 never became Ready enough to receive any traffic at all
```
**This is precisely why readiness probes are the single most important safety mechanism in this entire guide** — a genuinely broken v2 image can sit there failing forever without ever serving a single real user request, because `Ready: False` (Pods masterclass §6–9) keeps it out of the Service's EndpointSlice entirely.

## 16. Backward compatibility
Because old and new versions run **simultaneously** during any rolling update (by design, per Part 2), **the new version must be able to coexist with the old one** for the rollout's duration — this has real implications beyond just the application code itself:
- **Database schema changes** must be backward-compatible with the *previous* application version for the rollout's duration (additive migrations — add a column, don't rename/drop one in the same deploy that also changes code to use the new name)
- **API contracts** between services must tolerate both old and new callers/responders simultaneously
- This is precisely why "expand-contract" migration patterns exist: expand the schema (additive, safe with both versions) in one deploy, migrate code to use it, THEN contract (remove old columns/fields) in a later, separate deploy once the old version is completely gone

---

# PART 6 — ADVANCED STRATEGIES (CONCEPTUAL — BEYOND NATIVE DEPLOYMENT OBJECTS)

**None of the following are native Kubernetes Deployment features** — they require either manual orchestration, a service mesh, or a dedicated progressive-delivery controller (Argo Rollouts, Flagger). This section explains the *concept* and *which components participate*, since a Deployment object alone cannot express any of them directly.

## Blue-Green

```
┌─────────────────────┐         ┌─────────────────────┐
│   Deployment "blue"    │         │   Deployment "green"    │
│   (v1, currently LIVE)  │         │   (v2, fully deployed,   │
│   5/5 Ready               │         │    fully tested, but      │
│                             │         │    receiving ZERO          │
│                             │         │    real traffic yet)        │
└───────────┬───────────┘         └───────────┬───────────┘
             │                                    │
             └───────────────┬────────────────────┘
                              ▼
                   ┌────────────────────┐
                   │   Service selector    │  ← currently points at "blue"
                   │   (app: blue)           │
                   └────────────────────┘

CUTOVER: instantly flip the Service's selector to "app: green"
         → ALL traffic switches to v2 in one atomic step, no gradual
           transition, no mixed-version window at all
```
**Which components participate:** two **complete, independent** Deployments (both at full replica count simultaneously — this costs 2x the steady-state resources during the transition window) and a single **Service** whose `selector` is edited to cut over. No Kubernetes-native object models "blue-green" directly — it's just two Deployments plus one deliberate Service edit, usually scripted/automated by a CI/CD pipeline or a tool like Argo Rollouts, which can manage this pattern declaratively.

**Rollback:** instant — just flip the Service selector back to "blue." This is blue-green's headline advantage over rolling updates: rollback has zero propagation delay, since the old version's full-capacity Deployment never stopped running.

## Canary

```
┌─────────────────────┐         ┌─────────────────────┐
│   Deployment "stable"  │         │   Deployment "canary"   │
│   (v1) — 9 replicas      │         │   (v2) — 1 replica         │
└───────────┬───────────┘         └───────────┬───────────┘
             │                                    │
             └───────────────┬────────────────────┘
                              ▼
                   ┌────────────────────┐
                   │      Service           │  ← selector matches BOTH
                   │  (matches a label        │     Deployments' shared label
                   │   common to BOTH)          │
                   └────────────────────┘
                              │
                    ~10% of traffic lands on
                    the single canary Pod
                    (pure statistical accident
                     of Service load-balancing
                     across 10 total matching
                     Pods, per the Services guide)
```
**Which components participate:** two Deployments sharing enough common labels for **one Service** to select Pods from both — the traffic **percentage** is controlled crudely by the **replica-count ratio** in this native-only approach (1 canary Pod out of 10 total ≈ roughly 10% of traffic, since the Service load-balances essentially evenly across all matching, Ready Pods). A **service mesh** (Istio, Linkerd) or an **Ingress controller with weighted routing** can instead control the traffic percentage directly and precisely, decoupled from replica counts entirely — this is the more precise, production-grade version of the same idea.

**Progression:** gradually shift the replica ratio (or mesh traffic weight) from canary toward stable being fully replaced — 10% → 25% → 50% → 100%, watching error rates/latency at each step, pausing or reverting immediately if the canary's metrics look worse than the stable baseline.

## A/B testing

```
Request arrives at Ingress/mesh
        │
        ▼
Route based on a REQUEST ATTRIBUTE (a header, a cookie, a user ID
hash) — NOT randomly, unlike canary's statistical traffic split
        │
    ┌────┴────┐
    ▼           ▼
 Version A   Version B
 (users in    (users in
  cohort A)    cohort B)
```
**Which components participate:** this requires **Layer 7, content-aware routing** — a plain Kubernetes Service (L4, per the Services guide) has no concept of headers or cookies at all; A/B testing needs either an Ingress controller with routing-rule annotations, or a service mesh's request-routing rules, to make the version choice based on *who's asking*, not a random percentage split. **This is a fundamentally different goal from canary** — canary is about *safety* (catch bugs on a small slice before full rollout); A/B is about *measurement* (compare two genuinely different versions' business/UX outcomes on deliberately chosen user cohorts), even after both are considered "fully validated."

## Progressive delivery (the general, tooling-assisted pattern)

```
┌─────────────────────────────────────────────────────────┐
│                Argo Rollouts / Flagger                       │
│  (a CUSTOM RESOURCE + controller replacing the native          │
│   Deployment object, per the Architecture masterclass's         │
│   CRD-extensibility pattern)                                      │
│                                                                    │
│  Watches: real metrics (Prometheus error rate, latency,             │
│           success rate) DURING the rollout                          │
│  Acts: automatically advances traffic percentage on good              │
│        metrics, automatically PAUSES or ROLLS BACK on bad ones          │
└─────────────────────────────────────────────────────────────┘
```
This is canary/blue-green **automated and metric-gated**, rather than manually watched and manually advanced — the controller itself decides, based on real observed metrics, whether to proceed, pause, or abort, closing the loop that native Kubernetes Deployments (and even hand-rolled canary/blue-green) leave entirely to a human watching dashboards.

---

# PART 7 — STRATEGY COMPARISON TABLE

| Strategy | Native to Kubernetes? | Downtime | Resource cost during transition | Rollback speed | Traffic control precision |
|---|---|---|---|---|---|
| Recreate | Yes | Yes | None extra | Fast (same mechanism, reversed) | N/A (all-or-nothing) |
| RollingUpdate | Yes | No | Small (maxSurge) | Fast, reuses old ReplicaSet | None — proportional to Pod mix only |
| Blue-Green | No (manual/tooling) | No | 2x (both full versions running) | Instant (Service selector flip) | None — all-or-nothing cutover |
| Canary | No (manual/mesh/tooling) | No | Small (extra canary replicas) | Fast (scale canary to 0) | Coarse (native) to precise (mesh) |
| A/B | No (mesh/Ingress required) | No | Depends on cohort split | Depends on routing config | Precise, attribute-based, not random |
| Progressive delivery (Argo/Flagger) | No (CRD + controller) | No | Similar to canary | Fast, automated | Precise, and **automated** on real metrics |

---

# PART 8 — PRODUCTION CHECKLIST

1. **Always set a `readinessProbe`** — without one, a rolling update has no reliable signal for "is the new Pod actually safe to receive traffic," and the entire safety mechanism in Part 5 doesn't function.
2. **Set `progressDeadlineSeconds`** deliberately, and alert on `ProgressDeadlineExceeded` — a stuck rollout with no alerting can sit half-migrated for hours before anyone notices.
3. **Never assume automatic rollback** — Kubernetes never rolls back on its own; wire `kubectl rollout undo` into your pipeline's failure path explicitly, triggered by `kubectl rollout status`'s exit code or the progress-deadline condition.
4. **Use expand-contract for schema changes** — never ship a breaking schema change in the same deploy that also requires it, given old and new versions coexist during any rolling update.
5. **Tune `maxSurge`/`maxUnavailable`** deliberately based on actual cluster headroom — don't leave the defaults unexamined for workloads where either resource pressure or availability margins are tight.
6. **Meaningful `change-cause` annotations** on every Deployment update — `rollout history` with no context is nearly useless during an actual incident at 2 AM.
7. **For genuinely high-stakes rollouts**, consider Argo Rollouts/Flagger over hand-rolled canary — automated metric-gated promotion catches regressions faster and more reliably than a human watching a dashboard.

---

# PART 9 — FAILURE SCENARIOS AND ROLLBACK EXERCISES

## Exercise: induce a stuck rollout, then roll back
```bash
kubectl create deployment demo --image=nginx:1.25 --replicas=5
kubectl rollout status deployment/demo
kubectl set image deployment/demo nginx=nginx:this-tag-does-not-exist
kubectl rollout status deployment/demo --timeout=30s
# → times out; ImagePullBackOff on the new ReplicaSet's Pods
kubectl get replicasets -l app=demo
# → old RS still has Ready Pods; new RS stuck at 0 Ready
kubectl rollout undo deployment/demo
kubectl rollout status deployment/demo
# → recovers immediately, since the old RS never actually lost its Pods
```

## Exercise: readiness-probe-gated rollout, deliberately broken
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata: { name: demo2 }
spec:
  replicas: 5
  strategy:
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  selector: { matchLabels: { app: demo2 } }
  template:
    metadata: { labels: { app: demo2 } }
    spec:
      containers:
        - name: web
          image: nginx
          readinessProbe:
            httpGet: { path: /this-path-does-not-exist, port: 80 }
            periodSeconds: 5
EOF
kubectl rollout status deployment/demo2 --timeout=30s
# → never completes; new Pods run but NEVER pass readiness (404 on
#   every probe) — demonstrates a rollout stalling safely rather than
#   forcing broken Pods into service
kubectl get pods -l app=demo2
# → new Pods show READY 0/1, Running (not Crashing — the CONTAINER is
#   fine, only the PROBE PATH is wrong) — an important distinction from
#   CrashLoopBackOff
```

---

# INTERVIEW QUESTIONS

**Fundamentals**
1. What's the fundamental difference between Recreate and RollingUpdate?
2. Walk through, step by step, what happens updating 5 replicas from v1 to v2 with `maxSurge: 1, maxUnavailable: 0`.
3. Why does surge rounding go up while unavailable rounding goes down for percentage values?

**Rollback mechanics**
4. What object does `kubectl rollout undo` actually operate on, mechanically?
5. Why are rollbacks typically faster than the original rollout?
6. Does Kubernetes ever roll back a Deployment automatically? What DOES happen on a failed rollout?

**Safety**
7. What's the single mechanism that prevents a broken new version from receiving real traffic during a rolling update?
8. What does `progressDeadlineSeconds` actually do, and what does it NOT do?
9. Why must application changes deployed via rolling update be backward-compatible, even temporarily?

**Advanced strategies**
10. Why is blue-green not a native Kubernetes Deployment feature — what actually implements it?
11. How is canary traffic percentage controlled using only native Kubernetes objects, and why is that method imprecise?
12. What's the fundamental difference in GOAL between canary and A/B testing, even though both involve running two versions simultaneously?
13. What does a tool like Argo Rollouts or Flagger add on top of hand-rolled canary/blue-green?

**Production**
14. Why is "expand-contract" the standard pattern for database schema changes alongside rolling updates?
15. A rollout is stuck with new Pods `Running` but never `Ready` — is this the same failure class as CrashLoopBackOff? Explain the difference.

---

*This guide covers Kubernetes deployment strategies from native Recreate/RollingUpdate mechanics through blue-green, canary, A/B, and metric-gated progressive delivery — the complete arc from beginner rolling updates to production-grade release engineering.*
s
