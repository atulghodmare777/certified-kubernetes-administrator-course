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

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
      - secure.example.com
      secretName: tls-secret

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: secure-gateway
spec:
  gatewayClassName: example-gc
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: tls-secret
    allowedRoutes:
      kinds:
      - kind: HTTPRoute

# Ingress vs Gateway API — YAML Side by Side

| # | Ingress | Gateway API |
|---|---------|-------------|
| 1  | `apiVersion: networking.k8s.io/v1`                          | `apiVersion: gateway.networking.k8s.io/v1`    |
| 2  | `kind: Ingress`                                             | `kind: Gateway`                               |
| 3  | `metadata:`                                                 | `metadata:`                                   |
| 4  | `  name: secure-app`                                        | `  name: secure-gateway`                      |
| 5  | `  annotations:`                                            | ❌ _(no annotations needed)_                  |
| 6  | `    nginx.ingress.kubernetes.io/ssl-redirect: "true"`      | ❌ _(no annotations needed)_                  |
| 7  | `    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"`| ❌ _(no annotations needed)_                  |
| 8  | `spec:`                                                     | `spec:`                                       |
| 9  | ❌ _(no controller class concept)_                          | `  gatewayClassName: example-gc`              |
| 10 | ❌ _(no listeners block)_                                   | `  listeners:`                                |
| 11 | ❌ _(no listener name)_                                     | `  - name: https`                             |
| 12 | ❌ _(no port defined)_                                      | `    port: 443`                               |
| 13 | ❌ _(no protocol — SSL forced via annotation)_              | `    protocol: HTTPS`                         |
| 14 | `  tls:`                                                    | `    tls:`                                    |
| 15 | `  - hosts:`                                                | `      mode: Terminate`                       |
| 16 | `      - secure.example.com`                                | `      certificateRefs:`                      |
| 17 | `    secretName: tls-secret`                                | `      - kind: Secret`                        |
| 18 | ❌ _(no cert kind concept)_                                 | `        name: tls-secret`                    |
| 19 | ❌ _(no route control)_                                     | `    allowedRoutes:`                          |
| 20 | ❌ _(no route control)_                                     | `      kinds:`                                |
| 21 | ❌ _(no route control)_                                     | `      - kind: HTTPRoute`                     |

---

## What the ❌ rows tell you

| Ingress gap | Impact | Gateway API fix |
|---|---|---|
| SSL redirect via annotation (rows 5–7) | Breaks if you switch from NGINX controller | `protocol: HTTPS` on listener — controller agnostic |
| No controller class (row 9) | Can't target a specific implementation | `gatewayClassName` ties config to the right controller |
| No listener block (rows 10–13) | Port and protocol buried or implicit | Explicit `listeners[]` with `port` + `protocol` per entry |
| No TLS mode (row 15) | Termination behaviour assumed, not declared | `mode: Terminate` makes intent explicit |
| No cert kind (row 18) | Only k8s Secrets supported | `kind: Secret` — extensible to external cert providers |
| No route control (rows 19–21) | Any route type can bind to any ingress | `allowedRoutes.kinds` restricts to only `HTTPRoute` |
