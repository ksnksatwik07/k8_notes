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
