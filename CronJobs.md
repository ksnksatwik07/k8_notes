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
