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
