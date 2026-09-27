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
