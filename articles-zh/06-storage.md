# 06 · 持久化数据：卷、PersistentVolume 与 StorageClass

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[Persisting Data](../articles/06-storage.md) · 上一篇：[05 · 配置应用](05-configmaps-and-secrets.md) · 下一篇：[Services and Networking（英文）](../articles/07-services-and-networking.md)

容器的文件系统是临时的：容器一重启，写进去的东西就没了。Pod 又会在节点之间迁移，所以本地磁盘也靠不住。Kubernetes 分层解决这个问题，从「容器之间共享文件」一直到「不管我跑在哪里，都给我 10 GiB 持久存储」。

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| PV | PersistentVolume | 持久卷 |
| PVC | PersistentVolumeClaim | 持久卷申领 |
| RWO / ROX / RWX / RWOP | ReadWriteOnce / ReadOnlyMany / ReadWriteMany / ReadWriteOncePod | 卷的四种访问模式 |
| NFS | Network File System | 网络文件系统 |
| EBS | (Amazon) Elastic Block Store | 亚马逊弹性块存储 |
| GiB | Gibibyte | 吉字节（2³⁰ 字节） |

## 第一层：Pod 卷

卷在 `spec.volumes` 里声明，再用 `volumeMounts` 挂进容器。前面已经见过两种：

| 卷类型 | 生命周期 | 典型用途 |
|--------|----------|----------|
| `emptyDir` | 和 **Pod** 一样长 | 临时空间，容器之间共享文件（sidecar） |
| `configMap` / `secret` | 跟随对象 | 以文件形式提供配置和凭证（[第 5 篇](05-configmaps-and-secrets.md)） |
| `hostPath` | 和**节点**磁盘一样长 | 节点级代理、单节点实验环境 |
| `persistentVolumeClaim` | 独立于 Pod | 真正的应用数据 |

### 实验环境里的 hostPath

考试第 9 题用 Minikube 节点上的一个目录提供网页（[`task9-pv.yaml`](../exercise/ckad/task9-pv.yaml)）：

```bash
minikube ssh -- 'mkdir -p /home/docker/webapp && echo "welcome to store pod" > /home/docker/webapp/index.html'
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storepod
  labels:
    type: store
spec:
  volumes:
  - name: vol
    hostPath:
      path: /home/docker/webapp
  containers:
  - name: web
    image: nginx:latest
    volumeMounts:
    - name: vol
      mountPath: /usr/share/nginx/html
```

```bash
kubectl apply -f task9-pv.yaml
kubectl expose pod storepod --port=80 --type=NodePort
kubectl edit svc storepod            # set nodePort: 32032
minikube ssh -- curl -s localhost:32032
```

`hostPath` 把数据绑在了**某一个节点**上。单节点的 Minikube 没问题，真实集群里 Pod 一旦被调度到别的节点就失效了。它还有安全风险：Pod 能读到宿主机上的文件。只在学习时用。

## 第二层：PersistentVolume 与 PersistentVolumeClaim

Kubernetes 把存储拆成两个对象，让开发者和集群管理员不必了解彼此的细节：

```text
 Developer                      Cluster / admin
 ─────────                      ───────────────
 Pod ──uses──▶ PVC  ──binds──▶  PV ──points at──▶ real storage (NFS, EBS, Ceph, hostPath …)
              "I need 2Gi RWO"  "Here is 2Gi RWO"
```

- **PersistentVolume（PV，持久卷）**是一块存储，属于**集群级**对象，没有命名空间。
- **PersistentVolumeClaim（PVC，持久卷申领）**是对存储的请求，属于**命名空间级**对象，和 Pod 放在一起。
- Kubernetes 会把 PVC **绑定（bind）**到容量、访问模式、`storageClassName` 和卷模式都匹配的 PV 上。

图里的 NFS（Network File System，网络文件系统）、EBS（Amazon Elastic Block Store，亚马逊弹性块存储）、Ceph 都是 PV 背后可能的真实存储。

### 访问模式

| 模式 | 简写 | 含义 |
|------|------|------|
| `ReadWriteOnce` | RWO | 被单个**节点**以读写方式挂载 |
| `ReadOnlyMany` | ROX | 被多个节点以只读方式挂载 |
| `ReadWriteMany` | RWX | 被多个节点以读写方式挂载（NFS、CephFS……） |
| `ReadWriteOncePod` | RWOP | 被单个 **Pod** 以读写方式挂载 |

### 静态供给全流程（killer.sh 第 12 题）

1. **PV**（[`q-12-pv.yaml`](../exercise/killer-sh/q-12-pv.yaml)），没有命名空间：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: earth-project-earthflower-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /Volumes/data
```

2. **PVC**（[`q-12-pvc.yaml`](../exercise/killer-sh/q-12-pvc.yaml)），放在应用所在的命名空间：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: earth-project-earthflower-pvc
  namespace: earth
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

3. **Deployment** 挂载这个申领（[`q-12-dp.yaml`](../exercise/killer-sh/q-12-dp.yaml)）：

```yaml
    spec:
      containers:
      - name: httpd
        image: httpd:2.4.41-alpine
        volumeMounts:
        - name: data
          mountPath: /tmp/project-data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: earth-project-earthflower-pvc
```

```bash
kubectl get pv,pvc -n earth         # STATUS should be Bound on both
```

> **小心默认 StorageClass。** Minikube（以及大多数集群）都有一个*默认* StorageClass。没写 `storageClassName` 的 PVC 会被归到默认类，走**动态供给**，而不会绑定到你手工建的 PV。要绑定手工 PV，就在 PVC 上写 `storageClassName: ""`（或者让 PV 和 PVC 用同一个类名）。PVC 一直 `Pending` 时，`kubectl describe pvc` 会告诉你原因。

## 第三层：StorageClass 与动态供给

手工建 PV 没法规模化。**StorageClass** 描述一*类*存储，并指定一个**供给器（provisioner）**：只要有 PVC 申请这个类，供给器就自动建出一个 PV。

killer.sh 第 13 题（[`q-13-sc.yaml`](../exercise/killer-sh/q-13-sc.yaml)、[`q-13-pvc.yaml`](../exercise/killer-sh/q-13-pvc.yaml)）：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: moon-retain          # cluster-scoped: no namespace
provisioner: moon-retainer
reclaimPolicy: Retain        # default is Delete
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: moon-pvc-126
  namespace: moon
spec:
  storageClassName: moon-retain
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

`moon-retainer` 是个虚构的供给器，所以 PVC 会一直 `Pending`，`kubectl describe pvc` 显示 `waiting for a volume to be created … by external provisioner "moon-retainer"`——这正是这道题期望的结果。在 Minikube 里，内置的 `standard` 类（供给器 `k8s.io/minikube-hostpath`）才会真的建出卷。

```bash
kubectl get sc
kubectl get pvc -n moon
kubectl describe pvc moon-pvc-126 -n moon
```

### 回收策略：申领被删之后会怎样

| `reclaimPolicy` | 删除 PVC 时的效果 |
|-----------------|-------------------|
| `Delete` | PV **连同底层存储**一起删除（动态供给的默认值） |
| `Retain` | PV 变为 `Released`，数据保留，需要管理员手动清理或重新利用 |

凡是误删不起的数据，都用 `Retain`。

## 要点回顾

- `emptyDir` 和 Pod 同寿，`hostPath` 绑在单个节点上，真正的数据要用 PVC。
- PV 是集群级的存储，PVC 是命名空间级的请求，两者按容量、访问模式和存储类绑定。
- StorageClass 带来动态供给。要清楚默认类是哪个，以及 `storageClassName: ""` 如何改变绑定行为。
- 每一个 `Pending` 的 PVC，`kubectl describe pvc` 都能解释原因。

下一篇：[用 Service、DNS 和 NetworkPolicy 组网（英文）](../articles/07-services-and-networking.md)。
