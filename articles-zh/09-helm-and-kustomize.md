# 09 · 打包应用：Helm 与 Kustomize

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[Packaging Applications](../articles/09-helm-and-kustomize.md) · 上一篇：[08 · 把流量引进来](08-ingress-and-gateway-api.md) · 下一篇：[10 · 安全发布](10-deployment-strategies.md)

真实的应用从来不是一个 YAML 文件，而是一个 Deployment、一个 Service、一个 ConfigMap、一个 Secret、一个 Ingress，可能还有 PVC 和 RBAC 对象，而且每个环境的取值都略有不同。Kubernetes 世界里有两种常见答案：

| | Helm | Kustomize |
|-|------|-----------|
| 思路 | **模板** + 取值 → 清单 | 纯 YAML + **补丁/覆盖层（patch/overlay）** |
| 分发方式 | 放在仓库里的 chart（相当于 Kubernetes 的 apt/yum） | Git 里的目录 |
| 记录安装状态 | 有：release、历史、回滚 | 没有：只负责渲染和应用 |
| 是否内置于 kubectl | 否（单独的 `helm` 命令） | 是（`kubectl apply -k`） |
| 最适合 | 安装第三方软件；把自己的应用交付给别人 | 按环境调整自己的（或引入的第三方）YAML |

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| YAML | YAML Ain't Markup Language | 一种数据序列化格式 |
| PVC | PersistentVolumeClaim | 持久卷申领 |
| RBAC | Role-Based Access Control | 基于角色的访问控制 |
| CPU | Central Processing Unit | 中央处理器 |
| GitOps | Git-based Operations | 以 Git 为唯一事实来源的运维方式 |

## Helm

### 几个术语

- **Chart**：一个软件包，里面是带模板的清单，外加一份默认的 `values.yaml`。
- **Repository（仓库）**：存放 chart 的地方。Helm **没有默认仓库**，可以到 https://artifacthub.io 上找 chart。
- **Release**：chart 的一次安装实例，名字由你来起。同一个 chart 可以用不同的 release 名装很多次。

### 准备与搜索

```bash
source <(helm completion bash)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update                       # refresh the index, like apt update
helm repo list
helm search repo nginx                 # search repos you have added
helm search repo nginx --versions      # every available chart version
helm search hub wordpress              # search Artifact Hub
```

> Bitnami 在 2025 年调整了免费目录，很多镜像被挪走或受到限制。如果 Bitnami chart 的 Pod 卡在 `ImagePullBackOff`，去 Artifact Hub 看看该 chart 的说明，或者换一个 chart 练习，Helm 命令都一样。

### 安装、查看与卸载

```bash
helm install mysql1 bitnami/mysql                 # release name + chart
helm install bitnami/mysql --generate-name        # let Helm pick a name
helm install web bitnami/nginx -n web --create-namespace --version 15.0.0

helm list                     # releases in the current namespace
helm list -A                  # all namespaces
helm status mysql1
kubectl get all -l app.kubernetes.io/instance=mysql1

helm uninstall mysql1         # removes everything the release created
```

安装之前，先看清楚你要装的是什么：

```bash
helm show chart bitnami/mysql      # metadata
helm show readme bitnami/mysql
helm show values bitnami/nginx     # every configurable value (long!)
helm template web bitnami/nginx | less   # render manifests locally without installing
```

### 用 values 定制

chart 的模板从 `values.yaml` 读取设置。覆盖方式有两种：

```bash
# 1. A values file (preferred: reviewable, repeatable)
helm show values bitnami/nginx | grep -E 'replicaCount|commonLabels'
cat > values.yaml <<'EOF'
replicaCount: 3
commonLabels:
  type: helmapp
EOF
helm install mynginx bitnami/nginx --values values.yaml

# 2. Individual --set flags (handy for one-offs)
helm install mynginx bitnami/nginx --set replicaCount=3
```

```bash
helm get values mynginx          # only your overrides
helm get values mynginx --all    # the full merged set
helm get manifest mynginx        # the YAML that was applied
```

### 升级与回滚

```bash
helm install bitginx bitnami/nginx --set ingress.enabled=false
helm upgrade bitginx bitnami/nginx --set ingress.enabled=true --reuse-values
helm history bitginx              # revision list
helm rollback bitginx 1           # back to revision 1
helm upgrade --install app ./mychart -f values.yaml   # idempotent "install or upgrade"
```

> `helm upgrade` 时，你之前传过的值**不会**自动保留。要么再传一次 `-f values.yaml`，要么加 `--reuse-values`。忘了这一点，设置就会悄悄回到 chart 的默认值。

## Kustomize

Kustomize 从合法的纯 Kubernetes YAML 出发，按 `kustomization.yaml` 里描述的**变换（transformation）**处理它们。整个过程不涉及模板，所以基础文件单独拿出来也能用。这非常适合处理你控制不了的 YAML，比如每次发版都会变的上游清单。

### 一个基本的 kustomization

```text
myapp/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

```yaml
# kustomization.yaml
resources:            # which files to include
- deployment.yaml
- service.yaml
namePrefix: test-     # every object name gets this prefix
namespace: testing    # every object goes in this namespace
labels:               # added to every object (and to selectors)
- pairs:
    environment: testing
  includeSelectors: true
```

```bash
kubectl kustomize .          # render and print, don't apply
kubectl apply -k .           # render and apply
kubectl get all -n testing   # test-myapp deployment, test-myapp service ...
kubectl delete -k .          # remove everything it created
```

老一些的示例用的是 `commonLabels: {environment: testing}`，现在仍能用，但已被弃用，推荐改用 `labels`。

### base 与 overlay

Kustomize 真正的威力在于：一份 **base**，每个环境一个小小的 **overlay**：

```text
base/
  deployment.yaml
  service.yaml
  kustomization.yaml        # resources: [deployment.yaml, service.yaml]
overlays/
  dev/kustomization.yaml
  staging/kustomization.yaml
  prod/kustomization.yaml
```

```yaml
# overlays/prod/kustomization.yaml
resources:
- ../../base
namePrefix: prod-
namespace: production
labels:
- pairs:
    environment: production
replicas:
- name: myapp
  count: 5
images:
- name: nginx
  newTag: "1.27"
patches:
- path: resources-patch.yaml     # strategic-merge patch with prod CPU/memory limits
```

```bash
kubectl apply -k overlays/dev
kubectl apply -k overlays/prod
```

Kustomize 还能生成 ConfigMap 和 Secret（`configMapGenerator`、`secretGenerator`），并在名字后面加上内容哈希。这样配置一改，引用它的 Deployment 就会自动滚动更新。[第 10 篇](10-deployment-strategies.md)里的蓝绿部署演示，就是每种颜色一个 Kustomize 目录。

## 该用哪一个？

- **安装别人的软件**（Ingress 控制器、数据库、监控），用 **Helm**。
- **跨环境管理你自己的清单**，尤其是配合 Argo CD、Flux 这类 GitOps 工具时，用 **Kustomize**。
- 两者也能组合：先渲染 Helm chart，再用 Kustomize 做后处理。

## 要点回顾

- Helm 的流程：`repo add` → `repo update` → `show values` → `install -f values.yaml` → `list`/`status` → `upgrade`/`rollback`/`uninstall`。
- 每次 `helm upgrade` 都要重新传入你的 values。
- Kustomize：`resources` 加上各种变换器（`namePrefix`、`namespace`、`labels`、`images`、`patches`），用 `kubectl apply -k` 应用。
- 用 base 和 overlay 管理开发、预发、生产环境，不必复制 YAML。

下一篇：[用滚动更新、蓝绿和金丝雀安全发布](10-deployment-strategies.md)。
