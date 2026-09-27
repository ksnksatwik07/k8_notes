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
