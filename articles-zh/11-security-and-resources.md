# 11 · 保护工作负载：RBAC、ServiceAccount、SecurityContext 与资源限制

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[Securing Workloads](../articles/11-security-and-resources.md) · 上一篇：[10 · 安全发布](10-deployment-strategies.md) · 下一篇：[12 · 保持应用健康](12-observability.md)

每个发往 API server 的请求都要过三道关：**认证**（你是谁？）、**授权**（你能做这件事吗？）和**准入**（请求是否符合集群的规则？）。本文讲应用开发者要接触的那部分，再加上防止一个应用饿死其他应用的资源控制。

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| API | Application Programming Interface | 应用程序编程接口 |
| RBAC | Role-Based Access Control | 基于角色的访问控制 |
| OIDC | OpenID Connect | 开放身份连接协议 |
| CKA | Certified Kubernetes Administrator | Kubernetes 认证管理员 |
| CA | Certificate Authority | 证书颁发机构 |
| URL | Uniform Resource Locator | 统一资源定位符 |
| CI | Continuous Integration | 持续集成 |
| UID | User Identifier | 用户标识符 |
| CPU | Central Processing Unit | 中央处理器 |
| OOM | Out Of Memory | 内存耗尽 |

## 认证：你是谁？

身份有两类：

- **用户（User）**，指人。Kubernetes 里没有 User 对象，用户来自证书、OIDC（OpenID Connect，开放身份连接协议）等外部来源。在 Minikube 和 kubeadm 实验环境里，你用的是预先配好的本地管理员。创建用户属于 CKA（Certified Kubernetes Administrator，Kubernetes 认证管理员）的考查范围。
- **ServiceAccount**，指工作负载。它们*是* Kubernetes 对象，属于某个命名空间。

客户端身份保存在 `~/.kube/config` 里：

```bash
kubectl config view                 # clusters, users, contexts (secrets redacted)
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <name>
kubectl config set-context --current --namespace=dev   # change your default namespace
```

| kubeconfig 段落 | 内容 |
|--------------------|----------|
| `clusters` | API server 的 URL + CA（Certificate Authority，证书颁发机构）证书 |
| `users` | 客户端证书/私钥或令牌 |
| `contexts` | 集群 + 用户 + 默认命名空间 |

> 考试时每道题都会告诉你用哪个 context。**每一道题**都要先执行题目给出的 `kubectl config use-context` 命令。

## 授权：RBAC

RBAC（Role-Based Access Control，基于角色的访问控制）由四种对象组成：

| 对象 | 作用范围 | 表达的意思 |
|--------|-------|------|
| **Role** | 命名空间 | 「在这个命名空间里，可以对这些资源做这些动作」 |
| **ClusterRole** | 集群 | 同上，但作用于整个集群（或可在多个命名空间复用） |
| **RoleBinding** | 命名空间 | 「在这个命名空间里，把这个 (Cluster)Role 授予这个主体」 |
| **ClusterRoleBinding** | 集群 | 「在所有地方把这个 ClusterRole 授予这个主体」 |

主体（subject）可以是用户、组或 ServiceAccount。权限只会叠加，没有「拒绝」规则。

### 检查某个身份能做什么

```bash
kubectl auth can-i get pods
kubectl auth can-i create deployments -n prod
kubectl auth can-i '*' '*'                               # am I cluster-admin?
kubectl auth can-i get pods --as bob@example.com
kubectl auth can-i get pods -n bellevue \
  --as=system:serviceaccount:bellevue:viewer             # as a ServiceAccount
kubectl auth can-i --list -n bellevue --as=system:serviceaccount:bellevue:viewer
```

## ServiceAccount

每个 Pod 都以某个 ServiceAccount 的身份运行，不指定就用所在命名空间的 `default`。Kubernetes 会把一个短期令牌挂载到 `/var/run/secrets/kubernetes.io/serviceaccount/`，Pod 里的程序用它调用 API。

`default` 这个 ServiceAccount 几乎没有任何权限，大多数应用也不需要权限。只有当 Pod 需要和 API 打交道时，才给它建一个专用的 ServiceAccount，比如 operator、CI（Continuous Integration，持续集成）runner、监控代理或网络插件：

```bash
kubectl get sa -n kube-system
kubectl describe pod -n kube-system -l k8s-app=kube-dns | grep -i "service account"
kubectl describe clusterrolebinding system:coredns
kubectl describe clusterrole system:coredns       # what CoreDNS may read
```

### 动手：一个只读的 Pod 查看者

```bash
kubectl create ns bellevue
kubectl create role viewer --verb=get,list,watch --resource=pods -n bellevue
kubectl create sa viewer -n bellevue
kubectl create rolebinding viewer --role=viewer --serviceaccount=bellevue:viewer -n bellevue

kubectl create deploy viewginx --image=nginx --replicas=3 -n bellevue
kubectl set serviceaccount deploy viewginx viewer -n bellevue

kubectl auth can-i list pods   -n bellevue --as=system:serviceaccount:bellevue:viewer   # yes
kubectl auth can-i delete pods -n bellevue --as=system:serviceaccount:bellevue:viewer   # no
```

在 YAML 里，它只是 Pod spec 上的一个字段。这是考试第 15 题（[`task15.yaml`](../exercise/ckad/task15.yaml)）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: securepod
  namespace: oklahoma
spec:
  serviceAccountName: secure
  # automountServiceAccountToken: false   # if the app never calls the API
  containers:
  - name: securepod
    image: nginx:latest
```

运行中的 Pod 不能修改 `serviceAccountName`。要么删掉重建，要么设在 Deployment 上，让它滚动出新的 Pod。

## SecurityContext：限制容器能做什么

`securityContext` 控制 Linux 层面的权限。它可以设在 **Pod** 上（对所有容器生效），也可以设在每个**容器**上（覆盖 Pod 级的设置）。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:              # Pod level
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000               # group owner of mounted volumes
    runAsNonRoot: true
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox
    command: ["sh", "-c", "sleep 1h"]
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
    securityContext:            # container level
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```

```bash
kubectl exec security-context-demo -- id                # uid=1000 gid=3000 groups=2000
kubectl exec security-context-demo -- ls -ld /data/demo # group 2000
```

字段放在哪一级很重要，两级接受的字段并不一样：

| 只能放 Pod 级 | 只能放容器级 | 两级都可以 |
|----------------|----------------------|------|
| `fsGroup`、`supplementalGroups`、`sysctls` | `allowPrivilegeEscalation`、`capabilities`、`privileged`、`readOnlyRootFilesystem` | `runAsUser`、`runAsGroup`、`runAsNonRoot`、`seccompProfile`、`seLinuxOptions` |

考试第 13 题（[`task13-security.yaml`](../exercise/ckad/task13-security.yaml)）就同时用到了 Pod 级的 `supplementalGroups: [2000]` 和容器级的 `allowPrivilegeEscalation: false`。

安全设置导致 Pod 起不来时，会显示 `CreateContainerConfigError`、`Error` 或 `CrashLoopBackOff`。经典例子是设了 `runAsNonRoot: true`，镜像却以 root 运行。原版 `nginx` 用非 root 的 UID（User Identifier，用户标识符）运行也会失败，因为它要绑定 80 端口、要写 `/var/cache/nginx`。用 `kubectl describe pod` 和 `kubectl logs` 查原因。

## 资源请求、限制与配额

### requests 与 limits

```yaml
resources:
  requests:            # reserved: used by the scheduler to pick a node
    cpu: 250m          # 250 millicores = 0.25 CPU
    memory: 64Mi
  limits:              # ceiling: enforced on the node at runtime
    cpu: 500m
    memory: 128Mi
```

- **requests 决定调度**：Pod 只会被调度到*未预留*容量足够的节点上。没有节点放得下，Pod 就一直 **Pending**，事件里会出现 `FailedScheduling … Insufficient cpu/memory`。
- **limits 限制用量**：超过 **CPU**（Central Processing Unit，中央处理器）上限，容器会被**限流（throttle）**；超过**内存**上限，容器会被 **OOMKilled**（Out Of Memory，内存耗尽而被杀）并重启。
- CPU 以毫核为单位（`500m` 即半个核）；内存用 `Mi`/`Gi`（二进制）或 `M`/`G`（十进制）。

不改 YAML，直接给运行中的 Deployment 设置资源，这是考试第 11 题：

```bash
kubectl create ns nebraska
kubectl create deploy snowdeploy --image=nginx -n nebraska
kubectl set resources deploy snowdeploy -n nebraska \
  --requests=cpu=100m,memory=64Mi --limits=cpu=200m,memory=128Mi
```

查看实际用量，需要先装 metrics-server（`minikube addons enable metrics-server`）：

```bash
kubectl top pods -n nebraska
kubectl top nodes
```

### ResourceQuota 与 LimitRange（命名空间级）

**ResourceQuota** 限制一个命名空间的*总*用量：

```bash
kubectl create quota team-a -n team-a \
  --hard=requests.cpu=2,requests.memory=2Gi,limits.cpu=4,limits.memory=4Gi,pods=10
kubectl describe quota -n team-a          # used vs hard
```

一旦配额覆盖了 CPU/内存，**这个命名空间里的每个 Pod 都必须声明对应的 requests/limits**，否则会被拒绝。报错出现在 **ReplicaSet** 上，而不是 Deployment 上：

```bash
kubectl describe rs -n team-a | grep -A3 Events
# Error creating: pods "x" is forbidden: failed quota: team-a: must specify limits.cpu ...
```

**LimitRange** 会给没设置 requests/limits 的容器填上默认值（同时可以限定最小/最大值），和配额搭配使用效果很好。

## 要点回顾

- RBAC：Role/ClusterRole 说明*能做什么*，RoleBinding/ClusterRoleBinding 说明*谁能做*，用 `kubectl auth can-i --as` 验证。
- 只有当 Pod 要调用 API 时，才给它配专用的 ServiceAccount（`serviceAccountName`）。
- SecurityContext：以非 root 运行、去掉 capabilities、禁止权限提升；要清楚哪些字段放 Pod 级、哪些放容器级。
- requests 决定调度；limits 对 CPU 限流、对内存 OOM 杀进程；配额让命名空间里的资源声明变成强制要求。

下一篇：[用探针、日志和指标保持应用健康](12-observability.md)。
