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
