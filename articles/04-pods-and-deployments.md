# 4. Running Workloads: Pods, Multi-Container Pods and Deployments

> Series: [Working with Kubernetes](README.md) · Previous: [Lab Setup and kubectl](03-lab-setup-and-kubectl.md) · Next: [ConfigMaps and Secrets](05-configmaps-and-secrets.md)

## The Pod: the smallest unit

A **Pod** is one or more containers that are always scheduled together on the same node and share:

- **one network namespace**: same IP address, and they can reach each other on `localhost`;
- **volumes**: any volume in the Pod can be mounted by any of its containers.

Most Pods run one main container. You add more containers only when they are tightly coupled to the main one (see "Multi-container patterns" below).

### Creating a Pod

```bash
kubectl run web --image=nginx --port=80 --labels="app=web,tier=frontend"
kubectl get pods -o wide --show-labels
kubectl describe pod web
```

The same Pod as YAML, generated with `kubectl run web --image=nginx --dry-run=client -o yaml`:

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

### Commands and arguments

A container runs its image's default `ENTRYPOINT`/`CMD` unless you override it:

| Kubernetes field | Overrides Dockerfile | `kubectl run` syntax |
|------------------|----------------------|----------------------|
| `command` | `ENTRYPOINT` | `kubectl run p --image=busybox --command -- sleep 3600` |
| `args` | `CMD` | `kubectl run p --image=busybox -- sleep 3600` |

Busybox and alpine images exit at once with no command. Keep test Pods alive with `sleep infinity` or `sleep 1d`:

```yaml
command: ["sh", "-c", "touch /tmp/ready && sleep 1d"]
```

### Throw-away Pods for testing

```bash
# interactive shell, deleted on exit
kubectl run tmp --image=busybox:1.31.0 --restart=Never --rm -it -- sh

# one-shot command, output printed, Pod deleted
kubectl run tmp --image=nginx:alpine --restart=Never --rm -i -- curl -s 10.244.0.84
```

`--restart=Never` creates a bare Pod (no restart loop). `--rm` deletes it when the command ends.

### Deleting Pods fast

A normal delete waits up to 30 seconds for a graceful shutdown. In the exam, skip the wait:

```bash
kubectl delete pod pod1 --force --grace-period=0
```

## Labels and selectors: how things find each other

Labels are key/value tags in `metadata.labels`. **Selectors** query them. Deployments, Services and NetworkPolicies all find their Pods this way. There are no hard-coded links between objects.

```bash
kubectl get pods -A -l tier=control-plane              # filter by label
kubectl get pods -A --selector tier=control-plane      # same thing
kubectl get pods -l 'env in (prod,staging)'
kubectl label pod web env=prod                         # add
kubectl label pod web env=dev --overwrite              # change
kubectl label pod web env-                             # remove
```

Exam task 2 asked for the names of all control-plane Pods, saved to a file:

```bash
kubectl get pods -A -l tier=control-plane -o name | sed 's/^.*\///' > /tmp/pods.txt
# or, without sed:
kubectl get pods -A -l tier=control-plane -o custom-columns=NAME:.metadata.name --no-headers > /tmp/pods.txt
```

## Multi-container patterns

### Sidecar containers

A **sidecar** helps the main container. Typical jobs are shipping logs, refreshing a cache or running a proxy. Since Kubernetes 1.29 (stable in 1.33), the native way to write one is an **init container with `restartPolicy: Always`**. It starts before the app containers, keeps running beside them, and stops after them.

This is the pattern from [`exercise/ckad/sidcar-origin.yaml`](../exercise/ckad/sidcar-origin.yaml), as a Pod:

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

The two containers share an `emptyDir` volume. The app writes to it and the sidecar reads from it. In exam task 4, the sidecar ran `sleep 15`. Because it has `restartPolicy: Always`, Kubernetes restarted it every 15 seconds, which shows up in the RESTARTS column.

You can also write a sidecar as a **second entry in `containers:`**. killer.sh Q16 does this. It adds a `logger-con` container (`busybox:1.31.0`) to an existing Deployment and runs `tail -f` on the shared `cleaner.log`, so `kubectl logs ... -c logger-con` shows the file. Both forms are valid. The init-container form guarantees start and stop order.

### Init containers

A plain init container (no `restartPolicy`) runs **to completion before** the app containers start. Use it to prepare things such as config files, schema migrations or waiting for a dependency.

killer.sh Q17: an nginx Deployment serves an empty volume. Add an init container that writes `index.html` first:

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

Verify the result from a temporary Pod: `kubectl run tmp --rm -i --restart=Never --image=nginx:alpine -- curl -s <pod-ip>`.

## Deployments: don't run naked Pods

A bare Pod is not rescheduled if its node dies, and nobody replaces it if it is deleted. In real use, wrap Pods in a **Deployment**:

```text
Deployment ──manages──▶ ReplicaSet (one per Pod-template version) ──manages──▶ Pods
```

```bash
kubectl create deploy web --image=nginx:1.17 --replicas=3
kubectl get deploy,rs,pods -l app=web
kubectl delete pod <one-of-them>   # watch a replacement appear immediately
kubectl scale deploy web --replicas=5
```

The important parts of a Deployment spec, from exam task 6 ([`task6-dp.yaml`](../exercise/ckad/task6-dp.yaml)):

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

> `spec.selector` is immutable and must match `spec.template.metadata.labels`. If it doesn't, the API server rejects the Deployment.

### Rolling updates and rollbacks

Changing anything in `spec.template` (image, env, resources …) creates a **new ReplicaSet**. The Deployment controller then shifts Pods from the old one to the new one, within the `maxSurge`/`maxUnavailable` limits.

```bash
kubectl set image deploy/updates nginx=nginx:latest    # or: kubectl edit deploy updates
kubectl rollout status deploy/updates
kubectl get rs -l type=prod                            # two ReplicaSets now: old (0) and new (3)

kubectl rollout history deploy/updates                 # list revisions
kubectl rollout history deploy/updates --revision=2    # what changed in rev 2
kubectl rollout undo deploy/updates                    # back to the previous revision
kubectl rollout undo deploy/updates --to-revision=1
```

killer.sh Q8 is a typical case: a new revision is broken (bad image, failing Pods). Find it with `rollout history`, then `rollout undo` to the last good revision. Old ReplicaSets are kept, scaled to zero, which is why rollbacks are instant.

> **Try it:** the *Rolling update* tab in the [interactive labs](interactive/k8s-visual-labs.html#rollout) steps through a rollout pass by pass for any `replicas`, `maxSurge` and `maxUnavailable`.

The other strategy, `type: Recreate`, kills all old Pods before it starts new ones. Use it when two versions must never run at the same time (for example, a database schema change). It causes downtime.

## Key takeaways

- A Pod is containers sharing one IP and its volumes. Override `command`/`args` to change what runs.
- Labels and selectors connect everything. Learn `-l`, `--show-labels` and `kubectl label`.
- Sidecars are init containers with `restartPolicy: Always`. Plain init containers run once, before the app.
- Use Deployments, not bare Pods. Template changes trigger rolling updates, and `kubectl rollout undo` reverts them.

Next: [configuring applications with ConfigMaps and Secrets](05-configmaps-and-secrets.md).
