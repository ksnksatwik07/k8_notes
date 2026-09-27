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
