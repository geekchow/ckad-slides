# 12 · 保持应用健康：探针、日志与指标

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[Keeping Applications Healthy](../articles/12-observability.md) · 上一篇：[11 · 保护工作负载](11-security-and-resources.md) · 下一篇：[Extending the API（英文）](../articles/13-extending-the-api.md)

容器的进程**退出**时，Kubernetes 会重启它。但很多故障并不会让进程退出：死锁的线程、一直返回 500 的 Web 服务器、还在加载 2 GB 缓存的应用。**探针（probe）**让你告诉 Kubernetes，对*你的*应用来说「健康」意味着什么；**日志、事件和指标**帮你查清楚到底出了什么问题。

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| HTTP | Hypertext Transfer Protocol | 超文本传输协议 |
| TCP | Transmission Control Protocol | 传输控制协议 |
| gRPC | gRPC Remote Procedure Calls | 一种远程过程调用框架 |
| API | Application Programming Interface | 应用程序编程接口 |
| PVC | PersistentVolumeClaim | 持久卷申领 |
| OOM | Out Of Memory | 内存耗尽 |
| CPU | Central Processing Unit | 中央处理器 |
| HPA | HorizontalPodAutoscaler | Pod 水平自动扩缩器 |

## 探针

### 三种探针

| 探针 | 回答的问题 | 失败时会怎样 |
|-------|--------------------|---------------|
| **startupProbe** | 「应用启动完了吗？」 | 容器被杀掉并重启；在它成功之前，其他探针都不会开始 |
| **livenessProbe** | 「应用还活着吗？」 | 容器被**重启** |
| **readinessProbe** | 「现在能接流量吗？」 | Pod 被**移出 Service 的端点**，但不会重启 |

关键区别在于：**liveness 失败会重启，readiness 失败只是停止接流量**。readiness 还控制着滚动更新：新 Pod 只有 Ready 之后才算可用，旧 Pod 也要等新 Pod Ready 后才会被移除。

### 四种检查方式

| 方式 | 判定为通过的条件 |
|-----------|-------------|
| `exec` | 命令以状态码 0 退出 |
| `httpGet` | HTTP（Hypertext Transfer Protocol，超文本传输协议）响应码在 200–399 之间 |
| `tcpSocket` | 能与该端口建立 TCP（Transmission Control Protocol，传输控制协议）连接 |
| `grpc` | gRPC 健康检查返回 `SERVING` |

### 时间参数

```yaml
initialDelaySeconds: 5    # wait before the first check
periodSeconds: 10         # how often to check
timeoutSeconds: 1         # how long one check may take
failureThreshold: 3       # failures in a row before acting
successThreshold: 1       # successes needed to be OK again (readiness only can be >1)
```

从开始失败到采取动作，大约需要 `failureThreshold × periodSeconds`。对启动慢的应用，用一个 `failureThreshold` 给得宽裕的 `startupProbe`（比如 `30 × 10s` = 5 分钟），而不是给 liveness 探针设一个超大的 `initialDelaySeconds`。

### 例子：exec 探针（killer.sh 第 6 题）

*「创建 Pod `pod6`（busybox:1.31.0），带一个执行 `cat /tmp/ready` 的就绪探针，初始延迟 5 秒，周期 10 秒。容器运行 `touch /tmp/ready && sleep 1d`。」*（[`q-06-pod.yaml`](../exercise/killer-sh/q-06-pod.yaml)）

```bash
kubectl run pod6 --image=busybox:1.31.0 --dry-run=client -o yaml \
  --command -- sh -c "touch /tmp/ready && sleep 1d" > pod6.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod6
spec:
  containers:
  - name: pod6
    image: busybox:1.31.0
    command: ["sh", "-c", "touch /tmp/ready && sleep 1d"]
    readinessProbe:
      exec:
        command: ["sh", "-c", "cat /tmp/ready"]
      initialDelaySeconds: 5
      periodSeconds: 10
```

```bash
kubectl apply -f pod6.yaml
kubectl get pod pod6 -w        # READY goes 0/1 -> 1/1 after about 5-15 seconds
```

> 仔细读题：要的是 **readiness** 还是 **liveness** 探针？两者的检查语法一样，作用却完全不同。

### 例子：Web 应用的 HTTP 探针

```yaml
containers:
- name: web
  image: nginx
  ports:
  - containerPort: 80
  readinessProbe:
    httpGet:
      path: /
      port: 80
    periodSeconds: 5
  livenessProbe:
    tcpSocket:
      port: 80
    initialDelaySeconds: 10
    periodSeconds: 10
```

### 例子：探测一个依赖（考试第 5 题）

第 5 题要求写一个探针，检查 Kubernetes API 的健康检查端点。先在节点上找到这个端点：

```bash
minikube ssh -- curl -k -s https://localhost:8443/readyz     # ok
```

在 Pod 内部，API server 的地址永远是 `https://kubernetes.default.svc`，不需要写死任何节点 IP：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probepod
  namespace: probes
spec:
  containers:
  - name: probepod
    image: busybox
    command: ["sh", "-c", "sleep infinity"]
    livenessProbe:
      exec:
        command: ["wget", "-q", "-O-", "--no-check-certificate", "https://kubernetes.default.svc/readyz"]
      initialDelaySeconds: 5
      periodSeconds: 5
```

```bash
kubectl describe pod probepod -n probes   # Events: "Liveness probe failed: ..." if it is wrong
```

> 生产环境里，用 liveness 探针去检查*依赖*要格外小心：数据库一挂，所有应用容器会同时重启，情况只会更糟。检查依赖通常更适合用 readiness 探针。

## 日志

容器应该把日志写到 **stdout/stderr**。容器运行时把它们存在节点上，`kubectl logs` 负责读取：

```bash
kubectl logs pod1
kubectl logs pod1 -c sidecar           # pick a container in a multi-container Pod
kubectl logs pod1 --all-containers
kubectl logs pod1 -f --tail=50         # follow
kubectl logs pod1 --previous           # the crashed instance before the restart
kubectl logs deploy/web                # one Pod of the Deployment
kubectl logs -l app=web --prefix       # all Pods matching a label
```

如果应用只往文件里写日志，就加一个 sidecar，用 `tail -F` 把文件内容输出到 stdout（见[第 4 篇](04-pods-and-deployments.md)）。

对于运行在 Kubernetes *之外*的容器（killer.sh 第 11 题），思路一样：`docker logs <name>` 或 `podman logs <name>`。

## 事件与 describe

看日志之前，先看**事件**。调度问题、拉镜像失败、探针失败、OOM（Out Of Memory，内存耗尽）被杀、配额拒绝，事件里都有说明：

```bash
kubectl describe pod pod1                       # Events section at the bottom
kubectl get events --sort-by=.lastTimestamp     # recent events in the namespace
kubectl get events -A --field-selector type=Warning
```

### 读懂 Pod 状态

| 状态 / 原因 | 常见成因 | 去哪里看 |
|-----------------|-------------|---------|
| `Pending` | 没有节点放得下（资源、污点、亲和性），或 PVC 没有绑定 | `describe pod` 的事件、`describe pvc` |
| `ImagePullBackOff` / `ErrImagePull` | 镜像名/标签写错，或私有仓库没配 `imagePullSecrets` | `describe pod` |
| `CreateContainerConfigError` | 缺少 ConfigMap/Secret 或其中的键，或 securityContext 冲突 | `describe pod` |
| `CrashLoopBackOff` | 进程反复退出，或 liveness 探针反复失败 | `logs --previous`、`describe` |
| `OOMKilled`（上一次状态） | 内存上限设得太低 | `describe pod`、`top pod` |
| `Running` 但 READY 为 `0/1` | 就绪探针失败 | `describe pod` 的事件 |

## 指标

**metrics-server** 从每个 kubelet 收集 CPU/内存用量：

```bash
minikube addons enable metrics-server
kubectl -n kube-system get deploy metrics-server
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl top pod snowdeploy -n nebraska --containers
```

它还为 HorizontalPodAutoscaler（HPA，Pod 水平自动扩缩器）提供数据：

```bash
kubectl autoscale deploy web --cpu-percent=70 --min=2 --max=10   # needs CPU requests set
kubectl get hpa
```

要看仪表盘、做告警、留历史数据，还需要更多组件：指标用 **Prometheus + Grafana**，日志用 **Loki/Elasticsearch**，链路追踪用 **Jaeger/OpenTelemetry**。它们通常用 Helm 或 operator 安装（[第 13 篇，英文](../articles/13-extending-the-api.md)）。

## 要点回顾

- liveness 失败会重启，readiness 失败会移出端点，startup 保护启动慢的应用。
- 探针可以用 `exec`、`httpGet`、`tcpSocket` 或 `grpc`；要调好 `initialDelaySeconds`、`periodSeconds` 和 `failureThreshold`。
- 排查顺序：`get` → `describe`（看事件）→ `logs`（加 `--previous`）→ `exec`。
- `kubectl top` 需要 metrics-server；HPA 需要先设置 CPU requests。

下一篇：[用 CRD 和 operator 扩展 Kubernetes（英文）](../articles/13-extending-the-api.md)。
