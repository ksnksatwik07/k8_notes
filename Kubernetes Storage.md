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
