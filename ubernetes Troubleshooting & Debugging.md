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
