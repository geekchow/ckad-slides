# 08 · 把流量引进来：Ingress 与 Gateway API

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[Getting Traffic In](../articles/08-ingress-and-gateway-api.md) · 上一篇：[07 · 网络](07-services-and-networking.md) · 下一篇：[Helm and Kustomize（英文）](../articles/09-helm-and-kustomize.md)

NodePort Service 能用，但用户不想输入 `http://10.0.0.5:31742`，他们要的是 `https://shop.example.com/cart`。这就需要在 Service 前面放一个**懂 HTTP 的反向代理**。Kubernetes 为此提供了两套 API：

- **Ingress**：老牌 API，稳定、简单，仍在 CKAD 考纲内，目前处于*功能冻结*状态。
- **Gateway API**：它的继任者，表达力更强、按角色划分，还在积极开发。

> 一个实验集群只选一种。在 Minikube 上同时跑 Ingress 控制器和 Gateway 控制器，会造成端口冲突，结果也让人摸不着头脑。

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| CKAD | Certified Kubernetes Application Developer | Kubernetes 认证应用开发者 |
| API | Application Programming Interface | 应用程序编程接口 |
| HTTP / HTTPS | Hypertext Transfer Protocol / HTTP Secure | 超文本传输协议 / 安全超文本传输协议 |
| URL | Uniform Resource Locator | 统一资源定位符 |
| TLS | Transport Layer Security | 传输层安全协议 |
| CNCF | Cloud Native Computing Foundation | 云原生计算基金会 |
| CRD | Custom Resource Definition | 自定义资源定义 |
| gRPC | gRPC Remote Procedure Calls | 一种远程过程调用框架 |
| TCP / UDP | Transmission Control Protocol / User Datagram Protocol | 传输控制协议 / 用户数据报协议 |
| DNS | Domain Name System | 域名系统 |

## Ingress

![Ingress 网络流向](https://raw.githubusercontent.com/geekchow/ckad-slides/main/Ingress-network-flow.jpeg)

### 两部分：资源和控制器

- **Ingress 资源**只是一组路由规则：主机名 + 路径 → Service:端口。
- **Ingress 控制器**是运行在集群里的反向代理（NGINX、HAProxy、Traefik、Kong、Contour……），它监听 Ingress 资源，并按规则配置自己。

**没有控制器，Ingress 什么也不做。** 原版 Kubernetes 不带控制器，CNCF（Cloud Native Computing Foundation，云原生计算基金会）也不想替大家选定赢家，发行版通常会自带一个。

Ingress 可以：

- 给 Service 提供外部可访问的 URL（Uniform Resource Locator，统一资源定位符）；
- 对流量做负载均衡；
- 终止 TLS（Transport Layer Security，传输层安全协议）；
- 做基于域名的虚拟主机。

### 安装控制器

在 Minikube 上用插件：

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

在原版 Kubernetes 上用 Helm：

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

> 社区版 `ingress-nginx` 项目已宣布退役。新的生产集群建议看 Gateway API（见下文）或其他仍在维护的控制器；用来学习 Ingress，它依然是最简单的选择。

### 第一个 Ingress

```bash
kubectl create deploy nginxsvc --image=nginx --port=80
kubectl expose deploy nginxsvc
kubectl create ingress nginxsvc-ingress --class=nginx --rule="nginxsvc.info/*=nginxsvc:80"
echo "$(minikube ip) nginxsvc.info" | sudo tee -a /etc/hosts
curl nginxsvc.info
kubectl describe ing nginxsvc-ingress      # shows rules and backend endpoints
```

域名解析要自己解决：主机名必须解析到控制器的 IP，靠 DNS（Domain Name System，域名系统）或 `/etc/hosts` 都行。

### 规则、路径与 pathType

每条规则包含：

- 可选的 **host**，不写就匹配所有主机；
- 一组 **paths**，每条路径对应一个**后端** Service 和端口；
- 一个 **pathType**：

| pathType | `/foo` 能匹配 | 说明 |
|----------|----------------|-------|
| `Exact` | 只有 `/foo`（`/foo/` 不行） | |
| `Prefix` | `/foo`、`/foo/`、`/foo/bar`（`/foobar` 不行） | 按完整的路径段匹配 |
| `ImplementationSpecific` | 由控制器决定 | |

在 `kubectl create ingress --rule` 里，路径以 `*` 结尾就是 `Prefix`，没有 `*` 就是 `Exact`。`curl host/` 能通而 `curl host/index.html` 返回 404，常常就是这个原因。

### Ingress 的三种形态

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

一条规则的 YAML：

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

![Gateway API 网络流向](https://raw.githubusercontent.com/geekchow/ckad-slides/main/apigateway-network-flow.jpeg)

Ingress 把一切塞进一个对象，高级功能留给各控制器自己的注解（annotation）。**Gateway API** 则把职责拆成按角色划分的资源，并把高级功能直接做进 API：

- HTTP、HTTPS、**TCP、UDP、gRPC** 和 TLS 路由；
- 按权重**拆分流量**、请求镜像、请求头匹配、重定向与重写；
- WebSocket。

### 涉及的资源

| 资源 | 由谁负责 | 作用 |
|----------|----------|---------|
| **GatewayClass** | 基础设施提供方 | 指定控制器（`spec.controllerName`），类似 IngressClass |
| **Gateway** | 集群运维 | 实际的监听器：哪些端口/协议/主机名，允许哪些路由挂载 |
| **HTTPRoute**（GRPCRoute、TCPRoute……） | 应用开发者 | 匹配规则 → 后端 Service，可带权重和过滤器 |

这些资源都是 **CRD**（Custom Resource Definition，自定义资源定义，见[第 13 篇，英文](../articles/13-extending-the-api.md)），还需要一个实现它们的 **Gateway 控制器**。没有控制器，就没人处理流量。

### 操作步骤

1. 安装 Gateway API 的 CRD。
2. 安装 Gateway 控制器（这门课用的是 NGINX Gateway Fabric）。
3. 确认控制器在运行，并且存在 GatewayClass。
4. 部署应用及其 Service。
5. 创建 Gateway 和 HTTPRoute。
6. 通过控制器的 Service 测试。

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

较新版本的 NGINX Gateway Fabric 会为每个 Gateway 单独创建一套数据面的 Deployment 和 Service，用 `kubectl get svc -A` 就能看到该转发到哪个 Service。

### 按权重拆分流量

有了这个能力，金丝雀发布（[第 10 篇，英文](../articles/10-deployment-strategies.md)）只需要改几行配置：

```yaml
    backendRefs:
    - name: app-v1
      port: 80
      weight: 90
    - name: app-v2
      port: 80
      weight: 10
```

## Ingress 与 Gateway API 速览

| | Ingress | Gateway API |
|-|---------|-------------|
| 状态 | 稳定，功能冻结 | 核心已稳定（v1），持续演进 |
| 协议 | HTTP/HTTPS | HTTP、HTTPS、gRPC、TCP、UDP、TLS |
| 高级路由 | 各控制器自己的注解 | 内置（权重、请求头、镜像、重写） |
| 角色划分 | 一个对象 | GatewayClass / Gateway / Route 按角色拆分 |
| 考试 | 在当前 CKAD（Certified Kubernetes Application Developer）考纲内 | 重要性预计会上升 |

## 要点回顾

- Ingress 和 Gateway API 都需要**控制器**，光有资源什么也不会发生。
- Ingress：主机名 + 路径（`Exact`/`Prefix`）→ Service，用 `kubectl create ingress --rule` 能快速生成。
- Gateway API：GatewayClass → Gateway → HTTPRoute，内置权重和更多协议。
- DNS 或 `/etc/hosts` 必须把主机名指向控制器。

下一篇：[用 Helm 和 Kustomize 打包应用（英文）](../articles/09-helm-and-kustomize.md)。
