# 8. Getting Traffic In: Ingress and Gateway API

> Series: [Working with Kubernetes](README.md) · Previous: [Services and Networking](07-services-and-networking.md) · Next: [Helm and Kustomize](09-helm-and-kustomize.md)

NodePort Services work, but users don't want to type `http://10.0.0.5:31742`. They want `https://shop.example.com/cart`. For that you need an **HTTP-aware reverse proxy** in front of your Services. Kubernetes offers two APIs for this:

- **Ingress**: the long-standing API. It is stable, simple and still in the CKAD curriculum. It is now in *feature freeze*.
- **Gateway API**: its successor. It is more expressive, role-oriented and actively developed.

> Pick one per lab cluster. Running an Ingress controller and a Gateway controller side by side on Minikube leads to port clashes and confusing results.

## Ingress

![Ingress network flow](../Ingress-network-flow.jpeg)

### Two parts: the resource and the controller

- An **Ingress resource** is just a set of routing rules: host + path → Service:port.
- An **Ingress controller** is a reverse proxy (NGINX, HAProxy, Traefik, Kong, Contour …) running in the cluster. It watches Ingress resources and configures itself to match.

**An Ingress without a controller does nothing.** Vanilla Kubernetes ships no controller. The CNCF doesn't want to pick a winner. Distributions usually include one.

Ingress can:

- give Services externally reachable URLs,
- load-balance traffic,
- terminate TLS,
- do name-based virtual hosting.

### Installing a controller

On Minikube, use the addon:

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

On vanilla Kubernetes, use Helm:

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

> The community `ingress-nginx` project has announced its retirement. For new production clusters, look at Gateway API (below) or another maintained controller. For learning Ingress, it is still the simplest option.

### Your first Ingress

```bash
kubectl create deploy nginxsvc --image=nginx --port=80
kubectl expose deploy nginxsvc
kubectl create ingress nginxsvc-ingress --class=nginx --rule="nginxsvc.info/*=nginxsvc:80"
echo "$(minikube ip) nginxsvc.info" | sudo tee -a /etc/hosts
curl nginxsvc.info
kubectl describe ing nginxsvc-ingress      # shows rules and backend endpoints
```

Name resolution is up to you. The host name must resolve to the controller's IP, either through DNS or `/etc/hosts`.

### Rules, paths and pathType

Each rule has:

- an optional **host**. Without one, the rule matches all hosts;
- a list of **paths**, each with a **backend** Service and port;
- a **pathType**:

| pathType | `/foo` matches | Notes |
|----------|----------------|-------|
| `Exact` | `/foo` only (not `/foo/`) | |
| `Prefix` | `/foo`, `/foo/`, `/foo/bar` (not `/foobar`) | Matches whole path elements |
| `ImplementationSpecific` | Up to the controller | |

In `kubectl create ingress --rule`, a path ending in `*` becomes `Prefix`. Without `*` it is `Exact`. This is a common reason why `curl host/` works but `curl host/index.html` returns 404.

### Three Ingress shapes

```bash
# 1. Single Service
kubectl create ingress single --rule="/files*=fileservice:80"

# 2. Simple fan-out: one host, several paths
kubectl create ingress fanout --rule="/files*=fileservice:80" --rule="/db*=dbservice:80"

# 3. Name-based virtual hosting: several hosts on one IP
kubectl create deploy mars --image=nginx
kubectl create deploy saturn --image=httpd
kubectl expose deploy mars --port=80
kubectl expose deploy saturn --port=80
kubectl create ingress multihost \
  --rule="mars.example.com/*=mars:80" \
  --rule="saturn.example.com/*=saturn:80"
# add both names to /etc/hosts pointing at $(minikube ip)
curl mars.example.com; curl saturn.example.com
```

The YAML for one rule:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multihost
spec:
  ingressClassName: nginx
  rules:
  - host: mars.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mars
            port:
              number: 80
  # tls:
  # - hosts: [mars.example.com]
  #   secretName: mars-tls      # a kubernetes.io/tls Secret
```

## Gateway API

![Gateway API network flow](../apigateway-network-flow.jpeg)

Ingress packs everything into one object and leaves the advanced features to controller-specific annotations. **Gateway API** splits the job into role-specific resources and builds the advanced features into the API:

- HTTP, HTTPS, **TCP, UDP, gRPC** and TLS routes,
- **traffic splitting** by weight, request mirroring, header matching, redirects and rewrites,
- WebSockets.

### The resources

| Resource | Owned by | Purpose |
|----------|----------|---------|
| **GatewayClass** | Infrastructure provider | Names a controller (`spec.controllerName`). Like an IngressClass. |
| **Gateway** | Cluster operator | An actual listener: which ports/protocols/hostnames, and which routes may attach |
| **HTTPRoute** (GRPCRoute, TCPRoute …) | Application developer | Matching rules → backend Services, with weights and filters |

The resources are **CRDs** (see [article 13](13-extending-the-api.md)), and you need a **Gateway controller** that implements them. Without a controller, nothing handles the traffic.

### Procedure

1. Install the Gateway API CRDs.
2. Install a Gateway controller (this course uses NGINX Gateway Fabric).
3. Check that the controller is running and that a GatewayClass exists.
4. Deploy your application and its Service.
5. Create a Gateway and an HTTPRoute.
6. Test through the controller's Service.

```bash
# 1. CRDs (check https://github.com/kubernetes-sigs/gateway-api/releases for the current version)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml

# 2. Controller (see the NGINX Gateway Fabric docs for the current chart location)
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace -n nginx-gateway

# 3. Verify
kubectl get pods,svc -n nginx-gateway
kubectl get gatewayclass              # note the name, e.g. "nginx"

# 4. Application
kubectl create deploy nginxgw --image=nginx --replicas=3
kubectl expose deploy nginxgw --port=80
```

```yaml
# 5. Gateway + HTTPRoute (http-routing.yaml)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cluster-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    hostname: "*.example.com"
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginxgw-route
spec:
  parentRefs:
  - name: cluster-gateway
  hostnames:
  - whatever.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: nginxgw
      port: 80
```

```bash
kubectl apply -f http-routing.yaml
kubectl get gateway,httproute
kubectl describe httproute nginxgw-route      # look for Accepted / ResolvedRefs conditions

# 6. Test via port-forward to the controller (or make its Service a NodePort)
echo "127.0.0.1 whatever.example.com" | sudo tee -a /etc/hosts
kubectl -n nginx-gateway port-forward svc/<gateway-service-name> 8080:80
curl whatever.example.com:8080
```

Recent NGINX Gateway Fabric versions create a separate data-plane Deployment and Service for each Gateway. `kubectl get svc -A` shows which Service to forward to.

### Weighted traffic splitting

This is the feature that makes canary releases ([article 10](10-deployment-strategies.md)) a one-liner:

```yaml
    backendRefs:
    - name: app-v1
      port: 80
      weight: 90
    - name: app-v2
      port: 80
      weight: 10
```

## Ingress vs Gateway API at a glance

| | Ingress | Gateway API |
|-|---------|-------------|
| Status | Stable, feature-frozen | Stable core (v1), actively evolving |
| Protocols | HTTP/HTTPS | HTTP, HTTPS, gRPC, TCP, UDP, TLS |
| Advanced routing | Controller-specific annotations | Built in (weights, headers, mirroring, rewrites) |
| Roles | One object | GatewayClass / Gateway / Route split by role |
| Exam | In the current CKAD curriculum | Expected to grow in importance |

## Key takeaways

- Ingress and Gateway API both need a **controller**. The resources alone do nothing.
- Ingress: host + path (`Exact`/`Prefix`) → Service. Use `kubectl create ingress --rule` to generate it quickly.
- Gateway API: GatewayClass → Gateway → HTTPRoute, with weights and more protocols built in.
- DNS or `/etc/hosts` must point the host name at the controller.

Next: [packaging applications with Helm and Kustomize](09-helm-and-kustomize.md).
