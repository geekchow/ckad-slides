# 13 · 扩展 Kubernetes：API、CRD 与 Operator

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[Extending Kubernetes](../articles/13-extending-the-api.md) · 上一篇：[12 · 保持应用健康](12-observability.md) · 下一篇：[14 · CKAD 实战手册](14-ckad-exam-playbook.md)

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| API | Application Programming Interface | 应用程序编程接口 |
| HTTP | Hypertext Transfer Protocol | 超文本传输协议 |
| REST | Representational State Transfer | 表述性状态转移 |
| URL | Uniform Resource Locator | 统一资源定位符 |
| JSON | JavaScript Object Notation | 一种数据交换格式 |
| TLS | Transport Layer Security | 传输层安全协议 |
| CRD | Custom Resource Definition | 自定义资源定义 |
| CKAD | Certified Kubernetes Application Developer | Kubernetes 认证应用开发者 |

## Kubernetes API

这个系列里你做过的每一件事（`kubectl run`、`apply`、`get`、`delete`），本质上都是对 **kube-apiserver** 发起的一次 HTTP（Hypertext Transfer Protocol，超文本传输协议）调用。这个 API 有几个特点：

- **RESTful**：资源都有自己的 URL（Uniform Resource Locator，统一资源定位符），动作对应 HTTP 方法（GET、POST、PUT/PATCH、DELETE）。REST 即 Representational State Transfer，表述性状态转移。
- **声明式**：你以 YAML/JSON（JavaScript Object Notation）提交期望状态，由控制器去调谐。
- **按组分版本**：`v1`（核心组：pods、services、configmaps）、`apps/v1`（deployments）、`networking.k8s.io/v1`（ingress、networkpolicy）等等。
- **有安全保障**：TLS（Transport Layer Security，传输层安全协议）加上认证和授权（[第 11 篇](11-security-and-resources.md)）。
- **可扩展**：你可以加入自己的资源类型。

```bash
kubectl version                       # client and server version
kubectl api-versions                  # every group/version
kubectl api-resources -o wide         # every kind, its group, and the verbs it supports
```

完整的参考文档在 `https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.XX/`，把 `XX` 换成你集群的次版本号即可。

### 直接和 API 对话

`kubectl proxy` 会用你 kubeconfig 里的凭证，在本地打开一条已经认证好的通道连到 API server。之后用任何 HTTP 客户端都能访问：

```bash
kubectl proxy --port=8001 &
kubectl run proxypod --image=nginx

curl http://localhost:8001/version
curl http://localhost:8001/api/v1/namespaces/default/pods                 # list Pods
curl http://localhost:8001/api/v1/namespaces/default/pods/proxypod        # one Pod
curl -X DELETE http://localhost:8001/api/v1/namespaces/default/pods/proxypod
curl http://localhost:8001/apis/apps/v1/namespaces/default/deployments    # named groups live under /apis
```

> `kubectl proxy` 不是 `kube-proxy`。前者是你本机上的客户端辅助工具，后者是实现 Service 的节点组件。

想看 kubectl 实际发出的请求，加 `-v=6`（显示 URL）或 `-v=8`（显示请求和响应体）：

```bash
kubectl get pods -v=6
```

### API 弃用

API 版本会从 `v1alpha1` 演进到 `v1beta1` 再到 `v1`，旧版本最终会被**移除**。被弃用的 beta API 在弃用后还会保留一定的版本数（至少 9 个月或 3 个版本），之后就彻底消失，使用它的清单会被拒绝。著名的例子是 `extensions/v1beta1` 下的 Deployment 和 Ingress。

看到弃用警告，就该更新清单了：

```bash
kubectl apply -f redis-deploy.yaml
# Warning: ... is deprecated in v1.xx+, unavailable in v1.yy+; use <group>/<version>
kubectl explain deployment --api-version=apps/v1
kubectl explain deploy --recursive | less      # check the field structure for the current version
```

把旧 Deployment 迁到 `apps/v1` 时，通常要改两处：改 `apiVersion`，再补上如今必填的 `spec.selector.matchLabels`。

## 扩展 API 的三种方式

| 方式 | 工作量 | 能得到什么 |
|--------|--------|--------------|
| **CustomResourceDefinition（CRD）** | 写一些 YAML | 新的资源类型，存在 etcd 里，由常规的 API server 提供服务 |
| **自定义控制器 / operator** | 用客户端库写代码（Go、Python……） | 行为：真正*作用于*这些资源的程序 |
| **API 聚合（aggregation）** | 编写并运行你自己的 API server | 完全掌控；主 API server 把某个路径代理到你的 server（metrics-server 就是这么做的） |

CRD 和控制器通常成对出现：CRD 定义*数据*，控制器提供*行为*。

### 创建 CRD

定义一个自定义资源分两步：先定义类型，再创建这种类型的对象。

**第一步：定义类型**（`crd-object.yaml`）：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.stable.example.com     # <plural>.<group>
spec:
  group: stable.example.com
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    kind: BackUp
    shortNames: [bks]
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              backupType:
                type: string
              image:
                type: string
              replicas:
                type: integer
```

```bash
kubectl create -f crd-object.yaml
kubectl api-resources | grep backup    # backups  bks  stable.example.com/v1  true  BackUp
kubectl explain backup.spec            # your schema is documented automatically
```

**第二步：创建实例**（`crd-backup.yaml`）：

```yaml
apiVersion: stable.example.com/v1
kind: BackUp
metadata:
  name: mybackup
spec:
  backupType: full          # field names must match the schema exactly
  image: linux-backup-image
  replicas: 5
```

```bash
kubectl create -f crd-backup.yaml
kubectl get backups            # or: kubectl get bks
kubectl get bks mybackup -o yaml
```

> 字段名区分大小写。schema 里没有的字段（比如把 `backupType` 写成 `backUpType`）会被**修剪（prune）**，也就是被悄悄丢掉；开启严格字段校验时则会被拒绝。创建之后用 `kubectl get -o yaml` 核对一下。

光有 CRD 只能**存储**数据。在有控制器监听 `BackUp` 对象并据此行动之前，什么备份都不会发生。只有确实有东西会用到，加自定义资源才有意义。

## 控制器与 operator

**控制器**就是一个循环：*观察*当前状态，*比较*它和期望状态，*行动*以消除差距，周而复始。kube-controller-manager 运行着内置的那些控制器（Deployment、ReplicaSet、Job、Endpoints……），你在本系列的每一篇里都见过它们干活。

**operator** 是专门服务某一个应用的控制器，通常带着自己的 CRD。它把人类运维人员的经验编码进去：「升级这个数据库，先做 A，再做 B，然后等 C」。有了 operator，你只需要声明一个 `PostgresCluster` 或 `Prometheus` 对象，StatefulSet、Service、备份和故障切换都由 operator 处理。

去哪里找 operator：

- **OperatorHub.io**，以及 Artifact Hub 上的 Helm chart；
- 例子：**Prometheus Operator**（监控）、**Tigera Operator**（Calico 网络）、**Jaeger**（链路追踪）、**cert-manager**（TLS 证书），以及各种数据库 operator（CloudNativePG、面向 Kafka 的 Strimzi）。

本系列其实已经用过基于 CRD 的扩展了：**Gateway API**（[第 8 篇](08-ingress-and-gateway-api.md)）就是一组 CRD 加一个控制器（NGINX Gateway Fabric）。

```bash
kubectl get crd                         # what has been added to this cluster
kubectl get crd | grep gateway
```

## 要点回顾

- kubectl 是一个 HTTP 客户端，`kubectl proxy` + `curl` 以及 `-v=6` 能让你看到底层的 REST API。
- API 版本会被弃用和移除，收到警告时要更新 `apiVersion`（以及必填字段）。
- CRD = 新的资源类型（数据）；控制器/operator = 作用于它的行为；聚合 = 你自己的 API server。
- operator 封装了运维经验，生态里很多工具都以 operator 的形式发布。

下一篇：[CKAD（Certified Kubernetes Application Developer）实战手册](14-ckad-exam-playbook.md)。
