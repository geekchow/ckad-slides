# 12. Keeping Applications Healthy: Probes, Logs and Metrics

> Series: [Working with Kubernetes](README.md) · Previous: [Security and Resources](11-security-and-resources.md) · Next: [Extending the API](13-extending-the-api.md)

Kubernetes restarts a container when its process **exits**. But many failures don't end the process: a deadlocked thread, a web server returning 500s, an app still loading 2 GB of cache. **Probes** let you tell Kubernetes what "healthy" means for *your* app. **Logs, events and metrics** let you find out what went wrong.

## Probes

### Three kinds of probe

| Probe | Question it answers | When it fails |
|-------|--------------------|---------------|
| **startupProbe** | "Has the app finished starting?" | The container is killed and restarted. Other probes wait until this one succeeds. |
| **livenessProbe** | "Is the app still alive?" | The container is **restarted** |
| **readinessProbe** | "Can it take traffic right now?" | The Pod is **removed from Service endpoints**. It is not restarted. |

The key difference: **liveness restarts, readiness only stops traffic**. Readiness also controls rolling updates. A new Pod only counts as available, and an old one is only removed, once the new Pod is Ready.

### Four ways to check

| Mechanism | Passes when |
|-----------|-------------|
| `exec` | The command exits with status 0 |
| `httpGet` | The HTTP response code is 200–399 |
| `tcpSocket` | A TCP connection to the port can be opened |
| `grpc` | The gRPC health check returns `SERVING` |

### Timing knobs

```yaml
initialDelaySeconds: 5    # wait before the first check
periodSeconds: 10         # how often to check
timeoutSeconds: 1         # how long one check may take
failureThreshold: 3       # failures in a row before acting
successThreshold: 1       # successes needed to be OK again (readiness only can be >1)
```

The time before action is roughly `failureThreshold × periodSeconds`. For slow-starting apps, use a `startupProbe` with a generous `failureThreshold` (for example `30 × 10s` = 5 minutes) instead of a huge `initialDelaySeconds` on the liveness probe.

### Example: exec probe (killer.sh Q6)

*"Create Pod `pod6` (busybox:1.31.0) with a readiness probe that runs `cat /tmp/ready`, initial delay 5s, period 10s. The container should run `touch /tmp/ready && sleep 1d`."* ([`q-06-pod.yaml`](../exercise/killer-sh/q-06-pod.yaml))

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

> Read the question carefully: was it a **readiness** or a **liveness** probe? Both use the same check syntax, but they do very different things.

### Example: HTTP probes on a web app

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

### Example: probing a dependency (exam task 5)

Task 5 asked for a probe that checks the Kubernetes API's health endpoint. First find the endpoint from the node:

```bash
minikube ssh -- curl -k -s https://localhost:8443/readyz     # ok
```

From inside a Pod, the API server is always at `https://kubernetes.default.svc`, so no node IP needs to be hard-coded:

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

> Be careful with liveness probes that check a *dependency* in production. If the database goes down, every app container restarts at the same time, which makes things worse. A readiness probe is usually the better tool for dependencies.

## Logs

Containers should write logs to **stdout/stderr**. The container runtime stores them on the node, and `kubectl logs` reads them:

```bash
kubectl logs pod1
kubectl logs pod1 -c sidecar           # pick a container in a multi-container Pod
kubectl logs pod1 --all-containers
kubectl logs pod1 -f --tail=50         # follow
kubectl logs pod1 --previous           # the crashed instance before the restart
kubectl logs deploy/web                # one Pod of the Deployment
kubectl logs -l app=web --prefix       # all Pods matching a label
```

If an app only writes to a file, add a sidecar that runs `tail -F` on the file to stdout ([article 4](04-pods-and-deployments.md)).

For containers running *outside* Kubernetes (killer.sh Q11), the commands are the same idea: `docker logs <name>` or `podman logs <name>`.

## Events and describe

Before logs, check **events**. They explain scheduling problems, image pull errors, probe failures, OOM kills and quota rejections:

```bash
kubectl describe pod pod1                       # Events section at the bottom
kubectl get events --sort-by=.lastTimestamp     # recent events in the namespace
kubectl get events -A --field-selector type=Warning
```

### Reading Pod status

| Status / reason | Usual cause | Look at |
|-----------------|-------------|---------|
| `Pending` | No node fits (resources, taints, affinity) or a PVC is not bound | `describe pod` events, `describe pvc` |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name/tag, private registry without `imagePullSecrets` | `describe pod` |
| `CreateContainerConfigError` | A ConfigMap/Secret or key is missing, or a securityContext conflict | `describe pod` |
| `CrashLoopBackOff` | The process keeps exiting, or the liveness probe keeps failing | `logs --previous`, `describe` |
| `OOMKilled` (last state) | The memory limit is too low | `describe pod`, `top pod` |
| `Running` but `0/1` READY | The readiness probe is failing | `describe pod` events |

## Metrics

The **metrics-server** collects CPU/memory usage from each kubelet:

```bash
minikube addons enable metrics-server
kubectl -n kube-system get deploy metrics-server
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl top pod snowdeploy -n nebraska --containers
```

It also feeds the HorizontalPodAutoscaler:

```bash
kubectl autoscale deploy web --cpu-percent=70 --min=2 --max=10   # needs CPU requests set
kubectl get hpa
```

For dashboards, alerting and history you need more: **Prometheus + Grafana** for metrics, **Loki/Elasticsearch** for logs, **Jaeger/OpenTelemetry** for traces. They are often installed with Helm or an operator ([article 13](13-extending-the-api.md)).

## Key takeaways

- Liveness restarts, readiness removes from endpoints, startup protects slow starters.
- Probes use `exec`, `httpGet`, `tcpSocket` or `grpc`. Tune `initialDelaySeconds`, `periodSeconds` and `failureThreshold`.
- Debug order: `get` → `describe` (events) → `logs` (`--previous`) → `exec`.
- `kubectl top` needs metrics-server. HPAs need CPU requests.

Next: [extending Kubernetes with CRDs and operators](13-extending-the-api.md).
