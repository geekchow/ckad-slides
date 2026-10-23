# 6. Persisting Data: Volumes, PersistentVolumes and StorageClasses

> Series: [Working with Kubernetes](README.md) · Previous: [ConfigMaps and Secrets](05-configmaps-and-secrets.md) · Next: [Services and Networking](07-services-and-networking.md)

A container's filesystem is temporary. When the container restarts, everything it wrote is gone. Pods move between nodes, so local disks don't help either. Kubernetes solves this in layers, from "share files between containers" up to "get me 10 GiB of durable storage, wherever I run".

## Layer 1: Pod volumes

A volume is declared in `spec.volumes` and mounted into containers with `volumeMounts`. You have already seen two kinds:

| Volume type | Lifetime | Typical use |
|-------------|----------|-------------|
| `emptyDir` | As long as the **Pod** | Scratch space, sharing files between containers (sidecars) |
| `configMap` / `secret` | Follows the object | Config and credentials as files ([article 5](05-configmaps-and-secrets.md)) |
| `hostPath` | As long as the **node's** disk | Node-level agents, single-node labs |
| `persistentVolumeClaim` | Independent of the Pod | Real application data |

### hostPath in a lab

Exam task 9 serves a page from a folder on the Minikube node ([`task9-pv.yaml`](../exercise/ckad/task9-pv.yaml)):

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

`hostPath` ties the data to **one node**. That works on single-node Minikube. In a real cluster it breaks as soon as the Pod lands somewhere else. It is also a security risk, because the Pod can read the host's files. Use it only for learning.

## Layer 2: PersistentVolumes and PersistentVolumeClaims

Kubernetes splits storage into two objects so that developers and cluster admins don't have to know each other's details:

```text
 Developer                      Cluster / admin
 ─────────                      ───────────────
 Pod ──uses──▶ PVC  ──binds──▶  PV ──points at──▶ real storage (NFS, EBS, Ceph, hostPath …)
              "I need 2Gi RWO"  "Here is 2Gi RWO"
```

- A **PersistentVolume (PV)** is a piece of storage. It is a **cluster-scoped** object (no namespace).
- A **PersistentVolumeClaim (PVC)** is a request for storage. It is **namespaced** and lives next to the Pod.
- Kubernetes **binds** a PVC to a PV that matches its size, access mode, `storageClassName` and volume mode.

### Access modes

| Mode | Short | Meaning |
|------|-------|---------|
| `ReadWriteOnce` | RWO | Mounted read-write by a single **node** |
| `ReadOnlyMany` | ROX | Mounted read-only by many nodes |
| `ReadWriteMany` | RWX | Mounted read-write by many nodes (NFS, CephFS …) |
| `ReadWriteOncePod` | RWOP | Mounted read-write by a single **Pod** |

### Static provisioning, end to end (killer.sh Q12)

1. **The PV** ([`q-12-pv.yaml`](../exercise/killer-sh/q-12-pv.yaml)). No namespace:

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

2. **The PVC** ([`q-12-pvc.yaml`](../exercise/killer-sh/q-12-pvc.yaml)), in the application's namespace:

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

3. **The Deployment** mounts the claim ([`q-12-dp.yaml`](../exercise/killer-sh/q-12-dp.yaml)):

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

> **Watch out for the default StorageClass.** Minikube (and most clusters) have a *default* StorageClass. A PVC without `storageClassName` gets the default class and is **dynamically provisioned** instead of binding to your PV. To bind to a manually created PV, set `storageClassName: ""` on the PVC (or give the PV and PVC the same class name). If a PVC stays `Pending`, `kubectl describe pvc` tells you why.

## Layer 3: StorageClasses and dynamic provisioning

Creating PVs by hand doesn't scale. A **StorageClass** describes a *type* of storage and names a **provisioner** that creates a PV automatically whenever a PVC asks for that class.

killer.sh Q13 ([`q-13-sc.yaml`](../exercise/killer-sh/q-13-sc.yaml), [`q-13-pvc.yaml`](../exercise/killer-sh/q-13-pvc.yaml)):

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

`moon-retainer` is a made-up provisioner, so the PVC stays `Pending` and `kubectl describe pvc` shows `waiting for a volume to be created … by external provisioner "moon-retainer"`. That is the expected answer to the question. In Minikube, the built-in `standard` class (provisioner `k8s.io/minikube-hostpath`) actually creates volumes.

```bash
kubectl get sc
kubectl get pvc -n moon
kubectl describe pvc moon-pvc-126 -n moon
```

### Reclaim policy: what happens when the claim is deleted

| `reclaimPolicy` | Effect when the PVC is deleted |
|-----------------|-------------------------------|
| `Delete` | The PV **and the underlying storage** are deleted (the default for dynamic provisioning) |
| `Retain` | The PV goes to `Released` and the data is kept. An admin has to clean up or re-use it by hand |

Use `Retain` for anything you can't afford to lose by mistake.

## Key takeaways

- `emptyDir` lives as long as the Pod, `hostPath` lives on one node, and PVCs are for real data.
- PVs are cluster-scoped storage. PVCs are namespaced requests. They bind on size, access mode and class.
- StorageClasses add dynamic provisioning. Know the default class and how `storageClassName: ""` changes binding.
- `kubectl describe pvc` explains every `Pending` claim.

Next: [networking with Services, DNS and NetworkPolicy](07-services-and-networking.md).
