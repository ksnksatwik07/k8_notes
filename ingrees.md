# Kubernetes Ingress — Complete Mastery Guide
### From Absolute Beginner to Production-Grade HTTP Routing

---

# PART 1 — WHY INGRESS EXISTS

## 1. The Problem After Services

A `LoadBalancer` Service gets you one external IP per Service. That's fine for one app. It falls apart the moment you have 20 microservices, each needing its own public HTTP(S) endpoint — you'd be paying for and managing 20 cloud load balancers, none of which understand HTTP concepts like hostnames, URL paths, or TLS termination. They just forward raw TCP/UDP.

**Ingress** is Kubernetes' answer: a single object that describes **HTTP/HTTPS routing rules** — "requests for `api.example.com` go to the `api` Service, requests for `example.com/blog` go to the `blog` Service" — all fronted by **one** entry point.

## 2. What Ingress Actually Is

Ingress is **not** a running process. It is a Kubernetes API object — a declarative *set of routing rules*. By itself it does nothing. It requires a separate piece of software, the **Ingress Controller**, to read those rules and actually implement them.

This is the single most important distinction to internalize: **Ingress = the rulebook. Ingress Controller = the engine that enforces the rulebook.** Creating an Ingress object with no controller running is a no-op — nothing listens, nothing routes.

## 3. Ingress vs Service — the core distinction

| | Service | Ingress |
|---|---|---|
| OSI Layer | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| Understands hostnames/paths? | No | Yes |
| Understands TLS/SNI? | No (LoadBalancer can pass-through raw TLS, nothing more) | Yes — can terminate TLS |
| External IPs needed | One per `LoadBalancer` Service | One total, shared by all apps |
| Routing granularity | Whole Service, whole port | Host + path combinations |

## 4. Layer 4 vs Layer 7 — why it matters here

- **Layer 4 (Services)**: routing decisions are based purely on IP address and port. The load balancer/kube-proxy has zero visibility into what's inside the packet — it can't tell `example.com/api` apart from `example.com/blog`, because both look identical at L4 (same destination IP:port).
- **Layer 7 (Ingress)**: the controller actually terminates the HTTP connection, reads the `Host` header and URL path, and *then* decides where to forward — enabling routing rules that would be completely invisible at L4.

This is why Ingress must sit **in front of** Services, not replace them — Ingress hands off the final leg of the journey to a normal ClusterIP Service, which then does its usual L4 job.

---

# PART 2 — INGRESS CONTROLLERS

## 5. Ingress Controller

The controller is a normal Pod (usually run as a Deployment, exposed via its own `LoadBalancer` or `NodePort` Service) that:

1. Watches the API server for `Ingress` objects
2. Translates them into its own internal configuration (e.g., an NGINX config file, or an Envoy xDS config)
3. Reloads/updates itself to apply the new rules
4. Actually terminates and proxies the HTTP(S) traffic

Kubernetes ships **no built-in controller** — you must install one. Popular choices: **ingress-nginx** (community NGINX-based, most widely deployed), NGINX Inc.'s own controller, Traefik, HAProxy, Contour, and cloud-native ones (AWS ALB Ingress Controller, GKE Ingress).

## 6. NGINX Ingress (ingress-nginx)

The de facto default. Internally it's an NGINX process whose `nginx.conf` is regenerated every time Ingress objects change, watching via the Kubernetes API and using Lua (via OpenResty) for dynamic reloading without dropping connections.

```bash
# Minikube
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

On Minikube, the addon deploys the controller **and** configures it to bind on the node's IP, so `minikube ip` becomes your entry point.

On a real cloud cluster, you'd typically `helm install` it, and it provisions its own `LoadBalancer` Service — this is the **one and only** cloud load balancer your entire cluster needs, no matter how many apps sit behind it.

---

# PART 3 — THE FULL REQUEST PATH

```
Internet
   │
   ▼
Load Balancer               ← cloud LB (or MetalLB/minikube tunnel on-prem);
   │                          this is a plain L4 forwarder, knows nothing about HTTP
   ▼
Ingress Controller           ← real HTTP(S) server (NGINX/Envoy/etc.), terminates
   │                           TLS here, reads Host + Path, matches Ingress rules
   ▼
Service (ClusterIP)          ← ordinary L4 VIP, load-balances across Pods
   │
   ▼
Pods                         ← your application containers
```

Concretely, for `https://shop.example.com/cart`:

1. DNS resolves `shop.example.com` → the Load Balancer's public IP
2. TCP/TLS connection established to the Load Balancer, which forwards raw bytes to the Ingress Controller (L4 pass-through — the LB itself doesn't decrypt anything)
3. The Ingress Controller terminates TLS (decrypts using its configured certificate for `shop.example.com`)
4. It reads the `Host: shop.example.com` header and the path `/cart`
5. It matches these against Ingress rules, finds the target Service (e.g., `cart-service`)
6. It opens a normal internal HTTP connection to `cart-service`'s ClusterIP
7. The Service's usual kube-proxy/eBPF routing picks a ready Pod
8. The Pod handles the request and the response retraces the path

---

# PART 4 — ROUTING MECHANICS

## 7. Host-based routing

Route by the `Host` HTTP header — different domains/subdomains go to different Services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing-example
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-service
                port:
                  number: 80
    - host: blog.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: blog-service
                port:
                  number: 80
```
Any request with `Host: shop.example.com` → `shop-service`. Any with `Host: blog.example.com` → `blog-service`. Both share the same controller, same public IP, same certificate infrastructure.

## 8. Path-based routing

Route by URL path prefix under a **single** host.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing-example
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: api-service, port: { number: 8080 } }
          - path: /blog
            pathType: Prefix
            backend:
              service: { name: blog-service, port: { number: 80 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: frontend-service, port: { number: 80 } }
```
`pathType` matters:
- `Prefix` — matches `/api`, `/api/`, `/api/users`, etc. (segment-aware prefix match)
- `Exact` — matches only that literal path, nothing else
- `ImplementationSpecific` — controller decides (NGINX treats it like a regex if annotations enable that)

Order matters for overlapping prefixes — most controllers evaluate longest/most-specific match first, but always list more specific paths and verify controller behavior rather than assuming.

---

# PART 5 — TLS

## 9. TLS termination

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.example.com
      secretName: shop-tls-cert       # a Secret of type kubernetes.io/tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: shop-service, port: { number: 80 } }
```

- `tls.hosts` — which hostnames this cert block covers (must match SNI at connection time)
- `secretName` — a Kubernetes Secret holding `tls.crt` and `tls.key`
- The controller uses this to answer the TLS handshake **before** it even looks at HTTP headers (SNI happens at the TLS layer, one level below HTTP)

## 10. Certificates

You provide the cert/key yourself, or automate it. The overwhelmingly common production pattern is **cert-manager**, which watches Ingress objects (or its own `Certificate` CRDs) and automatically requests/renews certificates from Let's Encrypt via ACME.

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```
With this annotation present, cert-manager sees the Ingress, requests a cert for the listed hosts, completes an HTTP-01 or DNS-01 challenge, and writes the resulting cert straight into the `secretName` Secret referenced above — fully automated renewal thereafter.

Manually creating a TLS secret:
```bash
kubectl create secret tls shop-tls-cert --cert=cert.pem --key=key.pem
```

---

# PART 6 — MISC MECHANICS

## 11. Default backend

If a request's `Host`/path matches **no** rule in any Ingress, the controller falls back to a **default backend** — typically a simple Pod that returns a 404 page.

```yaml
spec:
  defaultBackend:
    service:
      name: default-404-service
      port:
        number: 80
```
Most controllers (ingress-nginx included) also ship their own built-in default backend if you don't specify one, which just returns a bare `404`.

## 12. Annotations

Because the Ingress **spec** is intentionally generic (portable across controllers), controller-specific behavior is bolted on via annotations — a well-known escape hatch.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/limit-rps: "10"
```
These are entirely controller-specific — an annotation meaningful to ingress-nginx does nothing on Traefik. Always check your controller's own annotation reference docs.

## 13. ingressClassName

When multiple controllers run in the same cluster (e.g., an internal one and an internet-facing one), `ingressClassName` tells Kubernetes **which controller** should implement a given Ingress object.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx    # <- must match an IngressClass name
```
An Ingress with no `ingressClassName` and no default `IngressClass` marked (`ingressclass.kubernetes.io/is-default-class: "true"`) is ignored by controllers that require explicit class matching — this is a very common "my Ingress just does nothing" bug on newer clusters.

## 14. Load balancing

Once the controller picks a Service, the actual Pod selection follows the same L4 Service load-balancing rules covered in the Services guide (round-robin by default for NGINX, configurable via annotations like `nginx.ingress.kubernetes.io/upstream-hash-by` for consistent hashing).

Note: **NGINX Ingress talks to Pod IPs directly**, bypassing kube-proxy's Service VIP entirely in most configurations (it reads Endpoints/EndpointSlices itself and load-balances across Pod IPs at the HTTP layer) — this gives it finer-grained L7 load balancing than a plain Service could offer alone.

## 15. Rewrite rules

Rewrite the path before forwarding upstream — useful when your app expects paths different from what's exposed publicly.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service: { name: api-service, port: { number: 80 } }
```
A request to `/api/users` is captured by the regex groups `(/|$)(.*)`, and `$2` (`users`) is substituted into the rewrite target — so the backend receives `/users`, not `/api/users`.

## 16. HTTP/HTTPS handling

Standard pattern: expose both port 80 and 443, and force redirect.
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"       # auto-redirect HTTP → HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```
Without a `tls:` block defined, ingress-nginx serves plain HTTP only; add `tls:` and it automatically also listens on 443 with that cert and (by default) redirects 80 → 443.

## 17. DNS relationship

Ingress has **no direct DNS integration** by default — you (or `external-dns`, a common add-on) must separately point your domain's DNS record at the Ingress Controller's external IP/hostname.

```
shop.example.com   A     <ingress-controller-external-ip>
```
With **external-dns** installed, it watches Ingress objects' `host` fields and automatically creates/updates the matching DNS records in your provider (Route53, Cloudflare, etc.) — removing the manual step.

---

# PART 7 — YAML COOKBOOK

### Single application
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: single-app
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: app-service, port: { number: 80 } }
```

### Multiple applications (host-based)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-app
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: shop-service, port: { number: 80 } } }
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: admin-service, port: { number: 80 } } }
```

### Host-based routing (repeated for clarity — see Part 4 §7)

### Path-based routing (repeated for clarity — see Part 4 §8)

### HTTPS/TLS
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: https-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [app.example.com]
      secretName: app-tls-cert
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: app-service, port: { number: 80 } } }
```

### Supporting resources for all examples above
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector: { app: demo }
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  replicas: 2
  selector: { matchLabels: { app: demo } }
  template:
    metadata: { labels: { app: demo } }
    spec:
      containers:
        - name: web
          image: hashicorp/http-echo
          args: ["-text=hello", "-listen=:8080"]
          ports: [{ containerPort: 8080 }]
```

---

# PART 8 — MINIKUBE SETUP

```bash
minikube start
minikube addons enable ingress
kubectl get pods -n ingress-nginx      # wait for controller Running

# apply your Ingress + Service + Deployment
kubectl apply -f ingress-cookbook.yaml

# Minikube doesn't have real DNS, so map the hostname locally:
echo "$(minikube ip) app.example.com" | sudo tee -a /etc/hosts

curl http://app.example.com
```
For TLS testing on Minikube, generate a self-signed cert:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout key.pem -out cert.pem -subj "/CN=app.example.com"
kubectl create secret tls app-tls-cert --cert=cert.pem --key=key.pem
curl -k https://app.example.com
```

---

# PART 9 — TROUBLESHOOTING PLAYBOOK

## 404
- No `Ingress` rule matches the request's `Host`/path → check `kubectl describe ingress` for the actual rules registered
- `ingressClassName` mismatch — the Ingress exists but **no controller is actually watching it** (see "wrong ingress class" below); the request may be hitting the controller's built-in default backend, which returns 404 for anything unmatched
- Typo in `host:` field vs the actual request's `Host` header

## 502 (Bad Gateway)
- The controller reached the Service but the **Pod** refused the connection or crashed mid-request
- `targetPort` mismatch on the underlying Service (same root cause as plain Service 502s)
- Backend Pod is overwhelmed / OOM-killed mid-request
- Check: `kubectl logs -n ingress-nginx <controller-pod>` for upstream connection errors

## 503 (Service Unavailable)
- The Service behind the Ingress has **zero ready endpoints** — same root causes as a bare Service with no endpoints (bad selector, failing readiness probes, scaled to 0)
- ingress-nginx specifically returns 503 from its own default backend when the resolved Service has no valid upstreams

## DNS failure
- The domain doesn't actually point at the Ingress Controller's external IP — verify with `dig shop.example.com` and compare to `kubectl get svc -n ingress-nginx`
- On Minikube: forgot to add the `/etc/hosts` entry (there's no real DNS in a local cluster)
- `external-dns` misconfigured or missing required cloud provider IAM permissions to write records

## TLS failure
- `secretName` doesn't exist, or exists but doesn't contain valid `tls.crt`/`tls.key` — `kubectl describe secret <name>`
- Certificate's Common Name/SAN doesn't include the requested hostname — browser/`curl` will show a hostname mismatch error
- cert-manager stuck: `kubectl describe certificate <name>` and `kubectl describe certificaterequest` to see ACME challenge failures (commonly: HTTP-01 challenge blocked because the Ingress isn't actually reachable from the internet yet)

## Wrong ingress class
- Symptom: Ingress object exists, looks perfectly correct, but **nothing happens** — no error, no logs, no traffic
- Check: `kubectl get ingress -o wide` — the `CLASS` column should show your controller's class name, not `<none>`
- Fix: set `ingressClassName` explicitly, or mark one `IngressClass` as default with the annotation `ingressclass.kubernetes.io/is-default-class: "true"`

## Service unavailable (from the controller's perspective)
- The Ingress correctly matched a rule, but the referenced **Service name/port doesn't exist** — check for typos between the Ingress's `backend.service.name` and the actual Service metadata name
- Port name/number mismatch — if using a named port, confirm the Service actually defines a port with that name

---

# PART 10 — PRODUCTION CONSIDERATIONS

1. **Run exactly one Ingress Controller Service of type `LoadBalancer`** — every app's Ingress shares it; don't provision per-app load balancers.
2. **Always set explicit `ingressClassName`.** Relying on an implicit default class is fragile once a second controller is introduced.
3. **Automate TLS with cert-manager.** Manual certificate rotation is a guaranteed future outage.
4. **Set resource requests/limits on the controller Pods** — it's now a single point of failure for *all* HTTP traffic into the cluster; under-provisioning it degrades everything at once.
5. **Run the controller with multiple replicas + PodDisruptionBudget** for HA; a single-replica Ingress Controller is a common, easily-avoidable outage cause.
6. **Rate-limit and cap body size at the Ingress layer** (`nginx.ingress.kubernetes.io/limit-rps`, `proxy-body-size`) to protect backend Pods from abuse before it ever reaches them.
7. **Prefer WAF/security annotations or a dedicated API gateway** for anything internet-facing needing more than basic routing (auth, complex rate limiting, request transformation) — Ingress is deliberately minimal by design.
8. **Separate internal and external Ingress Controllers** (two `IngressClass`es) if you have both internal-only and public-facing services — don't expose internal admin tools through the same public LB.
9. **Watch controller logs and metrics** (ingress-nginx exposes Prometheus metrics) — upstream latency and error-rate dashboards catch backend issues before users report them.
10. **Version-pin the controller** and test upgrades in staging — config-generation logic changes between major versions have historically caused subtle routing regressions.

---

# PART 11 — INTERVIEW QUESTIONS

**Conceptual**
1. What is the difference between an Ingress object and an Ingress Controller?
2. Why can't a plain `LoadBalancer` Service do what Ingress does?
3. Explain the difference between Layer 4 and Layer 7 routing in the context of Kubernetes.

**Routing**
4. Walk through how host-based routing differs from path-based routing.
5. What's the difference between `Prefix`, `Exact`, and `ImplementationSpecific` `pathType`s?
6. What happens when a request matches no rule in any Ingress?

**TLS**
7. Where does TLS termination happen in the full request path, and why does that matter for Layer 7 routing?
8. How does cert-manager automate certificate issuance for an Ingress?
9. What's a `kubernetes.io/tls` Secret expected to contain?

**Configuration**
10. What is `ingressClassName` for, and what happens if it's omitted with no default class set?
11. Why are so many Ingress features controller-specific annotations instead of part of the core spec?

**Full flow**
12. Trace a request from `Internet → Load Balancer → Ingress Controller → Service → Pod`, explaining what happens at each hop.
13. Does the Ingress Controller use kube-proxy's Service routing, or does it bypass it? Explain.

**Troubleshooting (scenario-based)**
14. Users get a 502 on one specific endpoint — walk through your debugging steps.
15. An Ingress object looks correct but traffic never reaches it — what's the first thing you check?
16. A newly requested Let's Encrypt certificate is stuck pending — what are the likely causes?

---

*This guide covers Kubernetes Ingress from the Service-vs-Ingress distinction through TLS automation, controller internals, and full production troubleshooting — the complete arc from beginner to Ingress networking expert.*
