# 11. Securing Workloads: RBAC, ServiceAccounts, SecurityContext and Resource Limits

> Series: [Working with Kubernetes](README.md) · Previous: [Deployment Strategies](10-deployment-strategies.md) · Next: [Observability](12-observability.md)

Every request to the API server goes through three checks: **authentication** (who are you?), **authorization** (are you allowed to do this?) and **admission** (does the request follow the cluster's rules?). This article covers the parts an application developer works with, plus the resource controls that keep one app from starving the others.

## Authentication: who are you?

There are two kinds of identity:

- **Users** (people). Kubernetes has no User object. Users come from certificates, OIDC and similar sources. In Minikube and kubeadm labs you work as a local admin that is set up for you. Creating users is CKA material.
- **ServiceAccounts** (workloads). These *are* Kubernetes objects, with a namespace.

Your client identity lives in `~/.kube/config`:

```bash
kubectl config view                 # clusters, users, contexts (secrets redacted)
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <name>
kubectl config set-context --current --namespace=dev   # change your default namespace
```

| kubeconfig section | Contains |
|--------------------|----------|
| `clusters` | API server URL + CA certificate |
| `users` | Client certificate/key or token |
| `contexts` | cluster + user + default namespace |

> In the exam, each question tells you which context to use. Run the `kubectl config use-context` command it gives you **every time**.

## Authorization: RBAC

Role-Based Access Control has four objects:

| Object | Scope | Says |
|--------|-------|------|
| **Role** | Namespace | "These verbs on these resources, in this namespace" |
| **ClusterRole** | Cluster | The same, but cluster-wide (or reusable in many namespaces) |
| **RoleBinding** | Namespace | "This subject gets this (Cluster)Role, in this namespace" |
| **ClusterRoleBinding** | Cluster | "This subject gets this ClusterRole everywhere" |

Subjects are users, groups or ServiceAccounts. Permissions only add up. There is no "deny" rule.

### Check what an identity can do

```bash
kubectl auth can-i get pods
kubectl auth can-i create deployments -n prod
kubectl auth can-i '*' '*'                               # am I cluster-admin?
kubectl auth can-i get pods --as bob@example.com
kubectl auth can-i get pods -n bellevue \
  --as=system:serviceaccount:bellevue:viewer             # as a ServiceAccount
kubectl auth can-i --list -n bellevue --as=system:serviceaccount:bellevue:viewer
```

## ServiceAccounts

Every Pod runs as a ServiceAccount. If you don't name one, it runs as `default` in its namespace. Kubernetes mounts a short-lived token at `/var/run/secrets/kubernetes.io/serviceaccount/`, and programs in the Pod use it to call the API.

The `default` ServiceAccount has almost no permissions, and most apps need none. Create a dedicated ServiceAccount when a Pod needs to talk to the API, for example operators, CI runners, monitoring agents or network plugins:

```bash
kubectl get sa -n kube-system
kubectl describe pod -n kube-system -l k8s-app=kube-dns | grep -i "service account"
kubectl describe clusterrolebinding system:coredns
kubectl describe clusterrole system:coredns       # what CoreDNS may read
```

### Walkthrough: a read-only Pod viewer

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

In YAML, it is one field on the Pod spec. This is exam task 15 ([`task15.yaml`](../exercise/ckad/task15.yaml)):

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

`serviceAccountName` can't be changed on a running Pod. Delete and recreate it, or set it on a Deployment, which rolls out new Pods.

## SecurityContext: limit what the container can do

A `securityContext` controls Linux-level privileges. It can be set on the **Pod** (applies to all containers) and on each **container** (wins over the Pod setting).

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

Field placement matters, and the two levels accept different fields:

| Pod level only | Container level only | Both |
|----------------|----------------------|------|
| `fsGroup`, `supplementalGroups`, `sysctls` | `allowPrivilegeEscalation`, `capabilities`, `privileged`, `readOnlyRootFilesystem` | `runAsUser`, `runAsGroup`, `runAsNonRoot`, `seccompProfile`, `seLinuxOptions` |

Exam task 13 ([`task13-security.yaml`](../exercise/ckad/task13-security.yaml)) combined `supplementalGroups: [2000]` at Pod level with `allowPrivilegeEscalation: false` at container level.

When a security setting stops a Pod from starting, the Pod shows `CreateContainerConfigError`, `Error` or `CrashLoopBackOff`. A classic example is `runAsNonRoot: true` with an image that runs as root. Stock `nginx` also fails with a non-root UID, because it needs to bind port 80 and write to `/var/cache/nginx`. Use `kubectl describe pod` and `kubectl logs` to find out why.

## Resource requests, limits and quotas

### Requests and limits

```yaml
resources:
  requests:            # reserved: used by the scheduler to pick a node
    cpu: 250m          # 250 millicores = 0.25 CPU
    memory: 64Mi
  limits:              # ceiling: enforced on the node at runtime
    cpu: 500m
    memory: 128Mi
```

- **Requests** decide scheduling. A Pod only goes to a node with that much *unreserved* capacity. If no node fits, the Pod stays **Pending** with a `FailedScheduling … Insufficient cpu/memory` event.
- **Limits** cap usage. Going over the **CPU** limit means the container is **throttled**. Going over the **memory** limit means it is **OOMKilled** and restarted.
- CPU uses millicores (`500m` = half a core). Memory uses `Mi`/`Gi` (binary) or `M`/`G` (decimal).

Set them on a running Deployment without editing YAML. This is exam task 11:

```bash
kubectl create ns nebraska
kubectl create deploy snowdeploy --image=nginx -n nebraska
kubectl set resources deploy snowdeploy -n nebraska \
  --requests=cpu=100m,memory=64Mi --limits=cpu=200m,memory=128Mi
```

Check real usage. This needs the metrics-server (`minikube addons enable metrics-server`):

```bash
kubectl top pods -n nebraska
kubectl top nodes
```

### ResourceQuota and LimitRange (namespace level)

A **ResourceQuota** caps the *total* use of a namespace:

```bash
kubectl create quota team-a -n team-a \
  --hard=requests.cpu=2,requests.memory=2Gi,limits.cpu=4,limits.memory=4Gi,pods=10
kubectl describe quota -n team-a          # used vs hard
```

Once a quota covers CPU/memory, **every Pod in that namespace must declare those requests/limits**, or it is rejected. The error shows on the **ReplicaSet**, not the Deployment:

```bash
kubectl describe rs -n team-a | grep -A3 Events
# Error creating: pods "x" is forbidden: failed quota: team-a: must specify limits.cpu ...
```

A **LimitRange** fills in default requests/limits for containers that don't set them (and enforces a min/max). It works well together with a quota.

## Key takeaways

- RBAC: Role/ClusterRole say *what*. RoleBinding/ClusterRoleBinding say *who*. Test with `kubectl auth can-i --as`.
- Give Pods a dedicated ServiceAccount (`serviceAccountName`) only when they call the API.
- SecurityContext: run as non-root, drop capabilities, no privilege escalation. Know which fields go at Pod level and which at container level.
- Requests drive scheduling. Limits throttle CPU and OOM-kill on memory. Quotas make resources mandatory in a namespace.

Next: [keeping applications healthy with probes, logs and metrics](12-observability.md).
