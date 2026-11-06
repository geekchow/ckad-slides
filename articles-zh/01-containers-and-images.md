# 01 · 容器与镜像：Kubernetes 的基本构件

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[Containers and Images](../articles/01-containers-and-images.md) · 下一篇：[02 · Kubernetes 是怎么工作的](02-kubernetes-architecture.md)

Kubernetes 并不直接运行你的代码，它运行的是**容器**，而容器从**镜像**启动。所以在写第一个 Kubernetes 清单之前，先要把三件事想清楚：镜像是什么、存在哪里、由谁来启动。

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| CKAD | Certified Kubernetes Application Developer | Kubernetes 认证应用开发者 |
| GHCR | GitHub Container Registry | GitHub 容器镜像仓库 |
| UBI | Universal Base Image | （Red Hat）通用基础镜像 |
| CI | Continuous Integration | 持续集成 |
| CLI | Command-Line Interface | 命令行界面 |

## 分层看：镜像仓库、运行时、引擎、编排器

| 组件 | 职责 | 例子 |
|------|------|------|
| **镜像仓库（registry）** | 存放镜像，谁来拉就给谁 | Docker Hub、Quay、GHCR（GitHub Container Registry，GitHub 容器镜像仓库）、私有仓库 |
| **容器运行时（container runtime）** | 真正把容器进程启动起来的底层程序 | `runc`、`crun` |
| **容器引擎（container engine）** | 架在运行时之上、面向单台机器的易用工具 | Docker、Podman |
| **编排器（orchestrator）** | 在多台机器上运行容器，并让它们一直保持运行 | Kubernetes |

在笔记本上，你用 Docker 或 Podman 拉镜像、起容器。到了集群里，这件事由 Kubernetes 在每个节点上替你做，而且做得更多：容器挂了会重启，会把容器分散到不同节点，还会把它们接入网络。这就是为什么它叫容器**编排器**，而不只是一个引擎。

公共仓库把镜像分享给所有人，私有仓库只给本公司用。仓库需要登录时，Kubernetes 用一个 `docker-registry` 类型的 Secret 来保存凭证（见[第 5 篇，英文](../articles/05-configmaps-and-secrets.md)）。

## 镜像是怎么构建的：分层

镜像是一摞只读的**层（layer）**：

1. 最底下是**基础镜像（base image）**，比如 `alpine`、`busybox` 或 Red Hat UBI（Universal Base Image，通用基础镜像），本质上是一个极小的 Linux 发行版。
2. 构建过程中的每一步（复制文件、安装软件包等）都会在上面叠一层。

层是共享的。十个镜像都以 `nginx:alpine` 为基础，那么这些基础层只存一份、只下载一次。这让镜像既省存储，拉取又快。

> 基础镜像刻意做得很精简，`ps`、`curl` 甚至 `bash` 往往都没有。如果 `kubectl exec` 报 `/bin/bash: not found`，换成 `/bin/sh` 试试。

## 在本机管理镜像

```bash
docker images            # list images stored on this machine
docker pull alpine       # download an image ahead of time
docker image inspect alpine
docker image prune       # remove dangling, unused images
```

容器启动时，镜像必须已经在节点上。要不要重新拉一份，取决于**镜像拉取策略**，在 Kubernetes 里就是容器上的 `imagePullPolicy`：

- `Always`：每次启动容器都去仓库检查一遍。
- `IfNotPresent`：节点上还没有这个镜像时才拉。
- `Never`：只用本地已有的副本。

不设置时，Kubernetes 对 `:latest` 标签（或不写标签）默认用 `Always`，其他标签默认用 `IfNotPresent`。这也是应该写明确版本标签（如 `nginx:1.27`）的又一个理由。

## 创建镜像的三种方式

| 方式 | 适用场景 |
|------|----------|
| `docker commit` | 快速试验：把在运行中的容器里做的改动保存下来 |
| **Dockerfile / Containerfile** | 正规做法。用一个文本文件描述构建过程，可以重复执行、可以评审 |
| `buildah` | 不依赖守护进程的脚本化构建，常用于 CI（Continuous Integration，持续集成） |

Dockerfile 和 Containerfile 是同一个东西，Podman 管它叫 Containerfile。

### 用 Dockerfile 构建

下面是考试第 14 题用到的 Dockerfile（[`exercise/ckad/task14-dockerfile`](../exercise/ckad/task14-dockerfile)）：

```dockerfile
FROM nginx:alpine

# Copy static site into NGINX default directory
COPY ./index.html /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

- `FROM` 选定基础镜像。
- `COPY` 把你的文件加成新的一层。
- `EXPOSE` 只是声明端口，起文档作用，并不会真正发布端口。
- `CMD` 是容器要运行的进程。`daemon off` 让 nginx 留在前台运行。**主进程一退出，容器就停了。**

构建、打标签、运行：

```bash
echo "hello from my image" > index.html
docker build -f task14-dockerfile -t myapp:1.0 .
docker images | grep myapp
docker run -d -p 8080:80 myapp:1.0
curl localhost:8080
```

`-t` 设置镜像名和标签；`-f` 指定一个名字不叫 `Dockerfile` 的文件；最后那个 `.` 是**构建上下文（build context）**，也就是 `COPY` 能看到哪个目录下的文件。

### 把镜像导出成文件

考试里常让你把镜像存成 tar 包：

```bash
docker save myapp:1.0 -o /tmp/myapp.tar
# with Podman the commands are the same:
podman build -t myapp:1.0 .
podman save myapp:1.0 -o /tmp/myapp.tar
```

在另一台机器上用 `docker load -i /tmp/myapp.tar` 再导入。

### commit：给运行中的容器拍快照

```bash
docker run --name customweb -it nginx sh
#   inside: touch /tmp/testfile; exit
docker commit customweb nginx:custom
docker run -it nginx:custom ls -l /tmp/testfile
```

用来探索很方便，但别人完全看不出这个镜像是怎么做出来的。要长期保留的镜像，一律用 Dockerfile。

## 要点回顾

- 镜像是叠在小型基础镜像上的一摞层；容器是镜像的一个运行实例。
- 镜像仓库存镜像，运行时启动容器，Kubernetes 在整个集群里编排它们。
- 镜像标签要写明确的版本，并清楚自己的 `imagePullPolicy` 是什么。
- `docker build -t name:tag .`、`docker save -o file.tar` 以及对应的 Podman 命令，都是 CKAD（Certified Kubernetes Application Developer，Kubernetes 认证应用开发者）考试必须会的。

下一篇，我们看看大规模运行这些镜像的系统：[02 · Kubernetes 是怎么工作的](02-kubernetes-architecture.md)。
