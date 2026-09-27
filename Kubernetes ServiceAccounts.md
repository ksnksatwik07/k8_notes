# Kubernetes ServiceAccounts — Complete Masterclass (Beginner → Production)

---

## PART 1 — CONCEPTS

### What a ServiceAccount is

A **ServiceAccount** is a Kubernetes-native identity for **processes running inside Pods** — as opposed to a human user identity. When your application code needs to call the Kubernetes API (list Pods, read a Secret, watch ConfigMaps, create Jobs), it authenticates as a ServiceAccount, not as "you."

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
```

That's the entire minimal object — a ServiceAccount by itself is just a named identity with no permissions. Permissions come entirely from **RBAC bindings** layered on top (covered below).

### Why Pods need identities

Every request to the Kubernetes API server must be **authenticated as someone** — there's no anonymous access by default in a properly configured cluster. Two categories of "someone" exist:

| | Users | ServiceAccounts |
|---|---|---|
| Who/what it represents | A human (or external system) | A process running inside a Pod |
| Managed by | External identity provider (OIDC, certs) — Kubernetes has **no** User object | Kubernetes-native object (`kind: ServiceAccount`) — created/managed via API |
| Typical use | `kubectl` from your laptop | An app Pod calling the K8s API, a controller, an operator |
| Credential | kubeconfig (cert or OIDC token) | Automatically mounted token inside the Pod |

If a Pod needs to introspect the cluster — e.g., a controller watching for CRDs, a CI job creating other Pods, an app doing service discovery via the API — **it must present some identity** to be authenticated and then authorized. That's what ServiceAccounts exist for.

### ServiceAccount vs User

This is a common interview trip-up: **Kubernetes has no `User` API object at all.** Users are entirely external — authenticated via client certificates, OIDC tokens, or a cloud IAM integration, and the cluster just trusts whatever identity that external mechanism asserts (a username + group list). ServiceAccounts, by contrast, are **first-class API objects** you can create, list, delete, and bind RBAC roles to, entirely within the cluster.

```
User:            external identity → cluster trusts assertion (cert CN, OIDC claim)
ServiceAccount:  kubectl get sa    → an actual etcd-stored object, k8s-native
```

---

## PART 2 — TOKENS & AUTHENTICATION

### Token

Every ServiceAccount has an associated **credential** — a signed JWT bearer token — that a Pod using that ServiceAccount presents to the API server on every request, in the `Authorization: Bearer <token>` HTTP header.

There have been **two generations** of this mechanism:

**Legacy (pre-1.24, still supported):** a long-lived token stored in a `Secret` object, auto-created for every ServiceAccount, auto-mounted into Pods. Never expires unless manually deleted.

**Modern (1.24+, default today): Projected Service Account Tokens** — short-lived, audience-scoped, auto-rotating tokens generated on-demand by the kubelet via the **TokenRequest API**, not stored as a long-lived Secret at all.

### Projected Service Account Tokens

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-sa
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: kube-api-access
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      readOnly: true
  volumes:
  - name: kube-api-access
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600     # token auto-rotates, valid 1hr at a time
          audience: api                # scopes token to a specific intended audience
      - configMap:
          name: kube-root-ca.crt
          items:
          - key: ca.crt
            path: ca.crt
      - downwardAPI:
          items:
          - path: namespace
            fieldRef:
              fieldPath: metadata.namespace
```

**This is auto-injected by kubelet for every Pod by default** — you rarely write this YAML by hand; it's shown here to demystify what's actually happening under `/var/run/secrets/kubernetes.io/serviceaccount/` inside every Pod.

**Why this is better than the old long-lived Secret token:**
- **Short-lived** (`expirationSeconds`, default 1hr) — kubelet transparently refreshes it before expiry, but a leaked/exfiltrated token becomes useless quickly
- **Audience-scoped** — a token requested for the `api` audience can't be replayed against a different service expecting a different audience
- **Bound to the Pod** — the token includes claims tying it to the specific Pod's UID and node; if the Pod is deleted, the token is immediately invalid, unlike the old Secret-based token which lived independently of any particular Pod

### API Server Authentication

When a request arrives with `Authorization: Bearer <token>`, the API server's **ServiceAccount token authenticator** verifies:

1. The token's signature (against the cluster's signing key)
2. The token hasn't expired
3. (For projected tokens) the token is bound to a Pod that **still exists** and hasn't been deleted

If valid, the API server extracts the identity as:
```
username: system:serviceaccount:<namespace>:<serviceaccount-name>
groups:   system:serviceaccounts
          system:serviceaccounts:<namespace>
          system:authenticated
```

This synthetic username (`system:serviceaccount:default:myapp-sa`) is exactly what RBAC `RoleBinding`/`ClusterRoleBinding` subjects reference.

---

## PART 3 — RBAC (AUTHORIZATION)

Authentication answers "who are you." **RBAC** answers "what are you allowed to do." A ServiceAccount with zero RoleBindings can authenticate successfully but will be authorized for **nothing** — every API call returns `403 Forbidden`.

### Role / ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                          # namespace-scoped permission set
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]                    # "" = core API group (Pods, Services, ConfigMaps...)
  resources: ["pods"]
  verbs: ["get", "list", "watch"]     # read-only access to Pods
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole                    # cluster-wide (or reusable across namespaces via binding)
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

- `apiGroups` — which API group the resource belongs to (`""` = core, `apps` = Deployments, `batch` = Jobs, etc.)
- `resources` — the resource type (`pods`, `secrets`, `configmaps`, `deployments`)
- `verbs` — allowed operations (`get`, `list`, `watch`, `create`, `update`, `patch`, `delete`)

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-sa-pod-reader
  namespace: default
subjects:
- kind: ServiceAccount
  name: myapp-sa                    # WHICH ServiceAccount gets this permission
  namespace: default
roleRef:
  kind: Role                         # WHICH Role is being granted
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

This is the **glue** — it says "the `myapp-sa` ServiceAccount in namespace `default` now has all permissions defined in the `pod-reader` Role." Without this binding, the `pod-reader` Role exists but grants nothing to anyone.

```
ServiceAccount "myapp-sa"  ──bound via RoleBinding──►  Role "pod-reader"
                                                          (get/list/watch pods)
```

`ClusterRoleBinding` works identically but grants a `ClusterRole`'s permissions **cluster-wide**, across all namespaces — used sparingly, for genuinely cluster-scoped needs (e.g., a monitoring agent reading nodes cluster-wide).

---

## PART 4 — THE COMPLETE FLOW

```
┌─────────┐
│   Pod    │  spec.serviceAccountName: myapp-sa
└────┬────┘
     │ kubelet auto-mounts projected token at
     │ /var/run/secrets/kubernetes.io/serviceaccount/token
     ▼
┌──────────────────┐
│  ServiceAccount    │  myapp-sa (namespace: default)
│  "myapp-sa"         │
└────────┬──────────┘
         │ token = signed JWT, claims include:
         │   sub: system:serviceaccount:default:myapp-sa
         │   aud: api
         │   exp: <1hr from issuance>
         ▼
┌──────────────────┐
│      Token         │  app code reads this file, sends as:
│  (JWT, mounted)     │  Authorization: Bearer <token>
└────────┬──────────┘
         │  HTTPS request to API server
         ▼
┌──────────────────┐
│   kube-apiserver    │
└────────┬──────────┘
         │
         ▼
┌──────────────────┐
│ Authentication      │  verify JWT signature, expiry, audience,
│ (ServiceAccount      │  and (for bound tokens) that the Pod still exists
│  token authenticator)│  → resolves identity:
└────────┬──────────┘     system:serviceaccount:default:myapp-sa
         │  (if invalid → 401 Unauthorized, stop here)
         ▼
┌──────────────────┐
│ RBAC Authorization  │  does this identity have a RoleBinding/
│                      │  ClusterRoleBinding granting the requested
│                      │  verb on the requested resource?
└────────┬──────────┘
         │  (if not allowed → 403 Forbidden, stop here)
         ▼
┌──────────────────┐
│   API Response      │  request proceeds — e.g., returns the Pod list
└──────────────────┘
```

### How a Pod actually accesses the API (in-cluster client code)

```python
# Most Kubernetes client libraries auto-detect this "in-cluster config":
import kubernetes
kubernetes.config.load_incluster_config()
# under the hood, this reads:
#   token:     /var/run/secrets/kubernetes.io/serviceaccount/token
#   ca.crt:    /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
#   namespace: /var/run/secrets/kubernetes.io/serviceaccount/namespace
#   apiserver: from KUBERNETES_SERVICE_HOST / KUBERNETES_SERVICE_PORT env vars
#              (automatically injected into every Pod by kubelet)
```

Or manually via `curl` from inside a Pod (for understanding, not typical production code):
```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
curl --cacert $CACERT \
     -H "Authorization: Bearer $TOKEN" \
     https://kubernetes.default.svc/api/v1/namespaces/default/pods
```

---

## PART 5 — DEFAULT SERVICEACCOUNT & AUTOMOUNT CONTROL

### The default ServiceAccount

Every namespace automatically gets a ServiceAccount named `default`. **Any Pod that doesn't explicitly specify `serviceAccountName` uses this one.**

```bash
kubectl get sa default -n default
kubectl get sa default -n default -o yaml
```

By itself, `default` has **no RBAC permissions** bound to it out of the box in a properly locked-down cluster — but if someone has previously bound it to a permissive Role/ClusterRole (a common misconfiguration), **every unlabeled Pod in that namespace silently inherits that access.** This is one of the most common real-world Kubernetes privilege-escalation footguns.

### `automountServiceAccountToken`

By default, every Pod gets the ServiceAccount token auto-mounted — **even if the app never calls the Kubernetes API.** This is unnecessary attack surface: if that Pod is compromised (e.g., via a code injection vulnerability), the attacker now has a live credential to probe the Kubernetes API with, for free.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
automountServiceAccountToken: false    # set at ServiceAccount level (applies to Pods using it)
---
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false   # can also override per-Pod
  containers:
  - name: app
    image: myapp:1.0
```

**Best practice:** disable this by default for any workload that doesn't genuinely need to talk to the Kubernetes API (most application Pods — web servers, workers, databases — never need it).

---

## PART 6 — WORKLOAD IDENTITY & CLOUD IAM INTEGRATION

### The problem this solves

A Pod's ServiceAccount token authenticates it **to the Kubernetes API** — it says nothing about **cloud provider permissions** (e.g., "can this Pod read from this S3 bucket / GCS bucket / Azure Blob container"). Historically, teams solved this by injecting long-lived cloud credentials (AWS access keys, GCP service account JSON keys) as Kubernetes Secrets — a significant, static, hard-to-rotate liability.

**Workload Identity** (the generic term; AWS calls it IRSA — "IAM Roles for Service Accounts", GCP calls it "Workload Identity", Azure calls it "Workload Identity Federation") lets a Kubernetes ServiceAccount's **token itself** be trusted directly by the cloud IAM system, via **OIDC federation** — no static cloud credentials stored in the cluster at all.

```
Kubernetes ServiceAccount token (OIDC-compliant JWT)
       │
       ▼
Cloud IAM trusts the cluster's OIDC issuer (configured once, cluster-wide)
       │
       ▼
Cloud IAM maps: "this specific ServiceAccount (namespace + name)"
                 → "this specific cloud IAM role"
       │
       ▼
Pod's SDK calls (boto3, google-cloud SDK, azure SDK) automatically
   exchange the K8s SA token for temporary cloud credentials
       │
       ▼
Pod can call cloud APIs (S3, GCS, etc.) with SHORT-LIVED, auto-rotated
   credentials — no static keys anywhere
```

### Example — AWS IRSA

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-reader-role
    # this annotation is what wires the K8s ServiceAccount to a specific AWS IAM role
---
apiVersion: v1
kind: Pod
metadata:
  name: s3-app
spec:
  serviceAccountName: s3-reader-sa
  containers:
  - name: app
    image: myapp:1.0
    # AWS SDK auto-detects AWS_WEB_IDENTITY_TOKEN_FILE + AWS_ROLE_ARN env vars,
    # which EKS's Pod Identity webhook injects automatically based on the
    # ServiceAccount annotation above — no code changes needed
```

**Internally:** an EKS-installed mutating webhook sees a Pod using a ServiceAccount with the `eks.amazonaws.com/role-arn` annotation, and injects `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` env vars plus a projected token volume pointing at that role. The AWS SDK inside the container then automatically calls STS `AssumeRoleWithWebIdentity`, presenting the Kubernetes-issued OIDC token, and receives short-lived AWS credentials in return.

This is the modern standard for cloud-integrated Kubernetes workloads — eliminate static cloud secrets entirely.

---

## PART 7 — SECURITY RISKS & BEST PRACTICES

### Security risks

1. **Overly broad `default` ServiceAccount permissions** — if `default` in a namespace is bound to a permissive Role/ClusterRole (even accidentally, or via a legacy Helm chart), every Pod that doesn't specify its own ServiceAccount inherits that access silently.
2. **Unnecessary token automount** — Pods that never call the K8s API still get a live credential mounted, expanding blast radius if compromised.
3. **Overly broad ClusterRoleBindings** — binding `cluster-admin` (or anything close to it) to an application ServiceAccount "to make an error go away" is a critical common mistake — it means a single Pod compromise can pivot to full cluster control.
4. **Legacy long-lived Secret-based tokens** — if still in use (older clusters, `automountServiceAccountToken` referencing an explicit long-lived Secret), a leaked token has no expiry and remains valid until manually revoked/rotated.
5. **Cross-namespace token confusion** — tokens are namespace-scoped identities, but RBAC misconfig (e.g., a `ClusterRoleBinding` instead of a `RoleBinding`) can inadvertently grant a namespace-local ServiceAccount cluster-wide reach.
6. **Wildcard RBAC rules** — `resources: ["*"]`, `verbs: ["*"]` grants used "temporarily" during debugging and never cleaned up.

### Best Practices

- **One ServiceAccount per workload/purpose** — never share a single ServiceAccount across unrelated apps; blast radius should be as narrow as possible
- **Never bind broad permissions to `default`** — leave it permission-less; require every workload needing API access to use an explicitly created, narrowly-scoped ServiceAccount
- **Set `automountServiceAccountToken: false`** on the ServiceAccount (or Pod) for any workload that doesn't call the Kubernetes API — which is most application workloads
- **Principle of least privilege in RBAC** — grant only the specific `verbs` on the specific `resources` actually needed; prefer `Role`+`RoleBinding` (namespace-scoped) over `ClusterRole`+`ClusterRoleBinding` unless truly cluster-wide access is required
- **Use projected, short-lived tokens** (the 1.24+ default) — avoid reintroducing long-lived Secret-based tokens unless you have a specific, justified need
- **Use Workload Identity / IRSA for cloud access** — never store static cloud credentials as Kubernetes Secrets when a federation mechanism is available
- **Audit RBAC regularly** — tools like `kubectl-who-can`, `rbac-lookup`, or `kubectl auth can-i --list --as=system:serviceaccount:ns:name` help catch drift and over-permissioning
- **Avoid wildcard verbs/resources in Roles** — enumerate explicitly

```bash
# Audit what a ServiceAccount can actually do:
kubectl auth can-i --list --as=system:serviceaccount:default:myapp-sa
kubectl auth can-i delete pods --as=system:serviceaccount:default:myapp-sa -n default
```

---

## PART 8 — HANDS-ON MINIKUBE LABS

```bash
minikube start
```

### Lab 1 — Create a ServiceAccount and inspect the mounted token
```bash
kubectl create serviceaccount demo-sa
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sa-demo-pod
spec:
  serviceAccountName: demo-sa
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
EOF
kubectl exec sa-demo-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/
kubectl exec sa-demo-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

### Lab 2 — 403 Forbidden with no RBAC bound
```bash
kubectl exec sa-demo-pod -- wget -q -O- \
  --header="Authorization: Bearer $(kubectl exec sa-demo-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  --no-check-certificate \
  https://kubernetes.default.svc/api/v1/namespaces/default/pods
# → 403 Forbidden — authenticated, but zero RBAC permissions
```

### Lab 3 — Grant read access via Role + RoleBinding
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: demo-sa-pod-reader
subjects:
- kind: ServiceAccount
  name: demo-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl auth can-i list pods --as=system:serviceaccount:default:demo-sa
# yes
kubectl auth can-i delete pods --as=system:serviceaccount:default:demo-sa
# no
```

### Lab 4 — Disable automount
```bash
kubectl patch serviceaccount demo-sa -p '{"automountServiceAccountToken": false}'
kubectl delete pod sa-demo-pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sa-demo-pod-2
spec:
  serviceAccountName: demo-sa
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
EOF
kubectl exec sa-demo-pod-2 -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# no such file or directory — no token mounted at all
```

### Lab 5 — Default ServiceAccount danger demo
```bash
kubectl run default-sa-pod --image=busybox -- sleep 3600
kubectl get pod default-sa-pod -o jsonpath='{.spec.serviceAccountName}'
# "default" — implicitly assigned
kubectl auth can-i list secrets --as=system:serviceaccount:default:default
# should be "no" in a properly locked-down cluster — verify it stays that way!
```

---

## PART 9 — TROUBLESHOOTING SCENARIOS

### "403 Forbidden" calling the API from inside a Pod

```bash
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa-name> -n <ns>
```
Check: is there a `RoleBinding`/`ClusterRoleBinding` at all? Does it reference the correct `Role`/`ClusterRole`? Is the `subjects[].namespace` correct (a common typo — binding to the wrong namespace silently does nothing)?

### Token file missing inside Pod

```bash
kubectl get pod <name> -o yaml | grep -A3 serviceAccount
```
Check `automountServiceAccountToken` at both Pod and ServiceAccount level — either can disable it, and Pod-level setting (if explicitly set) overrides the ServiceAccount's.

### "Unauthorized" (401) rather than "Forbidden" (403)

Distinct failure mode — means **authentication** failed, not authorization. Check token expiry, whether the Pod (and thus its bound token) still exists, and whether the token's `audience` claim matches what the API server expects.

```bash
kubectl exec <pod> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | cut -d. -f2 | base64 -d
# decode the JWT payload manually to inspect claims: exp, aud, sub
```

### App using cloud SDK gets "no credentials found" despite Workload Identity setup

Check the ServiceAccount annotation (`eks.amazonaws.com/role-arn` or GCP/Azure equivalent) is present and correctly formatted, the cluster's OIDC provider is registered in the cloud IAM trust policy, and the mutating webhook that injects the identity env vars/volume is actually running (`kubectl describe pod` should show the injected env vars if it worked).

---

## PART 10 — INTERVIEW QUESTIONS

1. What is a ServiceAccount, and what identity gap does it fill?
2. Why doesn't Kubernetes have a native `User` object, but does have `ServiceAccount`?
3. What username format does the API server derive from a ServiceAccount token?
4. What's the difference between legacy long-lived ServiceAccount tokens and projected service account tokens?
5. What is the TokenRequest API, and how does it relate to token rotation?
6. Where inside a Pod is the ServiceAccount token mounted by default?
7. What three files does kubelet typically project into a Pod for in-cluster API access?
8. What does `automountServiceAccountToken: false` do, and why is it a security best practice for most workloads?
9. What is the `default` ServiceAccount, and why is binding broad RBAC to it dangerous?
10. Walk through the full authentication + authorization flow when a Pod calls the Kubernetes API.
11. What's the difference between a 401 and a 403 response, in terms of which stage failed?
12. What's the difference between `Role` and `ClusterRole`?
13. What's the difference between `RoleBinding` and `ClusterRoleBinding`?
14. Can a `RoleBinding` reference a `ClusterRole`? What does that achieve?
15. What RBAC fields make up a `Role`'s `rules` entry?
16. How would you audit exactly what permissions a given ServiceAccount has?
17. What is Workload Identity (or IRSA on AWS), and what problem does it solve versus static cloud credentials?
18. Explain the OIDC federation trust chain in a cloud Workload Identity setup.
19. What cloud-side configuration links a specific ServiceAccount to a specific cloud IAM role?
20. Why is a JWT audience (`aud`) claim important for ServiceAccount token security?
21. What happens to a projected ServiceAccount token's validity if the Pod it's bound to is deleted?
22. What's a realistic attack scenario enabled by overly broad `cluster-admin` binding to an app ServiceAccount?
23. Why should you avoid sharing one ServiceAccount across multiple unrelated workloads?
24. What tool/command would you use to quickly check "can this ServiceAccount delete Secrets"?
25. What's the security risk of `resources: ["*"], verbs: ["*"]` in a Role, even scoped to one namespace?
26. How would you migrate a workload from static cloud access keys stored as Kubernetes Secrets to Workload Identity?
27. What KUBERNETES_SERVICE_HOST/PORT env vars are auto-injected into every Pod, and what are they for?
28. Why is namespace correctness in a RoleBinding's `subjects` field a common source of silent RBAC failures?
29. What's the practical difference in blast radius between a compromised Pod with no ServiceAccount token mounted vs. one with a `cluster-admin`-bound token?
30. Design a least-privilege ServiceAccount + RBAC setup for a CI/CD Pod that needs to create Jobs and read their logs, but nothing else.

---

That completes the ServiceAccounts chapter. Ready to continue to the next batch whenever you'd like — Services & Networking, ConfigMaps/Secrets, Ingress, or Network Policies would all logically follow in the security/identity thread we're now in.
