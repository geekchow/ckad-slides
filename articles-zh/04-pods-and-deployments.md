# 04 · 运行工作负载：Pod、多容器 Pod 与 Deployment

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[Running Workloads](../articles/04-pods-and-deployments.md) · 上一篇：[03 · 搭建实验环境](03-lab-setup-and-kubectl.md) · 下一篇：[ConfigMaps and Secrets（英文）](../articles/05-configmaps-and-secrets.md)

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| IP | Internet Protocol | 网际协议（此处指 IP 地址） |
| YAML | YAML Ain't Markup Language | 一种数据序列化格式 |
| CKAD | Certified Kubernetes Application Developer | Kubernetes 认证应用开发者 |
| rs | ReplicaSet | 副本集（kubectl 短名） |

## Pod：最小的调度单位

**Pod** 是一个或多个容器的组合。它们总是被一起调度到同一个节点，并共享：

- **同一个网络命名空间**：同一个 IP 地址，彼此之间通过 `localhost` 就能访问；
- **卷（volume）**：Pod 里的任何卷，其中任何一个容器都可以挂载。

大多数 Pod 只跑一个主容器。只有当另一个容器和主容器紧密耦合时，才把它放进同一个 Pod（见下文「多容器模式」）。

### 创建 Pod

```bash
kubectl run web --image=nginx --port=80 --labels="app=web,tier=frontend"
kubectl get pods -o wide --show-labels
kubectl describe pod web
```

同一个 Pod 写成 YAML（用 `kubectl run web --image=nginx --dry-run=client -o yaml` 生成）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  containers:
  - name: web
    image: nginx
    ports:
    - containerPort: 80
```

### command 与 args

容器默认运行镜像里的 `ENTRYPOINT`/`CMD`，除非你覆盖它：

| Kubernetes 字段 | 覆盖 Dockerfile 中的 | `kubectl run` 写法 |
|------------------|----------------------|----------------------|
| `command` | `ENTRYPOINT` | `kubectl run p --image=busybox --command -- sleep 3600` |
| `args` | `CMD` | `kubectl run p --image=busybox -- sleep 3600` |

busybox 和 alpine 镜像没有命令时会立刻退出。测试用的 Pod 要靠 `sleep infinity` 或 `sleep 1d` 保持运行：

```yaml
command: ["sh", "-c", "touch /tmp/ready && sleep 1d"]
```

### 用完即删的测试 Pod

```bash
# interactive shell, deleted on exit
kubectl run tmp --image=busybox:1.31.0 --restart=Never --rm -it -- sh

# one-shot command, output printed, Pod deleted
kubectl run tmp --image=nginx:alpine --restart=Never --rm -i -- curl -s 10.244.0.84
```

`--restart=Never` 建出一个裸 Pod，不会反复重启；`--rm` 让命令结束后自动删除它。

### 快速删除 Pod

正常删除会等最多 30 秒做优雅退出。考试时跳过等待：

```bash
kubectl delete pod pod1 --force --grace-period=0
```

## 标签与选择器：对象之间靠它互相找到

标签是 `metadata.labels` 里的键值对，**选择器（selector）**用来按标签查询。Deployment、Service、NetworkPolicy 都是这样找到自己的 Pod 的，对象之间没有写死的引用。

```bash
kubectl get pods -A -l tier=control-plane              # filter by label
kubectl get pods -A --selector tier=control-plane      # same thing
kubectl get pods -l 'env in (prod,staging)'
kubectl label pod web env=prod                         # add
kubectl label pod web env=dev --overwrite              # change
kubectl label pod web env-                             # remove
```

考试第 2 题要求把所有控制平面 Pod 的名字存进一个文件：

```bash
kubectl get pods -A -l tier=control-plane -o name | sed 's/^.*\///' > /tmp/pods.txt
# or, without sed:
kubectl get pods -A -l tier=control-plane -o custom-columns=NAME:.metadata.name --no-headers > /tmp/pods.txt
```

## 多容器模式

### Sidecar 容器

**sidecar** 是给主容器打下手的容器，典型的活是转发日志、刷新缓存或者跑一个代理。从 Kubernetes 1.29 开始（1.33 稳定），原生写法是**带 `restartPolicy: Always` 的 init 容器**：它在应用容器之前启动，和它们一起运行，并在它们之后停止。

下面是 [`exercise/ckad/sidcar-origin.yaml`](../exercise/ckad/sidcar-origin.yaml) 里的模式，写成一个 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidepod
spec:
  containers:
  - name: myapp
    image: alpine:latest
    command: ['sh', '-c', 'while true; do echo "logging" >> /opt/logs.txt; sleep 1; done']
    volumeMounts:
    - name: data
      mountPath: /opt
  initContainers:
  - name: logshipper
    image: alpine:latest
    restartPolicy: Always          # <- makes it a sidecar
    command: ['sh', '-c', 'tail -F /opt/logs.txt']
    volumeMounts:
    - name: data
      mountPath: /opt
  volumes:
  - name: data
    emptyDir: {}
```

```bash
kubectl apply -f sidepod.yaml
kubectl logs sidepod -c logshipper -f     # the sidecar streams the app's log file
```

两个容器共享一个 `emptyDir` 卷：应用往里写，sidecar 从里读。考试第 4 题里的 sidecar 跑的是 `sleep 15`，因为它带着 `restartPolicy: Always`，Kubernetes 每 15 秒就把它重启一次，这会体现在 RESTARTS 列上。

sidecar 也可以写成 **`containers:` 里的第二项**，killer.sh 第 16 题就是这样：给一个已有的 Deployment 加一个 `logger-con` 容器（`busybox:1.31.0`），对共享的 `cleaner.log` 执行 `tail -f`，于是 `kubectl logs ... -c logger-con` 就能看到这个文件。两种写法都合法，init 容器的写法能保证启动和停止的先后顺序。

### Init 容器

普通的 init 容器（不设 `restartPolicy`）会在应用容器启动**之前运行到结束**。用它来做准备工作，比如生成配置文件、跑数据库 schema 迁移，或者等某个依赖就绪。

killer.sh 第 17 题：一个 nginx Deployment 对外提供的是一个空卷，要加一个 init 容器先写好 `index.html`：

```yaml
spec:
  template:
    spec:
      initContainers:
      - name: init-con
        image: busybox:1.31.0
        command: ['sh', '-c', 'echo "check this out!" > /tmp/web-content/index.html']
        volumeMounts:
        - name: web-content
          mountPath: /tmp/web-content
      containers:
      - name: nginx
        image: nginx:1.17.3-alpine
        volumeMounts:
        - name: web-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: web-content
        emptyDir: {}
```

用一个临时 Pod 验证结果：`kubectl run tmp --rm -i --restart=Never --image=nginx:alpine -- curl -s <pod-ip>`。

## Deployment：别直接跑裸 Pod

裸 Pod 所在的节点挂了，它不会被重新调度；被删了，也没人补。实际使用中，要把 Pod 包在 **Deployment** 里：

```text
Deployment ──manages──▶ ReplicaSet (one per Pod-template version) ──manages──▶ Pods
```

```bash
kubectl create deploy web --image=nginx:1.17 --replicas=3
kubectl get deploy,rs,pods -l app=web
kubectl delete pod <one-of-them>   # watch a replacement appear immediately
kubectl scale deploy web --replicas=5
```

Deployment spec 里的关键部分，来自考试第 6 题（[`task6-dp.yaml`](../exercise/ckad/task6-dp.yaml)）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: updates
  labels:
    type: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      type: prod          # must match the template labels below
  template:               # the Pod "cookie cutter"
    metadata:
      labels:
        type: prod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 50%        # up to 3 + 50% (rounded up) = 5 Pods during an update
      maxUnavailable: 50%  # at least 3 - 50% (rounded down) = 2 Pods stay available
```

> `spec.selector` 不可变，而且必须和 `spec.template.metadata.labels` 匹配，否则 API server 会直接拒绝这个 Deployment。

### 滚动更新与回滚

`spec.template` 里任何东西一改（镜像、环境变量、资源……），就会建出一个**新的 ReplicaSet**。随后 Deployment 控制器在 `maxSurge`/`maxUnavailable` 的限制内，把 Pod 从旧 ReplicaSet 逐步挪到新的上。

```bash
kubectl set image deploy/updates nginx=nginx:latest    # or: kubectl edit deploy updates
kubectl rollout status deploy/updates
kubectl get rs -l type=prod                            # two ReplicaSets now: old (0) and new (3)

kubectl rollout history deploy/updates                 # list revisions
kubectl rollout history deploy/updates --revision=2    # what changed in rev 2
kubectl rollout undo deploy/updates                    # back to the previous revision
kubectl rollout undo deploy/updates --to-revision=1
```

killer.sh 第 8 题是个典型场景：新版本坏了（镜像错误、Pod 起不来）。用 `rollout history` 找到它，再 `rollout undo` 回到最后一个好的版本。旧的 ReplicaSet 会被保留、缩到 0，所以回滚是瞬间完成的。

> **动手试试：**[交互实验](../articles/interactive/k8s-visual-labs.html#rollout)里的 *Rolling update* 标签页，可以按任意 `replicas`、`maxSurge`、`maxUnavailable` 一轮一轮地演示滚动更新。

另一种策略 `type: Recreate` 会先删掉所有旧 Pod，再启动新的。适用于两个版本绝不能同时运行的场景（比如数据库 schema 变更），代价是会有停机时间。

## 要点回顾

- Pod 是共享一个 IP 和卷的一组容器；覆盖 `command`/`args` 就能改变容器运行的内容。
- 标签和选择器把一切串起来，`-l`、`--show-labels` 和 `kubectl label` 必须熟练。
- sidecar 是带 `restartPolicy: Always` 的 init 容器；普通 init 容器在应用启动前跑一次。
- 用 Deployment，别用裸 Pod。改模板会触发滚动更新，`kubectl rollout undo` 可以撤回，这些都是 CKAD 考试的高频考点。

下一篇：[用 ConfigMap 和 Secret 配置应用（英文）](../articles/05-configmaps-and-secrets.md)。
