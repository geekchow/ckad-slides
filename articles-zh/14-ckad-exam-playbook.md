# 14 · CKAD 实战手册：速度、习惯与常见失误

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[The CKAD Playbook](../articles/14-ckad-exam-playbook.md) · 上一篇：[13 · 扩展 Kubernetes](13-extending-the-api.md)

CKAD（Certified Kubernetes Application Developer，Kubernetes 认证应用开发者）是一场纯动手的考试：大约 2 小时，15–20 道题，每道题都在真实集群上操作。懂 Kubernetes 只占一半，另一半是**做得快，并且会检查自己的结果**。本文汇总了 `exercise/ckad` 和 `exercise/killer-sh` 那些解答背后的习惯。

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| CKAD | Certified Kubernetes Application Developer | Kubernetes 认证应用开发者 |
| YAML | YAML Ain't Markup Language | 一种数据序列化格式 |
| PV / PVC | PersistentVolume / PersistentVolumeClaim | 持久卷 / 持久卷申领 |
| CNI | Container Network Interface | 容器网络接口 |
| CPU | Central Processing Unit | 中央处理器 |

## 做第一道题之前

```bash
alias k=kubectl                                   # usually already set up
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# vim: sane YAML indentation
cat >> ~/.vimrc <<'EOF'
set expandtab tabstop=2 shiftwidth=2
EOF
```

## 每道题的固定流程

1. **切换 context**：把题目里的 `kubectl config use-context …` 那一行复制粘贴执行，每道题都要做。
2. **记下命名空间**：每条命令都带上 `-n <ns>`，或者执行 `kubectl config set-context --current --namespace=<ns>`。
3. **生成，而不是手敲**：`k create … $do > qN.yaml`。
4. **补齐缺的片段**：用 `kubectl explain` 或文档搜索（关键词见下文）。
5. **校验**：`k apply -f qN.yaml --dry-run=server` 连 schema 错误也能查出来。
6. **应用，然后验证**：`get`、`describe`、`logs`，或者从临时 Pod 里 `curl` 一下。
7. **按要求保存文件**：很多题还要求把 YAML 或输出存到指定路径。

## 值得背下来的命令式命令

```bash
# Workloads
k run web --image=nginx --port=80 --labels=app=web $do
k run tmp --image=busybox --restart=Never --rm -it -- sh
k create deploy web --image=nginx --replicas=3 $do
k create job hello --image=busybox -- echo hi
k create cronjob hello --image=busybox --schedule="*/5 * * * *" -- date
k set image deploy/web nginx=nginx:1.27
k scale deploy web --replicas=5
k rollout history|undo|status|restart deploy/web

# Config
k create cm cfg --from-literal=K=V --from-file=index.html
k create secret generic s1 --from-literal=user=test
k set env deploy/web --from=configmap/cfg --prefix=APP_
k set resources deploy web --requests=cpu=100m,memory=64Mi --limits=memory=128Mi

# Networking
k expose deploy web --port=80 --target-port=8080 --type=NodePort
k create ingress web --rule="web.example.com/*=web:80"

# Security
k create sa app
k set serviceaccount deploy web app
k create role r --verb=get,list --resource=pods
k create rolebinding rb --role=r --serviceaccount=ns:app
k auth can-i list pods --as=system:serviceaccount:ns:app
k create quota q --hard=pods=10,requests.cpu=2
```

## 文档搜索关键词

考试允许查阅 kubernetes.io。下面这些搜索能直接找到可以复制的 YAML 片段：

| 你需要 | 搜索 |
|----------|--------|
| ConfigMap 作为卷 / 环境变量 | `configmap` → 「Configure a Pod to Use a ConfigMap」 |
| Secret 作为环境变量 | `secret` → 「Distribute Credentials Securely Using Secrets」 |
| 探针 | `liveness` → 「Configure Liveness, Readiness and Startup Probes」 |
| Sidecar / init 容器 | `sidecar`、`init containers` |
| PV / PVC / Pod | `persistent volume` → 「Configure a Pod to Use a PersistentVolume for Storage」 |
| StorageClass | `storageclass` |
| NetworkPolicy | `network policy` |
| SecurityContext | `security context` |
| 资源限制 | `assign memory resources` |
| 部署策略 | `maxSurge` |
| Ingress | `ingress` |

## 常见失误（以及怎么发现）

**1. 修改了 Pod 的不可变字段。**
`kubectl edit pod` → 「Forbidden: pod updates may not change fields other than …」（见 [`frequent-error.md`](../exercise/ckad/frequent-error.md)）。执行 `k replace --force -f /tmp/kubectl-edit-*.yaml`，或者删掉后重新 apply。

**2. 引用的名字写错。**
`secretKeyRef.name`、`configMap.name` 或 `claimName` 和真实对象对不上，Pod 就会卡在 `CreateContainerConfigError`/`ContainerCreating`。`k describe pod` 会指出缺的是哪个对象。

**3. 标签放错了地方。**
Service 和 NetworkPolicy 选择的是 **Pod** 的标签，Deployment metadata 上的标签不算数。用 `k get pods --show-labels` 和 `k get ep <svc>` 检查。

**4. 忘了命名空间。**
对象建出来了，却在 `default` 里，而判分程序去 `moon` 里找。执行 `k get <kind> -A | grep <name>` 查一下。

**5. 忽略了默认 StorageClass。**
PVC 绑定到了动态供给的卷，而不是你的 PV，或者一直 `Pending`。用 `storageClassName: ""`（或者题目给定的类名）。

**6. 把 readiness 和 liveness 弄混。**
重读题目。readiness 管流量，liveness 会重启容器。

**7. 在错误的位置测试。**
ClusterIP 只在集群内部有响应，要用临时 Pod 测：

```bash
k run tmp --image=nginx:alpine --restart=Never --rm -i -- curl -s -m 2 svc.ns:80
```

在 macOS 上用 Docker 驱动跑 Minikube 时，宿主机访问不到 NodePort，要先 `minikube ssh` 进节点再 curl。

**8. 实验环境里 NetworkPolicy「不生效」。**
是 CNI（Container Network Interface，容器网络接口）没有执行它。用 `--cni=calico` 启动 Minikube。

**9. YAML 缩进。**
`volumeMounts` 属于容器，`volumes` 属于 `spec`，`fsGroup` 属于 **Pod** 级的 `securityContext`。`k apply --dry-run=server` 能查出大部分这类问题，`k explain pod.spec.containers.volumeMounts` 能确认字段该放在哪里。

**10. 参数拼写错误。**
`--from-liberal`、`--iamge`、只写一个横线的 `-from`，真实的笔记里都出现过。用 Tab 补全，并且认真看报错，kubectl 通常会明确告诉你哪个参数它不认识。

## 练习对照表

| 练习 | 主题 | 对应文章 |
|----------|-------|---------|
| 第 1 题、killer 第 14 题 | Secret → 环境变量 | [5](05-configmaps-and-secrets.md) |
| 第 2 题 | 标签选择器、输出格式 | [4](04-pods-and-deployments.md) |
| 第 3 题、killer 第 15 题 | ConfigMap → 卷 | [5](05-configmaps-and-secrets.md) |
| 第 4 题、killer 第 16/17 题 | Sidecar 与 init 容器 | [4](04-pods-and-deployments.md) |
| 第 5 题、killer 第 6 题 | 探针 | [12](12-observability.md) |
| 第 6 题、killer 第 8 题 | 滚动更新与回滚 | [4](04-pods-and-deployments.md)、[10](10-deployment-strategies.md) |
| 第 7 题、killer 第 18 题 | Service 与故障排查 | [7](07-services-and-networking.md) |
| 第 8 题 | NetworkPolicy | [7](07-services-and-networking.md) |
| 第 9 题、killer 第 12/13 题 | 卷、PV/PVC、StorageClass | [6](06-storage.md) |
| 第 10 题 | Helm | [9](09-helm-and-kustomize.md) |
| 第 11 题 | requests/limits、`kubectl top` | [11](11-security-and-resources.md)、[12](12-observability.md) |
| 第 12 题 | 金丝雀部署 | [10](10-deployment-strategies.md) |
| 第 13 题 | SecurityContext | [11](11-security-and-resources.md) |
| 第 14 题、killer 第 11 题 | 构建/保存镜像 | [1](01-containers-and-images.md) |
| 第 15 题 | ServiceAccount | [11](11-security-and-resources.md) |

## 最后的建议

- **先跳过，回头再做。** 一道只占 2% 的题耗掉 15 分钟，很不划算。标记一下，继续往下做。
- **每道题都验证。** 「apply 了」不等于「能用了」。花 10 秒 `get`/`curl` 检查一下，能挽回大部分丢分。
- **限时练习。** 重建实验环境（`minikube delete && minikube start --cni=calico`），把 `exercise/` 里的题反复做，直到每道题只需几分钟。

祝你好运，顺利上线。
