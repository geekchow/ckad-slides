# 03 · 搭建实验环境，用 kubectl 驾驭集群

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[Setting Up a Lab and Driving It with kubectl](../articles/03-lab-setup-and-kubectl.md) · 上一篇：[02 · Kubernetes 是怎么工作的](02-kubernetes-architecture.md) · 下一篇：[04 · 运行工作负载](04-pods-and-deployments.md)

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| VM | Virtual Machine | 虚拟机 |
| WSL2 | Windows Subsystem for Linux 2 | Windows 的 Linux 子系统（第 2 版） |
| CNI | Container Network Interface | 容器网络接口 |
| EKS / AKS / GKE | Amazon Elastic Kubernetes Service / Azure Kubernetes Service / Google Kubernetes Engine | 三大公有云的托管 Kubernetes 服务 |
| IP | Internet Protocol | 网际协议（此处指 IP 地址） |
| YAML | YAML Ain't Markup Language | 一种数据序列化格式 |

## 选一个实验环境

学习用的集群，要的是便宜、重建快、尽量接近原版 Kubernetes。云上的托管集群（EKS、AKS、GKE）在这里并不合适：Ingress、网络策略、存储这些恰恰需要学的部分，它们要么藏起来了，要么换成了云厂商自己的实现。

推荐两种做法：

1. **在 Ubuntu 虚拟机里跑 Minikube**：VirtualBox、VMware，或任意云上的一台 Ubuntu VM（Virtual Machine，虚拟机）都行。
2. **在 Windows 的 WSL2（Windows Subsystem for Linux 2）里跑 Minikube**，配合 Docker Desktop。

### 在 Ubuntu 虚拟机里跑 Minikube

```bash
sudo apt install git
git clone https://github.com/sandervanvugt/ckad
cd ckad
./minikube-docker-setup.sh          # installs docker, kubectl and minikube
minikube start --cni=calico
kubectl get all
```

### 在 WSL2 里跑 Minikube

```powershell
wsl -l -v          # make sure your Ubuntu distro runs WSL version 2
```

然后进入 Ubuntu，跑同一个安装脚本，用 Docker 驱动启动 Minikube：

```bash
minikube start --driver=docker --cni=calico
```

### 为什么要加 `--cni=calico`？

Minikube 默认的网络配置**不会**执行 NetworkPolicy，Calico 会。不加这个参数，你写的 NetworkPolicy 练习会悄无声息地「生效」——实际上什么都放行（见[第 7 篇，英文](../articles/07-services-and-networking.md)）。CNI 是 Container Network Interface（容器网络接口）的缩写，Calico 就是一种 CNI 网络插件。

### 常用的 Minikube 命令

```bash
minikube status
minikube ip                          # node IP, for NodePort / Ingress tests
minikube ssh                         # shell on the node (curl ClusterIPs from here)
minikube addons list
minikube addons enable ingress
minikube addons enable metrics-server
minikube delete && minikube start --cni=calico   # fresh cluster in a minute
```

> **macOS / Docker 驱动注意：** Minikube 节点运行在一个 Mac 访问不到的 Docker 网络里，所以 `curl $(minikube ip):30080` 会卡住不动。改用 `minikube ssh` 进到节点里再 curl，或者用 `minikube service <name> --url`、`kubectl port-forward`。

## kubectl：你唯一需要的工具

`kubectl` 是 API server 的命令行客户端，连接信息从 `~/.kube/config` 读取（见[第 11 篇，英文](../articles/11-security-and-resources.md)）。

### 先把 shell 配好

```bash
source <(kubectl completion bash)       # tab completion (add to ~/.bashrc)
alias k=kubectl
complete -o default -F __start_kubectl k  # completion for the alias too
export do="--dry-run=client -o yaml"    # k run x --image=nginx $do
```

### 不离开终端就能查帮助

kubectl 没有 man 手册，`-h` 就是手册，而且大多数命令**一开头就给出示例**：

```bash
kubectl create -h
kubectl create deployment -h
kubectl expose -h | less
kubectl get pods -h | grep -A5 label    # find the flag you need quickly
```

### 日常最常用的几个动作

| 目的 | 命令 |
|------|------|
| 从零创建 | `kubectl create deploy web --image=nginx` / `kubectl run pod1 --image=nginx` |
| 按文件创建或更新 | `kubectl apply -f file.yaml` |
| 查看 | `kubectl get pods -o wide`、`kubectl get all -n ns`，加 `-A` 看所有命名空间 |
| 看详情和**事件（Events）** | `kubectl describe pod pod1` |
| 看日志 | `kubectl logs pod1 [-c container] [-f] [--previous]` |
| 进入容器的 shell | `kubectl exec -it pod1 -- sh` |
| 在线修改 | `kubectl edit deploy web`、`kubectl set image ...`、`kubectl scale ...` |
| 删除 | `kubectl delete -f file.yaml`、`kubectl delete pod pod1` |

第一个应用：

```bash
kubectl create deploy firstapp --image=nginx
kubectl get deploy,rs,pods
```

一条命令建出了三个对象：Deployment 建出 ReplicaSet，ReplicaSet 又建出 Pod。这条链正是[第 2 篇](02-kubernetes-architecture.md)讲的调谐循环在起作用。

## 从命令式到声明式的工作流

凭记忆手写 YAML 既慢又容易错。快的做法是：

1. **生成**：用命令式命令加 `--dry-run=client -o yaml` 生成骨架。
2. **编辑**：补上生成器生成不了的部分（卷、探针、从 Secret 读环境变量……），片段可以从 kubernetes.io 复制。
3. **校验**：用 dry run 检查一遍。
4. **应用**。

```bash
kubectl run inpod --image=nginx -n indiana --dry-run=client -o yaml > inpod.yaml
vim inpod.yaml
kubectl apply -f inpod.yaml --dry-run=client     # syntax check
kubectl apply -f inpod.yaml
```

也可以把一个正在运行的对象导出成 YAML，但复用之前要清理干净：删掉 `status`、`metadata.uid`、`resourceVersion`、`creationTimestamp` 和 `managedFields`。

```bash
kubectl get deploy mydb -o yaml > mydb.yaml
```

## 有些字段改不了

Pod 的很多字段是**不可变的（immutable）**。比如你想用 `kubectl edit` 改一个 Pod 的 `command`，会得到：

```text
The Pod "nevatest" is invalid: spec: Forbidden: pod updates may not change fields other than
`spec.containers[*].image`, `spec.initContainers[*].image`, `spec.activeDeadlineSeconds`,
`spec.tolerations` (only additions ...), `spec.terminationGracePeriodSeconds` ...
```

解决办法是删掉重建。修改被拒绝时，`kubectl edit` 会把你改过的那份存到 `/tmp/kubectl-edit-xxxx.yaml`，直接拿来用：

```bash
kubectl replace --force -f /tmp/kubectl-edit-xxxx.yaml
```

Deployment 没有这个问题：改 Pod 模板只会触发一次滚动更新，换上新的 Pod。

## 别指望 Dashboard

很多发行版自带一个网页版 Dashboard。考试时别依赖它：它不一定装了，而且能做的事远不如 kubectl。

## 要点回顾

- 本系列以加了 `--cni=calico` 的 Minikube 作为标准实验环境。
- 动手之前先配好命令补全、`k` 别名和 `$do` 这个 dry-run 变量。
- `-h`、`kubectl explain` 和 `kubectl describe`（重点看 **Events**）能回答大部分问题。
- 用 `--dry-run=client -o yaml` 生成 YAML（YAML Ain't Markup Language），改好再 apply。

下一篇：[用 Pod 和 Deployment 运行工作负载](04-pods-and-deployments.md)。
