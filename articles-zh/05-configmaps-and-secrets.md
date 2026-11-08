# 05 · 配置应用：环境变量、ConfigMap 与 Secret

> 系列：[Kubernetes 实战](../articles/README.md)（英文索引）· 英文原文：[Configuring Applications](../articles/05-configmaps-and-secrets.md) · 上一篇：[04 · 运行工作负载](04-pods-and-deployments.md) · 下一篇：[06 · 持久化数据](06-storage.md)

同一个镜像，应该在开发、预发和生产环境里一模一样。环境之间真正不同的是**配置**：数据库地址、功能开关、密码、配置文件。Kubernetes 把这些配置作为 API 对象存在集群里，既不打进镜像，也不放在节点上。

## 缩写对照表

| 缩写 | 英文全称 | 中文 |
|---|---|---|
| MiB | Mebibyte | 兆字节（2²⁰ 字节） |
| RBAC | Role-Based Access Control | 基于角色的访问控制 |
| TLS | Transport Layer Security | 传输层安全协议 |
| SSH | Secure Shell | 安全外壳协议 |
| YAML | YAML Ain't Markup Language | 一种数据序列化格式 |
| URL | Uniform Resource Locator | 统一资源定位符 |

## 直接写环境变量

`kubectl create deploy` 没有 `--env` 参数，变量要在创建之后用 `kubectl set env` 加上：

```bash
kubectl create deploy mydb --image=mariadb
kubectl logs deploy/mydb          # mariadb refuses to start: no root password set
kubectl set env deploy mydb MARIADB_ROOT_PASSWORD=password
kubectl get pods                  # new Pods roll out and start fine
```

`kubectl run` 倒是接受 `--env`，但它建出来的是裸 Pod：

```bash
kubectl run mydb --image=mysql --env="MYSQL_ROOT_PASSWORD=password"
```

做实验时这样把值写死没问题。但要做到**配置即代码**，就得把这些值放进单独的对象里。

## ConfigMap

**ConfigMap** 存放不敏感的配置，上限 **1 MiB**（Mebibyte，2²⁰ 字节）。它有两种用法：

1. **作为变量**：键值对变成环境变量。
2. **作为文件**：每个键是文件名，值是文件内容，以卷的形式挂载进容器。

### 创建 ConfigMap

```bash
kubectl create cm app-vars --from-literal=COLOR=blue --from-literal=MODE=prod
kubectl create cm app-env  --from-env-file=app.env          # KEY=value lines
kubectl create cm myindex  --from-file=index.html            # key = file name
kubectl create cm myindex  --from-file=home.html=./index.html  # custom key
kubectl create cm nginx-conf --from-file=./conf.d/           # one key per file in the dir
kubectl get cm app-vars -o yaml
```

以上任何一条加上 `--dry-run=client -o yaml`，就会输出清单而不是真的创建对象。

### 把 ConfigMap 当变量用

最快的办法是把所有键一次性导入成变量，前缀可选：

```bash
kubectl create deploy mydb --image=mariadb --replicas=3
kubectl create cm mydbvars --from-literal=ROOT_PASSWORD=password
kubectl set env deploy/mydb --from=configmap/mydbvars --prefix=MARIADB_
kubectl get deploy mydb -o yaml | grep -A5 env
# -> MARIADB_ROOT_PASSWORD from configMapKeyRef mydbvars/ROOT_PASSWORD
```

写成 YAML 有两种形式：

```yaml
containers:
- name: app
  image: nginx
  env:
  - name: APP_COLOR                # one variable, picked from one key
    valueFrom:
      configMapKeyRef:
        name: app-vars
        key: COLOR
  envFrom:                         # all keys at once
  - configMapRef:
      name: app-vars
    prefix: CFG_                   # optional
```

### 把 ConfigMap 当文件用

把 ConfigMap 作为卷挂载。这就是考试第 3 题和 killer.sh 第 15 题：让 nginx 提供一个自定义的 `index.html`。

```bash
echo "welcome to the task3 webserver" > index.html
kubectl create configmap task3cm --from-file=index.html
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oregonpod
spec:
  containers:
  - name: oregonpod
    image: nginx
    volumeMounts:
    - name: config-vol
      mountPath: /usr/share/nginx/html   # every key becomes a file here
  volumes:
  - name: config-vol
    configMap:
      name: task3cm
      # optional: pick keys and rename them
      # items:
      # - key: index.html
      #   path: index.html
```

```bash
kubectl apply -f oregonpod.yaml
kubectl exec -it oregonpod -- curl -s localhost
# welcome to the task3 webserver
```

注意，挂载会**遮住**镜像在这个路径下原有的内容。只想加一个文件、又不想遮住整个目录，就用 `subPath`；但 `subPath` 挂载收不到 ConfigMap 的更新。

**更新行为：**修改 ConfigMap 后，挂载的文件会在短暂延迟后刷新（最长大约一分钟）。**环境变量不会刷新**，它们只在容器启动时读一次。要让新值生效，用 `kubectl rollout restart deploy/<name>` 重启 Pod。

## Secret

**Secret** 的用法和 ConfigMap 一样，但用来放敏感数据。它的值只是 **base64 编码，并没有加密**：

```bash
kubectl create secret generic dbpw --from-literal=ROOT_PASSWORD=password
kubectl get secret dbpw -o yaml               # data: ROOT_PASSWORD: cGFzc3dvcmQ=
echo cGFzc3dvcmQ= | base64 -d                 # password
kubectl get secret dbpw -o jsonpath='{.data.ROOT_PASSWORD}' | base64 -d
```

能读到 Secret 对象的人就能读到值。即便如此，Secret 仍然有用，因为：

- 值不会出现在你的 Deployment 清单或 Git 仓库里；
- 可以用 RBAC（Role-Based Access Control，基于角色的访问控制）单独限制访问（见[第 11 篇，英文](../articles/11-security-and-resources.md)）；
- 集群可以开启 etcd 的静态加密（encryption at rest），也可以接入外部密钥存储。

### Secret 的类型

| `kubectl create secret ...` | 用途 |
|-----------------------------|------|
| `generic` | 密码、令牌、SSH（Secure Shell）密钥或任意文件：`--from-literal`、`--from-file` |
| `tls` | 证书与私钥对：`--cert=tls.crt --key=tls.key`（Ingress 会用到），TLS 即 Transport Layer Security，传输层安全协议 |
| `docker-registry` | 从私有镜像仓库拉取镜像的凭证 |

```bash
kubectl create secret generic my-ssh-key --from-file=ssh-privatekey=$HOME/.ssh/id_rsa
kubectl create secret tls my-tls --cert=tls/my.crt --key=tls/my.key
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com --docker-username=me --docker-password=...
```

### 把 Secret 当变量用

考试第 1 题和 killer.sh 第 14 题（[`q-14-pod.yaml`](../exercise/killer-sh/q-14-pod.yaml)）：

```bash
kubectl create secret generic secret1 -n moon --from-literal=user=test --from-literal=pass=pwd
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-14
  namespace: moon
spec:
  containers:
  - name: pod-14
    image: nginx
    env:
    - name: SECRET1_USER
      valueFrom:
        secretKeyRef:
          name: secret1      # must be the Secret's real name
          key: user
    - name: SECRET1_PASS
      valueFrom:
        secretKeyRef:
          name: secret1
          key: pass
```

```bash
kubectl exec -n moon pod-14 -- printenv | grep SECRET
```

> 一个常见失误：`secretKeyRef` 下的 `name` 必须和你建的 Secret 同名。第 1 题里 Secret 叫 `insecret`，如果照抄文档里的 `my-secret`，Pod 就会卡在 `CreateContainerConfigError`。`kubectl describe pod` 会告诉你原因。

用 `envFrom` 加 `secretRef:` 可以导入全部键，命令行里 `kubectl set env deploy/x --from=secret/dbpw` 效果相同。

### 把 Secret 当文件用

```yaml
volumes:
- name: creds
  secret:
    secretName: my-ssh-key
    defaultMode: 0400        # owner read-only, good for keys
containers:
- name: app
  volumeMounts:
  - name: creds
    mountPath: /etc/creds
    readOnly: true
```

和 ConfigMap 一样，Secret 变了，挂载的文件会刷新，环境变量不会。

### 从私有仓库拉镜像

```yaml
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: registry.example.com/team/app:1.0
```

也可以把 `imagePullSecrets` 挂到 ServiceAccount 上，这样所有使用该 ServiceAccount 的 Pod 都会自动带上它。

## 要点回顾

- 配置不要打进镜像：普通配置用 ConfigMap，敏感配置用 Secret。
- 两者都能作为**环境变量**（`valueFrom`、`envFrom`、`kubectl set env --from`）或**文件**（卷）使用。
- 对象变化后，挂载的文件会刷新；环境变量要重启 Pod 才生效。
- Secret 只是 base64，不是加密，要靠 RBAC 和静态加密来保护。

下一篇：[用卷和 PersistentVolume 持久化数据](06-storage.md)。
