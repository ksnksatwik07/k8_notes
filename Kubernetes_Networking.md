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
