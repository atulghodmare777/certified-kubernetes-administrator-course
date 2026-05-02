# Ingress Limitations vs Gateway API Solutions

| # | Ingress Limitation | Problem in Practice | How Gateway API Solves It |
|---|-------------------|--------------------|-----------------------------|
| 1 | **Host & path routing only** | Cannot route traffic based on HTTP headers, request methods, or query parameters without hacky annotations | Native support for header-based, method-based, and query parameter routing in `HTTPRoute` spec |
| 2 | **No standard traffic splitting** | Canary deployments and blue/green rollouts require controller-specific workarounds or external tools | Built-in weighted backend routing — split traffic between services by percentage directly in `HTTPRoute` |
| 3 | **Annotation overload** | Features like timeouts, retries, rate limiting, and TLS config are set via custom annotations that differ per controller, making configs non-portable | All advanced features are first-class fields in the API spec — no annotations needed, works the same across controllers |
| 4 | **Monolithic resource model** | Listener config, TLS, and routing rules are all jammed into one Ingress object — hard to manage, hard to delegate | Split into three clear resources: `GatewayClass` (infra type) → `Gateway` (listener/TLS) → `HTTPRoute` (routing rules) |
| 5 | **No role-based ownership** | Platform teams and app teams cannot independently manage their parts — one team must touch the same object | Designed for multi-team use — cluster admins own `GatewayClass`, platform teams own `Gateway`, app devs own `HTTPRoute` |
| 6 | **HTTP/HTTPS only** | Cannot handle TCP, UDP, gRPC, or raw TLS traffic — forces teams to use separate solutions per protocol | Supports `HTTPRoute`, `TCPRoute`, `UDPRoute`, `TLSRoute`, and `GRPCRoute` under a single unified API |
| 7 | **No cross-namespace routing** | An Ingress resource can only route to services in its own namespace — multi-team service exposure is painful | `HTTPRoute` can reference backends in other namespaces using `ReferenceGrant` for explicit, secure delegation |
| 8 | **No extensibility standard** | Every controller (NGINX, Traefik, HAProxy) extends Ingress differently — switching controllers means rewriting configs | Structured extension points via `ExtensionRef` and request/response `filters` — extensions are portable and predictable |


Infrastructure providers > GatewayClass Resource
Cluster Operators > Gateway resource
Application Developers > HTTPRoutes, TCPRoutes, GRPCRoutes, UDPRoute

## Gateway class

```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```
We have to provide controller name in this which we need to deploy

## Gateway
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```
## HTTP route
```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend
spec:
  parentRefs:
    - name: eg
  hostnames:
    - "www.example.com"
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: backend
          port: 3000
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /
```



# Ingress vs Gateway API

| Ingress | Gateway API |
|---------|-------------|
| <pre>apiVersion: networking.k8s.io/v1<br>kind: Ingress<br>metadata:<br>  name: secure-app<br>  annotations:<br>    nginx.ingress.kubernetes.io/ssl-redirect: "true"<br>    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"<br>spec:<br>  tls:<br>  - hosts:<br>      - secure.example.com<br>    secretName: tls-secret</pre> | <pre>apiVersion: gateway.networking.k8s.io/v1<br>kind: Gateway<br>metadata:<br>  name: secure-gateway<br>spec:<br>  gatewayClassName: example-gc<br>  listeners:<br>  - name: https<br>    port: 443<br>    protocol: HTTPS<br>    tls:<br>      mode: Terminate<br>      certificateRefs:<br>      - kind: Secret<br>        name: tls-secret<br>    allowedRoutes:<br>      kinds:<br>      - kind: HTTPRoute</pre> |

---
     
# Canary Traffic Split — Ingress vs Gateway API

| Ingress | Gateway API |
|---------|-------------|
| <pre>apiVersion: networking.k8s.io/v1<br>kind: Ingress<br>metadata:<br>  name: canary-ingress<br>  annotations:<br>    nginx.ingress.kubernetes.io/canary: "true"<br>    nginx.ingress.kubernetes.io/canary-weight: "20"<br>spec:<br>  rules:<br>  - http:<br>      paths:<br>      - path: /<br>        pathType: Prefix<br>        backend:<br>          service:<br>            name: app-v2<br>            port:<br>              number: 80</pre> | <pre>apiVersion: gateway.networking.k8s.io/v1<br>kind: HTTPRoute<br>metadata:<br>  name: split-traffic<br>spec:<br>  parentRefs:<br>  - name: app-gateway<br>  rules:<br>  - backendRefs:<br>     - name: app-v1<br>       port: 80<br>       weight: 80<br>     - name: app-v2<br>       port: 80<br>       weight: 20</pre> |

---

# CORS Configuration — Ingress vs Gateway API

| Ingress (NGINX + Traefik) | Gateway API (HTTPRoute) |
|---------------------------|--------------------------|
| <pre># NGINX Ingress<br>apiVersion: networking.k8s.io/v1<br>kind: Ingress<br>metadata:<br>  name: cors-ingress<br>  annotations:<br>    nginx.ingress.kubernetes.io/enable-cors: "true"<br>    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, PUT, POST"<br>    nginx.ingress.kubernetes.io/cors-allow-origin: "https://allowed-origin.com"<br>    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"<br><br># Traefik Ingress<br>apiVersion: networking.k8s.io/v1<br>kind: Ingress<br>metadata:<br>  name: traefik-ingress<br>  annotations:<br>    traefik.ingress.kubernetes.io/headers.customresponseheaders: &#124;<br>      Access-Control-Allow-Origin: '*'<br>      Access-Control-Allow-Methods: GET,POST,PUT,DELETE,OPTIONS<br>      Access-Control-Allow-Headers: Content-Type,Authorization<br>      Access-Control-Allow-Credentials: true<br>      Access-Control-Max-Age: 3600</pre> | <pre>apiVersion: gateway.networking.k8s.io/v1<br>kind: HTTPRoute<br>metadata:<br>  name: cors-route<br>spec:<br>  parentRefs:<br>  - name: my-gateway<br>  rules:<br>  - matches:<br>    - path:<br>        type: PathPrefix<br>        value: /api<br>    filters:<br>    - type: ResponseHeaderModifier<br>      responseHeaderModifier:<br>        add:<br>        - name: Access-Control-Allow-Origin<br>          value: "*"<br>        - name: Access-Control-Allow-Methods<br>          value: "GET,POST,PUT,DELETE,OPTIONS"<br>        - name: Access-Control-Allow-Headers<br>          value: "Content-Type,Authorization"<br>        - name: Access-Control-Allow-Credentials<br>          value: "true"<br>        - name: Access-Control-Max-Age<br>          value: "3600"<br>    backendRefs:<br>    - name: api-service<br>      port: 80</pre> |


## gateway is already deployed in nginx-gateway namespace now deploy gatewayclass and httproute for a service
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
```
```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
    sectionName: http
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-svc
      port: 80
```




















# Kubernetes Gateway API — Complete Study Notes (2025)

> A modern, extensible replacement for Kubernetes Ingress.
> Structured, role-aware, and protocol-agnostic traffic management for Kubernetes clusters.
>
> 📚 Official Docs: [gateway-api.sigs.k8s.io](https://gateway-api.sigs.k8s.io)

---

## Table of Contents

- [1. Why Gateway API? — Ingress Limitations](#1-why-gateway-api--ingress-limitations)
- [2. Role-Based Ownership Model](#2-role-based-ownership-model)
- [3. Installation — NGINX Gateway Controller](#3-installation--nginx-gateway-controller)
- [4. GatewayClass](#4-gatewayclass)
- [5. Gateway](#5-gateway)
- [6. HTTPRoute — Basic Routing](#6-httproute--basic-routing)
- [7. HTTPRoute — Redirects and Rewrites](#7-httproute--redirects-and-rewrites)
- [8. HTTPRoute — Header Modification](#8-httproute--header-modification)
- [9. HTTPRoute — Traffic Splitting](#9-httproute--traffic-splitting)
- [10. HTTPRoute — Request Mirroring](#10-httproute--request-mirroring)
- [11. TLS Configuration](#11-tls-configuration)
- [12. TCP, UDP, and gRPC](#12-tcp-udp-and-grpc)
- [13. CORS Configuration](#13-cors-configuration)
- [14. Real-World Example — Deploy for a Service](#14-real-world-example--deploy-for-a-service)
- [15. Quick Reference](#15-quick-reference)

---

## 1. Why Gateway API? — Ingress Limitations

Before learning Gateway API, understand **what problems it solves** in Ingress.

| # | Ingress Limitation | Problem in Practice | Gateway API Fix |
|---|-------------------|---------------------|-----------------|
| 1 | **Host & path routing only** | No header, method, or query param routing without annotations | Native `HTTPRoute` supports all match types |
| 2 | **No traffic splitting** | Canary/blue-green needs controller-specific hacks | Built-in `weight` field in `backendRefs` |
| 3 | **Annotation overload** | Timeouts, retries, TLS differ per controller — non-portable | All features as first-class typed API fields |
| 4 | **Monolithic resource** | Listener, TLS, and routing all in one object | Split into `GatewayClass` → `Gateway` → `HTTPRoute` |
| 5 | **No role separation** | Platform and app teams share the same object | Each resource maps to a distinct team role |
| 6 | **HTTP/HTTPS only** | No TCP, UDP, or gRPC support | `TCPRoute`, `UDPRoute`, `GRPCRoute` all supported |
| 7 | **No cross-namespace routing** | Ingress only routes within its own namespace | `ReferenceGrant` enables secure cross-namespace routing |
| 8 | **No extensibility standard** | Every controller extends differently — switching means rewriting | `ExtensionRef` and `filters` are portable across controllers |

### Side-by-Side YAML — TLS (Ingress vs Gateway API)

| Ingress | Gateway API |
|---------|-------------|
| <pre>apiVersion: networking.k8s.io/v1<br>kind: Ingress<br>metadata:<br>  name: secure-app<br>  annotations:<br>    nginx.ingress.kubernetes.io/ssl-redirect: "true"<br>    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"<br>spec:<br>  tls:<br>  - hosts:<br>      - secure.example.com<br>    secretName: tls-secret</pre> | <pre>apiVersion: gateway.networking.k8s.io/v1<br>kind: Gateway<br>metadata:<br>  name: secure-gateway<br>spec:<br>  gatewayClassName: example-gc<br>  listeners:<br>  - name: https<br>    port: 443<br>    protocol: HTTPS<br>    tls:<br>      mode: Terminate<br>      certificateRefs:<br>      - kind: Secret<br>        name: tls-secret<br>    allowedRoutes:<br>      kinds:<br>      - kind: HTTPRoute</pre> |

**Differences:** SSL redirect via 2 NGINX-only annotations → `protocol: HTTPS` single field. TLS mode implicit → `mode: Terminate` explicit. `secretName` only → `certificateRefs` extensible.

---

### Side-by-Side YAML — Canary Traffic Split (Ingress vs Gateway API)

| Ingress | Gateway API |
|---------|-------------|
| <pre>apiVersion: networking.k8s.io/v1<br>kind: Ingress<br>metadata:<br>  name: canary-ingress<br>  annotations:<br>    nginx.ingress.kubernetes.io/canary: "true"<br>    nginx.ingress.kubernetes.io/canary-weight: "20"<br>spec:<br>  rules:<br>  - http:<br>      paths:<br>      - path: /<br>        pathType: Prefix<br>        backend:<br>          service:<br>            name: app-v2<br>            port:<br>              number: 80</pre> | <pre>apiVersion: gateway.networking.k8s.io/v1<br>kind: HTTPRoute<br>metadata:<br>  name: split-traffic<br>spec:<br>  parentRefs:<br>  - name: app-gateway<br>  rules:<br>  - backendRefs:<br>     - name: app-v1<br>       port: 80<br>       weight: 80<br>     - name: app-v2<br>       port: 80<br>       weight: 20</pre> |

**Differences:** Canary via 2 annotations + 2 separate Ingress files → single `HTTPRoute` with `weight` field. v1 backend implicit → explicitly declared. NGINX-only → works on any controller.

---

## 2. Role-Based Ownership Model

Gateway API is designed for **multi-team Kubernetes environments**. Each resource maps to a distinct team.

```
Infrastructure Providers  →  GatewayClass   (defines which controller to use)
       Cluster Operators  →  Gateway        (defines ports, protocols, TLS)
  Application Developers  →  HTTPRoute      (defines routing rules per service)
                             TCPRoute
                             GRPCRoute
                             UDPRoute
```

> This separation means app developers never need to touch Gateway or GatewayClass — they only manage their own routes.

---

## 3. Installation — NGINX Gateway Controller

A **controller** is required to implement Gateway API CRDs. This guide uses NGINX Gateway Fabric.

```bash
# Step 1 — Install standard Gateway API CRDs
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.6.2" | kubectl apply -f -

# Step 2 — Install experimental Gateway API CRDs
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/experimental?ref=v1.6.2" | kubectl apply -f -

# Step 3 — Install NGINX Gateway Controller via Helm
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway
```

> This installs the controller in the `nginx-gateway` namespace along with all required CRDs.

---

## 4. GatewayClass

A `GatewayClass` is a **cluster-scoped blueprint** — it tells Kubernetes which controller manages Gateways of this class.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: nginx.org/gateway-controller
```

| Field | Purpose |
|-------|---------|
| `metadata.name` | Name referenced by `Gateway.spec.gatewayClassName` |
| `controllerName` | Must match the installed controller's expected name |

> **Owned by:** Infrastructure Provider / Cluster Admin
> One cluster can have multiple `GatewayClass` resources — e.g. one for NGINX, one for Istio.

---

## 5. Gateway

A `Gateway` defines **how traffic enters the cluster** — protocols, ports, TLS, and which namespaces can attach routes.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
```

| Field | Purpose |
|-------|---------|
| `gatewayClassName` | Links to the `GatewayClass` that manages this Gateway |
| `listeners[].name` | Unique name for this listener — referenced by `HTTPRoute.sectionName` |
| `listeners[].protocol` | Protocol handled — `HTTP`, `HTTPS`, `TCP`, `UDP` |
| `listeners[].port` | Port to listen on |
| `allowedRoutes.namespaces.from` | `All` = any namespace can attach routes to this Gateway |

> **Owned by:** Cluster Operator / Platform Team

---

## 6. HTTPRoute — Basic Routing

An `HTTPRoute` defines **how HTTP traffic is forwarded** to backend services. It attaches to a `Gateway` via `parentRefs`.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: basic-route
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
    sectionName: http
  hostnames:
  - "www.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /app
    backendRefs:
    - name: my-app
      port: 80
      weight: 1
```

| Field | Purpose |
|-------|---------|
| `parentRefs` | Attaches this route to a specific Gateway listener |
| `hostnames` | Only match requests for this hostname |
| `matches.path.type` | `PathPrefix` matches `/app`, `/app/anything` |
| `matches.path.value` | Path to match |
| `backendRefs.name` | Kubernetes Service to forward traffic to |
| `backendRefs.port` | Port on the Service |

> **Owned by:** Application Developer

---

## 7. HTTPRoute — Redirects and Rewrites

### HTTP → HTTPS Redirect

Redirects all incoming HTTP traffic to HTTPS before it reaches the backend.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-redirect
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
```

| Field | Purpose |
|-------|---------|
| `type: RequestRedirect` | Filter that redirects the request |
| `requestRedirect.scheme: https` | Forces redirect to HTTPS scheme |

---

### Path Rewrite

Modifies the request path before forwarding to the backend — client sends `/old`, backend receives `/new`.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rewrite-path
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /old
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          replacePrefixMatch: /new
    backendRefs:
    - name: my-app
      port: 80
```

| Field | Purpose |
|-------|---------|
| `type: URLRewrite` | Filter that rewrites the URL path |
| `replacePrefixMatch: /new` | Replaces `/old` with `/new` before forwarding |

> `/old/page` → forwarded as `/new/page` to `my-app`.

---

## 8. HTTPRoute — Header Modification

Add, set, or remove HTTP headers on requests or responses.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: header-mod
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
  rules:
  - filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: x-env
          value: staging
    backendRefs:
    - name: my-app
      port: 80
```

| Field | Purpose |
|-------|---------|
| `type: RequestHeaderModifier` | Filter that adds/sets/removes request headers |
| `add` | Injects new headers into every forwarded request |

> Use for environment flags (`x-env: staging`), feature toggles, or tracing headers.

---

## 9. HTTPRoute — Traffic Splitting

Distribute traffic across multiple backends by **weight** — used for canary deployments and A/B testing.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: traffic-split
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
  rules:
  - backendRefs:
    - name: v1-service
      port: 80
      weight: 80
    - name: v2-service
      port: 80
      weight: 20
```

| Backend | Weight | Traffic Share |
|---------|--------|---------------|
| `v1-service` | 80 | 80% of requests |
| `v2-service` | 20 | 20% of requests |

> Weights are relative. `80 + 20 = 100` → 80% stable, 20% canary. Adjust weights to gradually shift traffic.

---

## 10. HTTPRoute — Request Mirroring

Send a **silent copy** of live traffic to a secondary service — primary service response is unaffected.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: request-mirror
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
  rules:
  - filters:
    - type: RequestMirror
      requestMirror:
        backendRef:
          name: mirror-service
          port: 80
    backendRefs:
    - name: my-app
      port: 80
```

| Field | Purpose |
|-------|---------|
| `type: RequestMirror` | Filter that duplicates every request |
| `requestMirror.backendRef` | Secondary service receiving the mirrored copy |
| `backendRefs` | Primary service — client gets response from here |

> Client ← response from `my-app`. `mirror-service` receives a copy silently for testing or analysis.

---

## 11. TLS Configuration

Terminate TLS at the Gateway level — Gateway decrypts traffic, backend services receive plain HTTP.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway-tls
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: tls-secret
    allowedRoutes:
      namespaces:
        from: All
```

| Field | Purpose |
|-------|---------|
| `protocol: HTTPS` | Listener accepts encrypted HTTPS traffic |
| `tls.mode: Terminate` | Gateway decrypts TLS — backends get plain HTTP |
| `certificateRefs.kind: Secret` | Kubernetes Secret holding the TLS cert and key |
| `certificateRefs.name` | Name of the secret — must exist before applying |

> Create the TLS secret first: `kubectl create secret tls tls-secret --cert=cert.pem --key=key.pem`

---

## 12. TCP, UDP, and gRPC

### TCP — Database Traffic

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tcp-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
  - name: tcp
    protocol: TCP
    port: 3306
    allowedRoutes:
      namespaces:
        from: All
```

> Port `3306` = MySQL default. Use for any TCP-based service (databases, message brokers).

---

### UDP — DNS Traffic

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: udp-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
  - name: udp
    protocol: UDP
    port: 53
    allowedRoutes:
      namespaces:
        from: All
```

> Port `53` = DNS default. Use for DNS services or any connectionless UDP-based application.

---

### gRPC — Microservice RPC

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grpc-route
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
  rules:
  - matches:
    - method:
        service: my.grpc.Service
        method: GetData
    backendRefs:
    - name: grpc-service
      port: 50051
```

| Field | Purpose |
|-------|---------|
| `method.service` | Full gRPC service name to match |
| `method.method` | Specific gRPC method to match |
| `backendRefs.port` | `50051` is the gRPC standard port |

> gRPC uses `HTTPRoute` — not a separate GRPCRoute. Match on service + method for precise routing.

---

## 13. CORS Configuration

### Side-by-Side — Ingress vs Gateway API

| Ingress (NGINX + Traefik) | Gateway API |
|---------------------------|-------------|
| <pre># NGINX<br>annotations:<br>  nginx.ingress.kubernetes.io/enable-cors: "true"<br>  nginx.ingress.kubernetes.io/cors-allow-methods: "GET,PUT,POST"<br>  nginx.ingress.kubernetes.io/cors-allow-origin: "https://allowed-origin.com"<br>  nginx.ingress.kubernetes.io/cors-allow-credentials: "true"<br><br># Traefik<br>annotations:<br>  traefik.ingress.kubernetes.io/headers.customresponseheaders: \|<br>    Access-Control-Allow-Origin: '*'<br>    Access-Control-Allow-Methods: GET,POST,PUT<br>    Access-Control-Allow-Headers: Content-Type<br>    Access-Control-Allow-Credentials: true<br>    Access-Control-Max-Age: 3600</pre> | <pre>filters:<br>- type: ResponseHeaderModifier<br>  responseHeaderModifier:<br>    add:<br>    - name: Access-Control-Allow-Origin<br>      value: "*"<br>    - name: Access-Control-Allow-Methods<br>      value: "GET,POST,PUT,DELETE,OPTIONS"<br>    - name: Access-Control-Allow-Headers<br>      value: "Content-Type,Authorization"<br>    - name: Access-Control-Allow-Credentials<br>      value: "true"<br>    - name: Access-Control-Max-Age<br>      value: "3600"</pre> |

**Full Gateway API CORS HTTPRoute:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cors-route
  namespace: default
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    - type: ResponseHeaderModifier
      responseHeaderModifier:
        add:
        - name: Access-Control-Allow-Origin
          value: "*"
        - name: Access-Control-Allow-Methods
          value: "GET,POST,PUT,DELETE,OPTIONS"
        - name: Access-Control-Allow-Headers
          value: "Content-Type,Authorization"
        - name: Access-Control-Allow-Credentials
          value: "true"
        - name: Access-Control-Max-Age
          value: "3600"
    backendRefs:
    - name: api-service
      port: 80
```

**Differences:** NGINX and Traefik use completely different annotation formats → Gateway API uses one unified `ResponseHeaderModifier` filter that works on any controller.

---

## 14. Real-World Example — Deploy for a Service

> Scenario: Gateway is already deployed in `nginx-gateway` namespace. Deploy routing for a `frontend-svc` service.

### Step 1 — Gateway (already deployed, shown for reference)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
```

### Step 2 — HTTPRoute (deploy this for your service)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
    sectionName: http
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-svc
      port: 80
```

| Field | Value | Why |
|-------|-------|-----|
| `parentRefs.namespace` | `nginx-gateway` | Gateway lives in a different namespace |
| `parentRefs.sectionName` | `http` | Targets the specific listener named `http` |
| `backendRefs.name` | `frontend-svc` | The Kubernetes Service to route traffic to |

> Apply with: `kubectl apply -f httproute.yaml`
> Verify with: `kubectl get httproute -n default`

---

## 15. Quick Reference

### Resource Ownership

| Resource | Owned By | Scope |
|----------|----------|-------|
| `GatewayClass` | Infrastructure Provider | Cluster-wide |
| `Gateway` | Cluster Operator / Platform Team | Namespace |
| `HTTPRoute` / `TCPRoute` / `UDPRoute` / `GRPCRoute` | Application Developer | Namespace |

### Feature Map

| Feature | Resource | Key Field |
|---------|----------|-----------|
| Controller binding | `GatewayClass` | `controllerName` |
| Port + protocol | `Gateway` | `listeners[].protocol` + `port` |
| TLS termination | `Gateway` | `tls.mode: Terminate` |
| Basic path routing | `HTTPRoute` | `matches.path` |
| HTTPS redirect | `HTTPRoute` | `filters.RequestRedirect` |
| Path rewrite | `HTTPRoute` | `filters.URLRewrite` |
| Header inject | `HTTPRoute` | `filters.RequestHeaderModifier` |
| CORS headers | `HTTPRoute` | `filters.ResponseHeaderModifier` |
| Traffic split / canary | `HTTPRoute` | `backendRefs[].weight` |
| Request mirroring | `HTTPRoute` | `filters.RequestMirror` |
| TCP routing | `Gateway` | `listeners[].protocol: TCP` |
| UDP routing | `Gateway` | `listeners[].protocol: UDP` |
| gRPC routing | `HTTPRoute` | `matches.method` |
| Cross-namespace routing | `HTTPRoute` | `ReferenceGrant` |
