# Kubernetes RBAC — Complete Mastery Guide
### From Absolute Beginner to Production Security Architecture

---

# PART 1 — AUTHENTICATION VS AUTHORIZATION

*(Both already introduced structurally in the Architecture masterclass, Part 3 §24 — this guide goes deep specifically on the authorization stage, since that's what RBAC actually is.)*

## Authentication (AuthN) — "Who are you?"
Kubernetes doesn't have its own built-in user database — identity comes from external mechanisms: client certificates, OIDC tokens (from an identity provider like Google/Okta/Azure AD), static tokens, or ServiceAccount tokens for in-cluster workloads. By the time a request reaches the authorization stage, it carries exactly one thing forward: an **identity** — a username and a set of group memberships.

## Authorization (AuthZ) — "Are you allowed to do THIS?"
Given that identity, is this **specific request** (this verb, on this resource, in this namespace) permitted? This is where **RBAC** (Role-Based Access Control) lives — it is Kubernetes' dominant, though not only, authorization mode (others exist — ABAC, Webhook, Node — but RBAC is what the overwhelming majority of real clusters use).

```
Request arrives
      │
      ▼
AUTHENTICATION → produces an identity: "user: alice, groups: [developers]"
                  or "serviceaccount: default:my-app-sa"
      │
      ▼
AUTHORIZATION (RBAC) → "is alice, or the developers group, permitted
                         to do THIS specific verb/resource/namespace
                         combination?"
      │
      ▼
   allowed → proceed to admission control (Architecture masterclass §24)
   denied  → 403 Forbidden, request stops here entirely
```

---

# PART 2 — THE RBAC MENTAL MODEL

## The one sentence that is all of RBAC
**Who → can perform what action → on which resource → in which namespace.**

```
   WHO                  WHAT ACTION           WHICH RESOURCE        WHICH SCOPE
 (subject)               (verb)                (resource type)      (namespace or cluster)
─────────────         ──────────────         ─────────────────    ───────────────────
 User: alice            get, list, watch        pods                 namespace: "dev"
 Group: developers       create, update           deployments           namespace: "staging"
 ServiceAccount: sa       delete, patch              secrets              cluster-wide
```
Every RBAC question you'll ever be asked reduces to filling in these four blanks. The entire rest of this guide is just the mechanics of how Kubernetes objects express that sentence.

## The four RBAC objects, and how they pair up

```
Role / ClusterRole          =  WHAT ACTION + WHICH RESOURCE
                                (a reusable permission set, no "who" yet)

RoleBinding / ClusterRoleBinding  =  WHO  +  attaches to a Role/ClusterRole
                                       (connects a subject to a permission set)
```

| | Namespace-scoped | Cluster-scoped |
|---|---|---|
| Permission set | **Role** | **ClusterRole** |
| Attaches subjects | **RoleBinding** | **ClusterRoleBinding** |

**Four combinations, each with a distinct real meaning** — this 2×2 is the single most important table in this guide, unpacked fully in Part 4.

---

# PART 3 — VERBS, RESOURCES, AND API GROUPS

## Verbs
The action being performed — mapped directly onto the REST HTTP methods from the Kubernetes API (Architecture masterclass §18):

| Verb | Roughly maps to |
|---|---|
| `get` | Read one specific object |
| `list` | Read a collection of objects |
| `watch` | Subscribe to changes (the mechanism behind Informers, Architecture masterclass §22–23) |
| `create` | POST a new object |
| `update` | Replace an existing object entirely |
| `patch` | Partially modify an existing object |
| `delete` | Remove an object |
| `deletecollection` | Remove a whole collection at once |

## Resources
The object **type** being acted on — `pods`, `deployments`, `secrets`, `configmaps` — always the **plural, lowercase** REST resource name, exactly as it appears in the API path (Architecture masterclass §18), not the `kind` field's capitalized singular form.

## API groups
Because different resource types live in different API groups (Architecture masterclass §18), a Role must specify **which group** each resource belongs to:

```yaml
rules:
  - apiGroups: [""]              # the "core" group — pods, services, configmaps, secrets
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]           # deployments, replicasets, statefulsets, daemonsets
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
```
**`apiGroups: [""]` (empty string) specifically means the core group** — a common point of confusion, since it looks like an omission rather than a deliberate value.

```bash
kubectl api-resources    # lists every resource type WITH its API group — the authoritative reference
```

---

# PART 4 — Role, ClusterRole, RoleBinding, ClusterRoleBinding

## Role — namespace-scoped permission set
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev                  # this Role ONLY has meaning within the "dev" namespace
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```
This Role, by itself, grants **nobody** anything — it's purely a named, reusable permission template, inert until bound to a subject.

## RoleBinding — attaches subjects to a Role, within one namespace
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader
  namespace: dev
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
- `subjects` — WHO (a User, Group, or ServiceAccount — §5)
- `roleRef` — WHICH permission set this binding grants
- **Namespace of the RoleBinding itself determines the scope** — this grant only applies within `dev`, even though "alice" as an identity is cluster-wide; alice has zero pod-reading permission in any other namespace unless a separate binding grants it there too

## ClusterRole — the same shape, but reusable cluster-wide (or for cluster-scoped resources)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader          # NOT namespaced — ClusterRoles never are
rules:
  - apiGroups: [""]
    resources: ["nodes"]      # Nodes are CLUSTER-scoped resources — a Role could never grant this at all
    verbs: ["get", "list"]
```
**Why ClusterRole is sometimes mandatory, not just a convenience:** some resources (`nodes`, `persistentvolumes`, `namespaces` themselves, `clusterroles`) have **no namespace at all** — a plain `Role` is structurally incapable of granting permission on them, since a Role's scope is always exactly one namespace. Any permission touching a genuinely cluster-scoped resource *must* go through a ClusterRole.

## ClusterRoleBinding — grants a ClusterRole cluster-wide
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: alice-node-reader
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

## The fourth, commonly-missed combination: ClusterRole + RoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader-in-dev
  namespace: dev              # <- RoleBinding, so scope is limited to "dev"
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole            # <- referencing a CLUSTER-scoped permission SET
  name: generic-pod-reader      #    (defined once, reusable everywhere)
  apiGroup: rbac.authorization.k8s.io
```
**This is an extremely common, powerful production pattern:** define a `ClusterRole` **once** (e.g., a generic "pod-reader" permission set), then grant it to different users in different namespaces via **separate RoleBindings**, each scoping that same reusable permission set to just one namespace — avoiding having to redefine an identical `Role` object by hand in every namespace that needs the same permissions.

```
┌─────────────────────────────────────────────────────────────┐
│  ClusterRole "generic-pod-reader" (defined ONCE, cluster-wide)  │
└───────────┬─────────────────────────┬─────────────────────┘
            │                           │
   RoleBinding (namespace: dev)   RoleBinding (namespace: staging)
            │                           │
            ▼                           ▼
   Grants pod-read ONLY in "dev"   Grants pod-read ONLY in "staging"
   (to whichever subjects each RoleBinding names — could be
    different people/teams per namespace, sharing the same
    underlying permission definition)
```

---

# PART 5 — SUBJECTS: USERS, GROUPS, SERVICEACCOUNTS

## Users and Groups
Kubernetes has **no User or Group API objects at all** — these are purely identities asserted by whatever authentication mechanism is in front of the API server (a client certificate's Common Name/Organization fields, an OIDC token's claims). RBAC references them by name, but there is nothing to `kubectl get users` — they exist only as strings that authentication produces and RBAC subsequently checks against.

## ServiceAccounts — the one subject type that IS a real Kubernetes object
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: dev
```
Every Pod runs **as** some ServiceAccount (a default one is auto-provisioned per namespace if none is specified explicitly) — this is the identity a **process inside a Pod** uses when it calls the Kubernetes API itself (a controller you wrote, a CI/CD tool running inside the cluster, `kubectl` invoked from within a Pod).

```yaml
# on the Pod itself:
spec:
  serviceAccountName: my-app-sa
```
```yaml
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: dev              # ServiceAccounts ALWAYS need their namespace specified in a binding,
                                  # since the same NAME could exist in multiple namespaces as
                                  # genuinely different ServiceAccount objects
```
**This directly connects to the Secrets guide's most important RBAC-adjacent security fact:** every Pod, by default, gets its ServiceAccount's token mounted automatically — so **any process running inside that Pod effectively inherits whatever RBAC permissions that ServiceAccount has been granted**, which is exactly why scoping ServiceAccount RBAC tightly (and disabling auto-mounting where a Pod never needs to call the API at all, via `automountServiceAccountToken: false`) is a genuine, high-leverage security control, not a nitpick.

---

# PART 6 — YAML EXAMPLES: FIVE REAL-WORLD ROLES

## 1. Read-only user
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-only
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["*"]              # every resource type in these groups
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bob-read-only
subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: read-only
  apiGroup: rbac.authorization.k8s.io
```
- `resources: ["*"]` — wildcard, matches every resource type within the listed API groups
- `verbs` limited to read-only (`get`/`list`/`watch`) — bob can inspect anything in these groups but create/modify/delete nothing at all

## 2. Developer (namespace-scoped, broader verbs, no Secrets access)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # NOTE: "secrets" deliberately OMITTED — per the Secrets guide's
  # RBAC-scoping principle, developers get broad workload permissions
  # but NOT blanket secret access
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-developer
  namespace: dev
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```
- `subjects: kind: Group` — binds an entire group at once, rather than naming individuals one by one — the standard, scalable pattern for team-based access, since adding a new team member only requires updating the identity provider's group membership, not touching any Kubernetes RBAC object at all

## 3. Namespace administrator
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: team-a
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]                  # full control...
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-admins
  namespace: team-a
subjects:
  - kind: Group
    name: team-a-leads
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io
```
- `verbs: ["*"]` within a **Role** (not ClusterRole) — this is full control, but **strictly bounded to `team-a`'s namespace** — this subject cannot touch anything in `team-b`, cannot see cluster-scoped resources (nodes, PVs, other namespaces) at all — this is precisely the safe way to grant "admin-like" power without it becoming actual cluster-admin

## 4. Pod reader (minimal, single-purpose)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-only
  namespace: monitoring
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["pods/log"]        # a SUBRESOURCE — logs require explicit, SEPARATE permission
    verbs: ["get"]
```
- `pods/log` — subresources (logs, exec, status, scale) require their **own** explicit RBAC entry, distinct from the parent resource — granting `get` on `pods` alone does **not** implicitly grant `kubectl logs` access; this is a very common real-world gotcha for anyone building a minimal monitoring/debugging role

## 5. Deployment manager (CI/CD pipeline pattern)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]           # read-only, for verifying rollout status
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-cd-deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-cd-deployment-manager
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ci-cd-deployer
    namespace: production
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```
- No `create`/`delete` on Deployments — deliberately scoped to **updating existing** Deployments only (the exact operation a rollout pipeline performs via `kubectl set image`/`kubectl apply` on an already-existing object) — this pipeline's credentials, if ever leaked, cannot create brand-new workloads or delete existing ones, only push updates to what's already there

---

# PART 7 — kubectl auth can-i

The direct, authoritative way to check what a permission set actually allows, without trial and error:

```bash
kubectl auth can-i create deployments --namespace=dev
kubectl auth can-i delete secrets --namespace=production --as=bob
kubectl auth can-i get pods --as=system:serviceaccount:dev:my-app-sa
kubectl auth can-i '*' '*' --as=alice          # "can alice do literally anything, anywhere?"
kubectl auth can-i --list --namespace=dev       # everything the CURRENT identity can do here
kubectl auth can-i --list --as=bob --namespace=dev
```
- `--as` — impersonates another identity for the check (requires the caller to itself have `impersonate` permission — worth noting this is itself an RBAC-gated capability, not a universal debugging bypass)
- `--list` — dumps every permission the given identity has in the specified scope, the single fastest way to audit "what can this actually do" without reading every Role/Binding by hand

```bash
# real-world debugging flow: a CI pipeline's deploy step is failing with 403
kubectl auth can-i update deployments --namespace=production \
  --as=system:serviceaccount:production:ci-cd-deployer
# → "no" confirms an RBAC gap; "yes" means the 403 has some OTHER cause
# (wrong ServiceAccount actually attached to the Pod, wrong namespace, etc.)
```

---

# PART 8 — DIAGRAMS

## The full decision, visually
```
             "Can alice DELETE a POD in the DEV namespace?"
                              │
                              ▼
        Find every RoleBinding/ClusterRoleBinding naming "alice"
        (directly, or via a Group alice belongs to) that is either:
          - a RoleBinding IN the "dev" namespace, OR
          - a ClusterRoleBinding (applies everywhere, including "dev")
                              │
                              ▼
        For each such binding, look up its referenced Role/ClusterRole's
        rules — does ANY rule match: apiGroup="", resource="pods", verb="delete"?
                              │
              ┌────────────────┴────────────────┐
              ▼                                    ▼
          YES, at least one rule matches      NO rule matches, anywhere
              │                                    │
              ▼                                    ▼
          ALLOWED (RBAC is purely ADDITIVE —     DENIED — 403 Forbidden
          if ANY applicable binding grants it,   (there is NO explicit "deny"
          it's allowed — there's no "deny"        rule type in RBAC at all;
          rule type to override this)              absence of a grant IS the deny)
```

## RBAC is purely additive — no explicit deny
**This is the single most important RBAC design fact to internalize:** you cannot write a rule that says "deny X." Permissions only ever accumulate across every applicable binding — the only way to prevent an action is to ensure **no** binding anywhere grants it. This is precisely why auditing "what can this identity NOT do" is fundamentally a process of enumerating everything they CAN do (across every Role/ClusterRoleBinding that applies to them) and confirming the dangerous thing isn't among them — there's no single rule you can point to that "blocks" something.

---

# PART 9 — SECURITY BEST PRACTICES

## Least privilege
Grant the minimum verb/resource/namespace combination that lets a subject do its actual job — not "give broad access and restrict later," which almost never actually happens in practice once something works. The Deployment Manager example (§6, item 5) is a concrete template: `update`/`patch` only, no `create`/`delete`, scoped to one resource type, in one namespace.

## Avoiding cluster-admin
```bash
kubectl get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name=="cluster-admin")'
```
`cluster-admin` is a built-in ClusterRole granting `verbs: ["*"]` on `resources: ["*"]` in `apiGroups: ["*"]`, cluster-wide — genuinely unrestricted control over everything, including RBAC itself (a cluster-admin can grant themselves or anyone else any other permission, trivially). It should be bound to as few identities as possible — ideally just a small number of humans for genuine break-glass emergencies, and audited on a recurring basis, never handed out routinely "to make something work."

## Namespace isolation
RBAC's namespace scoping (Role/RoleBinding) is one pillar of multi-tenancy — but remember it's **only** an API-level boundary; it says nothing about network reachability (that's NetworkPolicy, from the Networking guide) or resource fairness (that's ResourceQuota, from the Resource Limits guide). Genuine tenant isolation requires **all three** layered together — RBAC alone stops a team from reading/editing another team's *objects*, but doesn't stop their Pods from reaching each other over the network or consuming all shared node capacity.

## ServiceAccount permissions
- **Never use the `default` ServiceAccount** for anything with real permissions — it's auto-created in every namespace and easy to accidentally grant broad permissions to without realizing every unspecified Pod in that namespace inherits them
- **Set `automountServiceAccountToken: false`** on Pods (or the ServiceAccount itself) that never call the Kubernetes API at all — reduces the blast radius of a container compromise to "no API credentials available inside it whatsoever"
- **One ServiceAccount per application/purpose**, not one shared broad ServiceAccount reused across many unrelated workloads — mirrors the "least privilege" principle applied specifically to machine identities rather than human ones

---

# PART 10 — TROUBLESHOOTING

**"Forbidden" errors despite believing the right RBAC is in place**
```bash
kubectl auth can-i <verb> <resource> --namespace=<ns> --as=<identity>
```
Always start here — it's authoritative and instant, versus manually re-reading YAML and hoping you've traced every applicable binding correctly by eye.

**A ServiceAccount's Pod gets 403s calling the API**
Confirm the Pod is actually running as the ServiceAccount you think it is:
```bash
kubectl get pod <pod> -o jsonpath='{.spec.serviceAccountName}'
```
A very common mistake: creating a correctly-scoped RoleBinding for `my-app-sa`, while the Pod itself is still silently running as `default` because `serviceAccountName` was never actually set on the Pod spec.

**A RoleBinding "isn't working"**
Check for a **Role vs ClusterRole** mismatch in `roleRef.kind` — a RoleBinding referencing a `ClusterRole` that doesn't actually exist (typo'd name, or it's actually a `Role` in a different namespace) silently grants nothing, with no obvious error pointing at the exact cause beyond the eventual 403.

**Someone has more access than expected**
Remember RBAC is purely additive (Part 8) — check for **group membership** granting access via a completely different binding than the one you were looking at; a user can accumulate permissions from their own direct bindings, plus every group they belong to, plus any binding naming `system:authenticated` or similarly broad built-in groups.

---

# PART 11 — REAL-WORLD SCENARIOS

- **A compromised CI/CD ServiceAccount with `cluster-admin`** is one of the most common real-world Kubernetes security incidents — a leaked pipeline credential with unrestricted cluster access turns a single leaked secret into full cluster compromise; scoping that ServiceAccount to exactly the Deployment Manager pattern from §6 instead would have contained the blast radius to "can update Deployments in one namespace."
- **A support/on-call engineer granted `view` (a built-in, broad read-only ClusterRole) cluster-wide** so they can debug incidents anywhere, without ever being handed write access anywhere — a genuinely common and reasonable use of a broad-but-read-only ClusterRoleBinding.
- **Namespace-per-team with a shared `ClusterRole` + per-namespace `RoleBinding`s** (Part 4's pattern) is the standard way platform teams scale RBAC across dozens of teams without redefining the same permission set by hand in every namespace.

---

# PART 12 — MINIKUBE LABS

### Lab 1: Prove RBAC's additive-only nature
```bash
minikube start
kubectl create namespace dev
kubectl create serviceaccount limited-sa -n dev
kubectl auth can-i get pods -n dev --as=system:serviceaccount:dev:limited-sa
# → no
kubectl create role pod-reader -n dev --verb=get,list,watch --resource=pods
kubectl create rolebinding limited-sa-binding -n dev \
  --role=pod-reader --serviceaccount=dev:limited-sa
kubectl auth can-i get pods -n dev --as=system:serviceaccount:dev:limited-sa
# → yes
kubectl auth can-i delete pods -n dev --as=system:serviceaccount:dev:limited-sa
# → still no — get/list/watch were granted, delete never was
```

### Lab 2: Subresource gotcha
```bash
kubectl run test-pod --image=nginx -n dev
kubectl auth can-i get pods/log -n dev --as=system:serviceaccount:dev:limited-sa
# → no, even though "get pods" is allowed — logs are a separate subresource
```

### Lab 3: ClusterRole + RoleBinding pattern
```bash
kubectl create clusterrole generic-reader --verb=get,list,watch --resource=pods,deployments
kubectl create namespace staging
kubectl create rolebinding staging-reader -n staging \
  --clusterrole=generic-reader --serviceaccount=dev:limited-sa
kubectl auth can-i get deployments -n staging --as=system:serviceaccount:dev:limited-sa
# → yes, in staging
kubectl auth can-i get deployments -n dev --as=system:serviceaccount:dev:limited-sa
# → no — the SAME ClusterRole was never bound in "dev" for this permission
```

### Lab 4: Audit an identity's full permission set
```bash
kubectl auth can-i --list --as=system:serviceaccount:dev:limited-sa -n dev
kubectl auth can-i --list --as=system:serviceaccount:dev:limited-sa -n staging
```

---

# INTERVIEW QUESTIONS

**Fundamentals**
1. What's the difference between authentication and authorization?
2. State the "who → what → which resource → which scope" model in your own words.
3. Does Kubernetes have a User object? Where does user identity actually come from?

**Core objects**
4. What's the difference between a Role and a ClusterRole?
5. What's the difference between a RoleBinding and a ClusterRoleBinding?
6. Why must some resources always use a ClusterRole, never a plain Role?
7. Explain the ClusterRole + RoleBinding pattern and why it's useful at scale.

**Mechanics**
8. What does `apiGroups: [""]` mean, and why does it look confusing at first?
9. Why does granting `get` on `pods` not automatically grant `kubectl logs` access?
10. Is RBAC additive, subtractive, or both? Can you write an explicit "deny" rule?

**ServiceAccounts**
11. What's the difference between a User/Group subject and a ServiceAccount subject, architecturally?
12. Why does every Pod need its ServiceAccount specified explicitly in a RoleBinding's namespace field?
13. What security risk does the `default` ServiceAccount pose if left unmanaged?
14. What does `automountServiceAccountToken: false` protect against?

**Best practices**
15. Why is `cluster-admin` dangerous even when "only used occasionally"?
16. Why isn't RBAC alone sufficient for true multi-tenant namespace isolation?

**Troubleshooting (scenario-based)**
17. A user gets 403 Forbidden despite what looks like a correctly configured RoleBinding — what's your first diagnostic command?
18. A Pod's application code gets 403s calling the Kubernetes API — what are your first two checks?
19. You suspect a user has more access than intended — why can't you find this by reading just one RoleBinding?
20. How would you verify, with a single command, exactly what a given ServiceAccount can do across an entire namespace?

---

*This guide covers Kubernetes RBAC from the authentication/authorization split through Role/ClusterRole mechanics, ServiceAccount security, and production least-privilege design — the complete arc from beginner to security-architect-level authorization design.*
