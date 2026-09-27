# Kubernetes # Kubernetes Pod Troubleshooting — Complete Masterclass
## Part 1 of 4: CrashLoopBackOff, ImagePullBackOff, ErrImagePull, Pending, OOMKilled

*(This masterclass covers 15 failure modes at full depth — delivered in installments so each gets the complete 12-point treatment rather than being compressed. Part 1 covers the five most frequently encountered failures. Say "continue" for Part 2: CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted.)*

---

# THE UNIVERSAL TROUBLESHOOTING ENTRY POINT

Before diving into individual failures, every single one of them starts the same way:
```bash
kubectl get pods                    # spot the abnormal STATUS column
kubectl describe pod <pod>           # Events section — almost always names the exact problem
kubectl logs <pod>                    # what the CONTAINER itself said before dying
kubectl logs <pod> --previous          # logs from the PREVIOUS crashed instance (crucial for CrashLoopBackOff)
```
Every section below assumes you've already run these four commands — they're the universal first move, not repeated as "step 1" for every failure type below.

---

# 1. CrashLoopBackOff

## What it means
The container **starts, then exits** (crashes, or even exits cleanly with code 0 when it shouldn't), repeatedly — and Kubernetes is now waiting increasingly long **backoff** periods between each restart attempt rather than restarting instantly forever.

## What Kubernetes is doing internally
This is a **kubelet-local** behavior (Pods masterclass, Diagram D1) — no scheduler, no controller-manager involvement. The kubelet observes the container exit, and per `restartPolicy: Always` (the Deployment/ReplicaSet default), restarts it — but with **exponential backoff**: 10s, 20s, 40s, 80s... capped at 5 minutes between attempts, specifically to avoid hammering a fundamentally broken container in a tight, wasteful restart loop.
```
Container exits → kubelet waits (backoff) → restarts → exits again → wait LONGER → restart → ...
```

## Common causes
- Application crashes immediately on startup (missing config, unhandled exception, failed dependency connection)
- Wrong `command`/`args` — e.g., a command that runs once and exits, used on a Pod expecting a long-running process
- A missing environment variable or file the app requires to even initialize
- The main process legitimately finishing (exit 0) in a container Kubernetes expects to run forever

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     CrashLoopBackOff   7          12m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl logs my-app                 # current attempt (often empty if it crashed instantly)
kubectl logs my-app --previous       # THE crash's actual output — usually the real answer
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      ...
  Finished:     ...
Restart Count:  7
```
**Exit code matters a lot here:**
- `0` — the process exited cleanly, meaning your restartPolicy expectations and the app's actual behavior disagree (app thinks it's done; Kubernetes expects it to run forever)
- `1` (or other nonzero) — generic application error; the actual reason is almost always in `logs --previous`
- `137` — this is actually `OOMKilled` wearing a CrashLoopBackOff's clothes (§5) — always check `Reason` explicitly, not just the exit code

## Root-cause investigation
```bash
kubectl logs my-app --previous --timestamps
kubectl describe pod my-app | grep -A10 "Last State"
# reproduce locally if possible:
docker run --rm <same-image> <same-command>
```
The single highest-leverage move: **read `logs --previous` before doing anything else** — the overwhelming majority of CrashLoopBackOff cases are fully explained by the application's own final log lines before it died.

## Fixes
- Fix the actual application bug/misconfiguration revealed by the logs
- If it's a legitimately run-to-completion process, use a **Job** (Jobs guide), not a Deployment
- Add a `startupProbe` (§15) if the app just needs more time before health checks should even begin evaluating it

## Verification
```bash
kubectl get pod my-app -w
# RESTARTS stops climbing, STATUS settles to Running, READY reaches 1/1
```

## Prevention
- Fail fast and log clearly on startup — a silent crash with no log output is far harder to diagnose than one that prints exactly what config was missing
- Use `startupProbe` for slow-initializing apps so the kubelet doesn't prematurely judge them unhealthy during legitimate startup time

## Production example
A Node.js app crash-looped in production because a required environment variable (`DATABASE_URL`) was renamed in a ConfigMap update but the Deployment's `env` reference still pointed at the old key name — `logs --previous` showed `TypeError: Cannot read property 'connect' of undefined` within milliseconds of each restart, immediately pointing at the missing config rather than a code bug.

## Interview question
**Q: A Pod is in CrashLoopBackOff with exit code 0. Why is that unusual, and what does it suggest?**
A: Exit code 0 means the process believed it finished successfully — but `restartPolicy: Always` (typical under a Deployment) treats *any* exit as something to restart. This mismatch suggests either the workload should actually be a Job (finite, run-to-completion) rather than a Deployment, or the application has a bug causing it to terminate early when it should keep running.

---

# 2–3. ImagePullBackOff and ErrImagePull

## What they mean
`ErrImagePull` is the **first** failed attempt to pull a container image. `ImagePullBackOff` is what you see on **subsequent** attempts, once the kubelet starts backing off between retries — the same exponential-backoff relationship as CrashLoopBackOff, just applied to image pulls instead of container starts.

## What Kubernetes is doing internally
```
kubelet sees a Pod assigned to its node
    → calls the container runtime (via CRI) to pull the specified image
    → runtime attempts to contact the registry, authenticate, download layers
    → FAILS → kubelet reports ErrImagePull, schedules a retry with backoff
    → next failure → reports ImagePullBackOff, backoff continues to grow
```

## Common causes
- Typo in the image name or tag (`myapp:1.O` instead of `1.0` — a classic letter-O-vs-zero mistake)
- Image genuinely doesn't exist in the registry, or was deleted
- Private registry requiring authentication, with no (or wrong) `imagePullSecrets` configured
- Rate limiting from the registry (Docker Hub's anonymous pull limits are a very common real-world trigger)
- Network policy or firewall blocking egress from nodes to the registry

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     ImagePullBackOff   0          2m
```

## Commands to run
```bash
kubectl describe pod my-app
```

## How to interpret the output
```
Events:
  Warning  Failed     kubelet  Failed to pull image "myapp:1.O":
                                rpc error: code = NotFound desc = failed to pull and
                                unpack image "docker.io/library/myapp:1.O":
                                failed to resolve reference: myapp:1.O: not found
  Warning  Failed     kubelet  Error: ErrImagePull
  Normal   BackOff    kubelet  Back-off pulling image "myapp:1.O"
  Warning  Failed     kubelet  Error: ImagePullBackOff
```
The **exact error text after "Failed to pull image"** tells you precisely which failure mode you're in — `not found` (bad name/tag), `unauthorized`/`403` (auth problem), `connection refused`/`timeout` (network problem) all point in genuinely different directions.

## Root-cause investigation
```bash
# verify the image actually exists and is spelled correctly:
docker pull myapp:1.0          # try it manually, from a machine with equivalent network access
kubectl get pod my-app -o jsonpath='{.spec.containers[0].image}'   # confirm exactly what was requested
kubectl get pod my-app -o jsonpath='{.spec.imagePullSecrets}'       # confirm a pull secret is even attached
```

## Fixes
```bash
# for private registries:
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io --docker-username=me --docker-password=secret
```
```yaml
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: myregistry.io/myapp:1.0    # fix any typo here
```

## Verification
```bash
kubectl delete pod my-app        # if managed by a controller, a fresh Pod will retry immediately
kubectl get pod -w
```

## Prevention
- Pin exact image tags (never `:latest` in production — Pods masterclass-adjacent best practice) and verify them in CI before deployment
- Test `imagePullSecrets` in a staging namespace before relying on them in production
- Consider a pull-through cache/mirror to avoid public registry rate limits entirely

## Production example
A cluster's nodes suddenly couldn't pull any Docker Hub images after Docker Hub introduced anonymous pull rate limits — every new Pod scheduled that day hit `ErrImagePull` with a `429 Too Many Requests` message buried in the Events, resolved by adding authenticated `imagePullSecrets` (which have a much higher rate limit) cluster-wide via a default ServiceAccount patch.

## Interview question
**Q: What's the actual difference between ErrImagePull and ImagePullBackOff?**
A: They're the same underlying failure — ErrImagePull is the immediate, first-attempt failure state; ImagePullBackOff is what's reported once the kubelet starts throttling retry attempts with exponential backoff after repeated failures. Neither indicates a fundamentally different problem — they're two points on the same timeline.

---

# 4. Pending

## What it means
The Pod object exists in etcd, but **has not been scheduled to a node at all** (or, less commonly, is scheduled but can't start for some pre-container reason) — this is the Pod phase from the Pods masterclass §6–9, at its most literal.

## What Kubernetes is doing internally
```
Pod created → scheduler watches for Pods with empty .spec.nodeName
            → filtering phase eliminates unsuitable nodes
            → IF NO NODE SURVIVES FILTERING: Pod stays Pending, scheduler
              retries on backoff and whenever cluster state changes
              (Architecture masterclass, Part 4 §28)
```

## Common causes
- **Insufficient resources** cluster-wide for the Pod's requests (Resource Requests guide)
- **Taints without matching tolerations** — every node the Pod could otherwise fit on is tainted against it
- **Node selector/affinity rules** that no current node satisfies
- **No available PersistentVolume** matching a required PVC (Storage guide) — scheduling waits on binding
- Zero worker nodes exist yet (a brand-new cluster, or all nodes cordoned/draining)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS    RESTARTS   AGE
# my-app    0/1     Pending   0          5m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl get nodes
kubectl describe nodes | grep -A5 "Allocated resources"
```

## How to interpret the output
```
Events:
  Warning  FailedScheduling  0/5 nodes are available: 3 Insufficient cpu,
           2 node(s) had taint {dedicated: gpu}, that the pod didn't tolerate.
```
This message is **exhaustive and literal** — it tells you exactly how many nodes were rejected and for which specific reason, across every node in the cluster. Read it fully before guessing.

## Root-cause investigation
```bash
kubectl get pod my-app -o jsonpath='{.spec.tolerations}'
kubectl get pod my-app -o jsonpath='{.spec.nodeSelector}'
kubectl get pod my-app -o jsonpath='{.spec.containers[*].resources}'
kubectl describe nodes | grep -B2 Taints
```

## Fixes
- Reduce over-inflated resource requests, or add cluster capacity (more nodes, or trigger a cluster autoscaler)
- Add the required `tolerations` if the workload genuinely belongs on a tainted node pool
- Correct a `nodeSelector`/`affinity` rule that's impossible to satisfy (e.g., referencing a label that no node actually has, often a typo)

## Verification
```bash
kubectl get pod my-app -w
# STATUS moves from Pending to ContainerCreating to Running
```

## Prevention
- Right-size resource requests based on actual observed usage (Resource Requests guide) rather than guessing high "to be safe," which wastes schedulable capacity across the whole cluster
- Keep taints/tolerations and nodeSelectors documented and reviewed — these are easy to silently break during node pool migrations

## Production example
A team migrated from one node pool to another with different labels; their Deployment's `nodeSelector` still referenced the old pool's label, and every new Pod sat `Pending` indefinitely after the old pool was fully drained — `FailedScheduling`'s message explicitly listed "0/12 nodes match node selector," which immediately pointed at the stale selector.

## Interview question
**Q: Is a Pending Pod always a resource-capacity problem?**
A: No — resource insufficiency is one cause among several (taints/tolerations mismatches, node selector/affinity that no node satisfies, an unbound PVC, or literally zero nodes existing). The `FailedScheduling` event message always states the specific reason(s); assuming it's always "not enough CPU/memory" without reading the actual message is a common, avoidable diagnostic mistake.

---

# 5. OOMKilled

## What it means
*(Full internal mechanics already covered exhaustively in the Resource Limits guide, Part 3 — this section is the troubleshooting-flow-focused summary.)* A container exceeded its memory limit and was forcibly killed by the kernel's OOM killer, scoped to that container's own cgroup.

## What Kubernetes is doing internally
```
Container's memory usage → approaches cgroup memory.max
    → kernel reclaims what it can (silent, no kill yet)
    → still over → kernel OOM killer sends SIGKILL to the container's process
    → container exits with code 137 (128 + SIGKILL's signal number 9)
    → kubelet observes the exit, reports Reason: OOMKilled
    → restartPolicy governs what happens next (same restart mechanics as
      any container exit, per Pods masterclass Diagram D1)
```

## Common causes
- Memory limit set genuinely too low for real peak usage
- A memory leak — usage climbs steadily over the container's lifetime rather than stabilizing
- A sudden legitimate spike (large request payload, batch processing burst) exceeding a limit sized for steady-state only
- No limit set at all, but the **node itself** ran out of memory (node-level OOM, per the Resource Limits guide — more disruptive, can affect unrelated Pods)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS      RESTARTS   AGE
# my-app    0/1     OOMKilled   3          8m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl top pod my-app --containers      # if it's currently running, see live usage
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Started:      ...
  Finished:     ...
```
**Exit code 137 plus `Reason: OOMKilled` together are unambiguous** — this is never a coincidence or a generic crash; it's specifically the kernel's memory enforcement, distinct from every other failure in this guide.

## Root-cause investigation
```bash
kubectl describe pod my-app | grep -A3 "Limits\|Requests"
kubectl top pod my-app --containers        # compare against the limit, if still running long enough
# check historical usage if you have metrics retention (Prometheus, etc.) —
# was this a SLOW climb (leak) or a SUDDEN spike (load-driven)?
```

## Fixes
- Raise the memory limit to a realistic ceiling based on observed peak usage, with headroom
- Fix an actual memory leak if usage climbs without bound over time rather than stabilizing
- For genuinely bursty workloads, consider whether the request should be raised too (affects scheduling and QoS, per the Resource Requests/Limits guides), not just the limit

## Verification
```bash
kubectl get pod my-app -w
kubectl top pod my-app --containers -w
# usage should stabilize comfortably under the new limit, RESTARTS stops climbing
```

## Prevention
- Load-test with realistic peak traffic before setting production memory limits
- Alert on Pods approaching their memory limit (e.g., >80% sustained) **before** they get OOMKilled, not just after
- Use Guaranteed QoS (Resource Limits guide) for workloads where an OOMKill is especially costly/disruptive

## Production example
A Java application was OOMKilled repeatedly after a Kubernetes migration because its JVM heap flags were set assuming the *node's* total memory, not the container's cgroup limit — the JVM allocated far more heap than the container was actually permitted, guaranteeing an eventual OOMKill under any real load; the fix was explicit `-Xmx` flags sized against the container's own memory limit, not the host's.

## Interview question
**Q: A container is OOMKilled even though `kubectl top pod` showed usage well under its limit most of the time. What's the likely explanation?**
A: `kubectl top` reports periodic snapshots, not continuous monitoring — a brief, sharp memory spike between polling intervals can cross the limit and trigger an instant SIGKILL without ever being visible in average or snapshot-based metrics. This is exactly why memory limit violations have zero grace period, unlike CPU throttling: there's no "average was fine" defense once a single instant crosses the cgroup ceiling.

---

## What's coming in Part 2

Part 2 covers: **CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted** — each with the full 12-point treatment, realistic error messages, and production examples.

Say **"continue"** whenever you're ready. — Complete Masterclass
## Part 1 of 4: CrashLoopBackOff, ImagePullBackOff, ErrImagePull, Pending, OOMKilled

*(This masterclass covers 15 failure modes at full depth — delivered in installments so each gets the complete 12-point treatment rather than being compressed. Part 1 covers the five most frequently encountered failures. Say "continue" for Part 2: CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted.)*

---

# THE UNIVERSAL TROUBLESHOOTING ENTRY POINT

Before diving into individual failures, every single one of them starts the same way:
```bash
kubectl get pods                    # spot the abnormal STATUS column
kubectl describe pod <pod>           # Events section — almost always names the exact problem
kubectl logs <pod>                    # what the CONTAINER itself said before dying
kubectl logs <pod> --previous          # logs from the PREVIOUS crashed instance (crucial for CrashLoopBackOff)
```
Every section below assumes you've already run these four commands — they're the universal first move, not repeated as "step 1" for every failure type below.

---

# 1. CrashLoopBackOff

## What it means
The container **starts, then exits** (crashes, or even exits cleanly with code 0 when it shouldn't), repeatedly — and Kubernetes is now waiting increasingly long **backoff** periods between each restart attempt rather than restarting instantly forever.

## What Kubernetes is doing internally
This is a **kubelet-local** behavior (Pods masterclass, Diagram D1) — no scheduler, no controller-manager involvement. The kubelet observes the container exit, and per `restartPolicy: Always` (the Deployment/ReplicaSet default), restarts it — but with **exponential backoff**: 10s, 20s, 40s, 80s... capped at 5 minutes between attempts, specifically to avoid hammering a fundamentally broken container in a tight, wasteful restart loop.
```
Container exits → kubelet waits (backoff) → restarts → exits again → wait LONGER → restart → ...
```

## Common causes
- Application crashes immediately on startup (missing config, unhandled exception, failed dependency connection)
- Wrong `command`/`args` — e.g., a command that runs once and exits, used on a Pod expecting a long-running process
- A missing environment variable or file the app requires to even initialize
- The main process legitimately finishing (exit 0) in a container Kubernetes expects to run forever

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     CrashLoopBackOff   7          12m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl logs my-app                 # current attempt (often empty if it crashed instantly)
kubectl logs my-app --previous       # THE crash's actual output — usually the real answer
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      ...
  Finished:     ...
Restart Count:  7
```
**Exit code matters a lot here:**
- `0` — the process exited cleanly, meaning your restartPolicy expectations and the app's actual behavior disagree (app thinks it's done; Kubernetes expects it to run forever)
- `1` (or other nonzero) — generic application error; the actual reason is almost always in `logs --previous`
- `137` — this is actually `OOMKilled` wearing a CrashLoopBackOff's clothes (§5) — always check `Reason` explicitly, not just the exit code

## Root-cause investigation
```bash
kubectl logs my-app --previous --timestamps
kubectl describe pod my-app | grep -A10 "Last State"
# reproduce locally if possible:
docker run --rm <same-image> <same-command>
```
The single highest-leverage move: **read `logs --previous` before doing anything else** — the overwhelming majority of CrashLoopBackOff cases are fully explained by the application's own final log lines before it died.

## Fixes
- Fix the actual application bug/misconfiguration revealed by the logs
- If it's a legitimately run-to-completion process, use a **Job** (Jobs guide), not a Deployment
- Add a `startupProbe` (§15) if the app just needs more time before health checks should even begin evaluating it

## Verification
```bash
kubectl get pod my-app -w
# RESTARTS stops climbing, STATUS settles to Running, READY reaches 1/1
```

## Prevention
- Fail fast and log clearly on startup — a silent crash with no log output is far harder to diagnose than one that prints exactly what config was missing
- Use `startupProbe` for slow-initializing apps so the kubelet doesn't prematurely judge them unhealthy during legitimate startup time

## Production example
A Node.js app crash-looped in production because a required environment variable (`DATABASE_URL`) was renamed in a ConfigMap update but the Deployment's `env` reference still pointed at the old key name — `logs --previous` showed `TypeError: Cannot read property 'connect' of undefined` within milliseconds of each restart, immediately pointing at the missing config rather than a code bug.

## Interview question
**Q: A Pod is in CrashLoopBackOff with exit code 0. Why is that unusual, and what does it suggest?**
A: Exit code 0 means the process believed it finished successfully — but `restartPolicy: Always` (typical under a Deployment) treats *any* exit as something to restart. This mismatch suggests either the workload should actually be a Job (finite, run-to-completion) rather than a Deployment, or the application has a bug causing it to terminate early when it should keep running.

---

# 2–3. ImagePullBackOff and ErrImagePull

## What they mean
`ErrImagePull` is the **first** failed attempt to pull a container image. `ImagePullBackOff` is what you see on **subsequent** attempts, once the kubelet starts backing off between retries — the same exponential-backoff relationship as CrashLoopBackOff, just applied to image pulls instead of container starts.

## What Kubernetes is doing internally
```
kubelet sees a Pod assigned to its node
    → calls the container runtime (via CRI) to pull the specified image
    → runtime attempts to contact the registry, authenticate, download layers
    → FAILS → kubelet reports ErrImagePull, schedules a retry with backoff
    → next failure → reports ImagePullBackOff, backoff continues to grow
```

## Common causes
- Typo in the image name or tag (`myapp:1.O` instead of `1.0` — a classic letter-O-vs-zero mistake)
- Image genuinely doesn't exist in the registry, or was deleted
- Private registry requiring authentication, with no (or wrong) `imagePullSecrets` configured
- Rate limiting from the registry (Docker Hub's anonymous pull limits are a very common real-world trigger)
- Network policy or firewall blocking egress from nodes to the registry

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# my-app    0/1     ImagePullBackOff   0          2m
```

## Commands to run
```bash
kubectl describe pod my-app
```

## How to interpret the output
```
Events:
  Warning  Failed     kubelet  Failed to pull image "myapp:1.O":
                                rpc error: code = NotFound desc = failed to pull and
                                unpack image "docker.io/library/myapp:1.O":
                                failed to resolve reference: myapp:1.O: not found
  Warning  Failed     kubelet  Error: ErrImagePull
  Normal   BackOff    kubelet  Back-off pulling image "myapp:1.O"
  Warning  Failed     kubelet  Error: ImagePullBackOff
```
The **exact error text after "Failed to pull image"** tells you precisely which failure mode you're in — `not found` (bad name/tag), `unauthorized`/`403` (auth problem), `connection refused`/`timeout` (network problem) all point in genuinely different directions.

## Root-cause investigation
```bash
# verify the image actually exists and is spelled correctly:
docker pull myapp:1.0          # try it manually, from a machine with equivalent network access
kubectl get pod my-app -o jsonpath='{.spec.containers[0].image}'   # confirm exactly what was requested
kubectl get pod my-app -o jsonpath='{.spec.imagePullSecrets}'       # confirm a pull secret is even attached
```

## Fixes
```bash
# for private registries:
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io --docker-username=me --docker-password=secret
```
```yaml
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: myregistry.io/myapp:1.0    # fix any typo here
```

## Verification
```bash
kubectl delete pod my-app        # if managed by a controller, a fresh Pod will retry immediately
kubectl get pod -w
```

## Prevention
- Pin exact image tags (never `:latest` in production — Pods masterclass-adjacent best practice) and verify them in CI before deployment
- Test `imagePullSecrets` in a staging namespace before relying on them in production
- Consider a pull-through cache/mirror to avoid public registry rate limits entirely

## Production example
A cluster's nodes suddenly couldn't pull any Docker Hub images after Docker Hub introduced anonymous pull rate limits — every new Pod scheduled that day hit `ErrImagePull` with a `429 Too Many Requests` message buried in the Events, resolved by adding authenticated `imagePullSecrets` (which have a much higher rate limit) cluster-wide via a default ServiceAccount patch.

## Interview question
**Q: What's the actual difference between ErrImagePull and ImagePullBackOff?**
A: They're the same underlying failure — ErrImagePull is the immediate, first-attempt failure state; ImagePullBackOff is what's reported once the kubelet starts throttling retry attempts with exponential backoff after repeated failures. Neither indicates a fundamentally different problem — they're two points on the same timeline.

---

# 4. Pending

## What it means
The Pod object exists in etcd, but **has not been scheduled to a node at all** (or, less commonly, is scheduled but can't start for some pre-container reason) — this is the Pod phase from the Pods masterclass §6–9, at its most literal.

## What Kubernetes is doing internally
```
Pod created → scheduler watches for Pods with empty .spec.nodeName
            → filtering phase eliminates unsuitable nodes
            → IF NO NODE SURVIVES FILTERING: Pod stays Pending, scheduler
              retries on backoff and whenever cluster state changes
              (Architecture masterclass, Part 4 §28)
```

## Common causes
- **Insufficient resources** cluster-wide for the Pod's requests (Resource Requests guide)
- **Taints without matching tolerations** — every node the Pod could otherwise fit on is tainted against it
- **Node selector/affinity rules** that no current node satisfies
- **No available PersistentVolume** matching a required PVC (Storage guide) — scheduling waits on binding
- Zero worker nodes exist yet (a brand-new cluster, or all nodes cordoned/draining)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS    RESTARTS   AGE
# my-app    0/1     Pending   0          5m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl get nodes
kubectl describe nodes | grep -A5 "Allocated resources"
```

## How to interpret the output
```
Events:
  Warning  FailedScheduling  0/5 nodes are available: 3 Insufficient cpu,
           2 node(s) had taint {dedicated: gpu}, that the pod didn't tolerate.
```
This message is **exhaustive and literal** — it tells you exactly how many nodes were rejected and for which specific reason, across every node in the cluster. Read it fully before guessing.

## Root-cause investigation
```bash
kubectl get pod my-app -o jsonpath='{.spec.tolerations}'
kubectl get pod my-app -o jsonpath='{.spec.nodeSelector}'
kubectl get pod my-app -o jsonpath='{.spec.containers[*].resources}'
kubectl describe nodes | grep -B2 Taints
```

## Fixes
- Reduce over-inflated resource requests, or add cluster capacity (more nodes, or trigger a cluster autoscaler)
- Add the required `tolerations` if the workload genuinely belongs on a tainted node pool
- Correct a `nodeSelector`/`affinity` rule that's impossible to satisfy (e.g., referencing a label that no node actually has, often a typo)

## Verification
```bash
kubectl get pod my-app -w
# STATUS moves from Pending to ContainerCreating to Running
```

## Prevention
- Right-size resource requests based on actual observed usage (Resource Requests guide) rather than guessing high "to be safe," which wastes schedulable capacity across the whole cluster
- Keep taints/tolerations and nodeSelectors documented and reviewed — these are easy to silently break during node pool migrations

## Production example
A team migrated from one node pool to another with different labels; their Deployment's `nodeSelector` still referenced the old pool's label, and every new Pod sat `Pending` indefinitely after the old pool was fully drained — `FailedScheduling`'s message explicitly listed "0/12 nodes match node selector," which immediately pointed at the stale selector.

## Interview question
**Q: Is a Pending Pod always a resource-capacity problem?**
A: No — resource insufficiency is one cause among several (taints/tolerations mismatches, node selector/affinity that no node satisfies, an unbound PVC, or literally zero nodes existing). The `FailedScheduling` event message always states the specific reason(s); assuming it's always "not enough CPU/memory" without reading the actual message is a common, avoidable diagnostic mistake.

---

# 5. OOMKilled

## What it means
*(Full internal mechanics already covered exhaustively in the Resource Limits guide, Part 3 — this section is the troubleshooting-flow-focused summary.)* A container exceeded its memory limit and was forcibly killed by the kernel's OOM killer, scoped to that container's own cgroup.

## What Kubernetes is doing internally
```
Container's memory usage → approaches cgroup memory.max
    → kernel reclaims what it can (silent, no kill yet)
    → still over → kernel OOM killer sends SIGKILL to the container's process
    → container exits with code 137 (128 + SIGKILL's signal number 9)
    → kubelet observes the exit, reports Reason: OOMKilled
    → restartPolicy governs what happens next (same restart mechanics as
      any container exit, per Pods masterclass Diagram D1)
```

## Common causes
- Memory limit set genuinely too low for real peak usage
- A memory leak — usage climbs steadily over the container's lifetime rather than stabilizing
- A sudden legitimate spike (large request payload, batch processing burst) exceeding a limit sized for steady-state only
- No limit set at all, but the **node itself** ran out of memory (node-level OOM, per the Resource Limits guide — more disruptive, can affect unrelated Pods)

## How to identify it
```bash
kubectl get pods
# NAME      READY   STATUS      RESTARTS   AGE
# my-app    0/1     OOMKilled   3          8m
```

## Commands to run
```bash
kubectl describe pod my-app
kubectl top pod my-app --containers      # if it's currently running, see live usage
```

## How to interpret the output
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Started:      ...
  Finished:     ...
```
**Exit code 137 plus `Reason: OOMKilled` together are unambiguous** — this is never a coincidence or a generic crash; it's specifically the kernel's memory enforcement, distinct from every other failure in this guide.

## Root-cause investigation
```bash
kubectl describe pod my-app | grep -A3 "Limits\|Requests"
kubectl top pod my-app --containers        # compare against the limit, if still running long enough
# check historical usage if you have metrics retention (Prometheus, etc.) —
# was this a SLOW climb (leak) or a SUDDEN spike (load-driven)?
```

## Fixes
- Raise the memory limit to a realistic ceiling based on observed peak usage, with headroom
- Fix an actual memory leak if usage climbs without bound over time rather than stabilizing
- For genuinely bursty workloads, consider whether the request should be raised too (affects scheduling and QoS, per the Resource Requests/Limits guides), not just the limit

## Verification
```bash
kubectl get pod my-app -w
kubectl top pod my-app --containers -w
# usage should stabilize comfortably under the new limit, RESTARTS stops climbing
```

## Prevention
- Load-test with realistic peak traffic before setting production memory limits
- Alert on Pods approaching their memory limit (e.g., >80% sustained) **before** they get OOMKilled, not just after
- Use Guaranteed QoS (Resource Limits guide) for workloads where an OOMKill is especially costly/disruptive

## Production example
A Java application was OOMKilled repeatedly after a Kubernetes migration because its JVM heap flags were set assuming the *node's* total memory, not the container's cgroup limit — the JVM allocated far more heap than the container was actually permitted, guaranteeing an eventual OOMKill under any real load; the fix was explicit `-Xmx` flags sized against the container's own memory limit, not the host's.

## Interview question
**Q: A container is OOMKilled even though `kubectl top pod` showed usage well under its limit most of the time. What's the likely explanation?**
A: `kubectl top` reports periodic snapshots, not continuous monitoring — a brief, sharp memory spike between polling intervals can cross the limit and trigger an instant SIGKILL without ever being visible in average or snapshot-based metrics. This is exactly why memory limit violations have zero grace period, unlike CPU throttling: there's no "average was fine" defense once a single instant crosses the cgroup ceiling.

---

## What's coming in Part 2

Part 2 covers: **CreateContainerError, CreateContainerConfigError, FailedMount, ContainerCreating (stuck), and Evicted** — each with the full 12-point treatment, realistic error messages, and production examples.

Say **"continue"** whenever you're ready.
