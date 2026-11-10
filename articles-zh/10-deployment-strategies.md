# 10 · 安全发布：滚动更新、蓝绿与金丝雀

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[Releasing Safely](../articles/10-deployment-strategies.md) · 上一篇：[09 · 打包应用](09-helm-and-kustomize.md) · 下一篇：[Security and Resources（英文）](../articles/11-security-and-resources.md)

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| DevOps | Development and Operations | 开发运维一体化 |
| GitOps | Git-based Operations | 以 Git 为唯一事实来源的运维方式 |
| HTTP / HTTPS | Hypertext Transfer Protocol / HTTP Secure | 超文本传输协议 / 安全超文本传输协议 |
| IP | Internet Protocol | 网际协议（此处指 IP 地址） |
| API | Application Programming Interface | 应用程序编程接口 |
| YAML | YAML Ain't Markup Language | 一种数据序列化格式 |

## Kubernetes 上的 DevOps 与 GitOps

在 Kubernetes 上「做 DevOps」，主要指这几件事：

- **配置即代码**：一切都是放在版本控制里的 YAML。
- **可复现**：同样的清单，每次都构建出同样的环境。
- **持续可用**：用户永远看不到维护窗口。
- **零停机更新**。

**GitOps** 更进一步：Git 仓库是唯一的事实来源，集群里的一个 operator（Argo CD、Flux）不停地拉取它并自动应用变更。发版就是合并一个 pull request。

怎样用版本 2 替换版本 1，就是你的**部署策略**。Kubernetes 内置了一种，其余几种可以用 Service、标签和 Ingress 自己搭出来。

| 策略 | 新旧版本会同时运行吗？ | 流量切换方式 | 回滚 |
|----------|---------------------------|---------------|----------|
| Recreate | 不会（有停机） | 一次性全切 | 重新部署旧版本 |
| 滚动更新 | 短暂混跑 | 按 Pod 数量逐步切换 | `kubectl rollout undo` |
| 蓝绿 | 会，并排运行 | *由你决定*何时一次性全切 | 切回去 |
| 金丝雀 | 会 | 先切一小部分，再逐步加大 | 删掉金丝雀 |

## 滚动更新（内置）

这是 Deployment 的默认策略，[第 4 篇](04-pods-and-deployments.md)已经讲过。用 `maxSurge` 和 `maxUnavailable` 调节，然后观察进度：

```bash
kubectl set image deploy/updates nginx=nginx:1.27
kubectl rollout status deploy/updates
kubectl rollout undo deploy/updates
```

滚动更新有两个前提：v1 和 v2 能同时对外服务；有 **readinessProbe** 告诉 Kubernetes 新 Pod 什么时候可以接流量（[第 12 篇，英文](../articles/12-observability.md)）。没有就绪探针，容器一启动 Pod 就开始接流量。

## 蓝绿部署

让新版本（**绿**）和当前版本（**蓝**）并排运行。先私下测试绿色，再一步把**全部**流量切过去。蓝色保留着，随时可以秒回滚。

### 基于 Service 的蓝绿

由 Service 的选择器决定哪种颜色接流量：

```bash
kubectl create deploy blue-nginx --image=nginx:1.14 --replicas=3
kubectl expose deploy blue-nginx --port=80 --name=bgnginx

# build green from blue
kubectl get deploy blue-nginx -o yaml > green-nginx.yaml
#   - remove status, uid, resourceVersion, creationTimestamp ...
#   - change the image to nginx:1.27
#   - replace "blue" with "green" everywhere (name, labels, selector)
kubectl create -f green-nginx.yaml
kubectl get pods -l app=green-nginx         # test green directly (port-forward, temp Pod ...)

# switch: point the Service at green
kubectl patch svc bgnginx -p '{"spec":{"selector":{"app":"green-nginx"}}}'
# (the slides delete and re-expose the Service instead; patching keeps the same ClusterIP)

kubectl delete deploy blue-nginx            # once you're confident
```

Service 层面的切换发生在连接级别，切换时，连到蓝色的长连接可能会断开。

### 基于 Ingress 的蓝绿

改为切换 **Ingress 的后端**。对 HTTP 客户端来说，代理处理得更平滑。课程演示里每种颜色一个 Kustomize 目录：

```bash
cd ~/ckad/kustomize-bluegreen/blue && kubectl apply -k .
echo "$(minikube ip) myapp.local" | sudo tee -a /etc/hosts
curl http://myapp.local                         # blue

cd ../green && kubectl apply -k .               # green runs, no traffic yet
sed -i 's/blue-svc/green-svc/' myapp-ing.yaml
kubectl apply -f myapp-ing.yaml                 # switch
curl http://myapp.local                         # green
kubectl scale deploy blue-deploy --replicas=0   # keep blue's spec for rollback
```

Ingress 只处理 HTTP/HTTPS，其他协议还是用 Service 的做法。

## 金丝雀部署

先把**一小部分**真实用户导到新版本，观察有没有错误，再一步步加大比例。出了问题就删掉金丝雀，受影响的只是少数用户。

### 基于 Service 的金丝雀：靠副本比例

Service 会在**所有**匹配选择器的 Pod 之间平均分配负载。所以给新旧 Pod 打上一个**共同的标签**，Service 只按这个标签选择，副本数量就决定了分流比例。

这就是考试第 12 题（[`task12-oldbirds.yaml`](../exercise/ckad/task12-oldbirds.yaml)、[`task12-newbirds.yaml`](../exercise/ckad/task12-newbirds.yaml)）：

```yaml
# oldbirds (newbirds is identical apart from name/app label/image)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oldbirds
  namespace: birds
  labels:
    app: oldbirds
    type: allbirds
spec:
  replicas: 4
  selector:
    matchLabels:
      app: oldbirds
  template:
    metadata:
      labels:
        app: oldbirds
        type: allbirds        # <- shared label, on the Pod template
    spec:
      containers:
      - name: nginx
        image: nginx:1.17
```

```bash
kubectl create ns birds
kubectl apply -f task12-oldbirds.yaml -f task12-newbirds.yaml   # newbirds: replicas 1
kubectl expose deploy oldbirds -n birds --name=allbirds \
  --selector type=allbirds --port=80 --type=NodePort
kubectl describe svc allbirds -n birds      # 5 endpoints: 4 old + 1 new = 20% canary
```

要推进，就扩容 newbirds、缩容 oldbirds；要回滚，删掉 newbirds 即可。

> 共同标签必须写在 **Pod 模板**的标签里。Deployment 自身 `metadata` 上的标签不会传到 Pod 上，而 Service 选择的是 Pod。

> **动手试试：**[交互实验](../articles/interactive/k8s-visual-labs.html#services)里的 *canary 3:1* 场景，展示了共享标签的 Service 下各版本分到的流量比例。

这种做法的局限：分流精度取决于副本数（想要 1% 就得有 100 个 Pod），而且是按连接分配，不是按用户。

### 基于 Ingress 的金丝雀：靠权重

用 ingress-nginx 时，给**同一个主机名**再建一个标记为 canary 的 Ingress，它就能按百分比分到请求，与副本数无关：

```bash
kubectl create ing old-version --rule="theapp.info/*=old-version:80"
```

```yaml
# new-ing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: new-version
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  ingressClassName: nginx
  rules:
  - host: theapp.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: new-version
            port:
              number: 80
```

```bash
kubectl apply -f new-ing.yaml
for i in $(seq 20); do curl -s theapp.info; done | sort | uniq -c   # ~16 old, ~4 new
```

ingress-nginx 还能按请求头或 cookie 路由金丝雀流量（`canary-by-header`、`canary-by-cookie`），这样测试人员可以主动选择进入新版本。

### Gateway API 的金丝雀

在 Gateway API 里，权重是标准 API 的一部分，不需要任何注解：

```yaml
backendRefs:
- name: old-version
  port: 80
  weight: 80
- name: new-version
  port: 80
  weight: 20
```

想要全自动的金丝雀发布（调整权重、检查指标、晋级或中止），可以看看 **Argo Rollouts** 或 **Flagger**。

## 要点回顾

- 滚动更新是内置的，要配合就绪探针使用，并熟练掌握 `rollout undo`。
- 蓝绿：两套完整环境，一次切换（Service 选择器或 Ingress 后端），回滚瞬间完成。
- 金丝雀：共同标签 + 副本比例（Service），或者权重（Ingress 注解 / Gateway API 的 `weight`）。
- 共同标签要放在 **Pod 模板**上，因为 Service 选择的是 Pod。

下一篇：[保护工作负载、控制资源（英文）](../articles/11-security-and-resources.md)。
