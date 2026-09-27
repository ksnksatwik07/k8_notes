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
