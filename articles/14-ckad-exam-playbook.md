# 14. The CKAD Playbook: Speed, Habits and Common Mistakes

> Series: [Working with Kubernetes](README.md) · Previous: [Extending the API](13-extending-the-api.md)

The CKAD is a hands-on exam: about 2 hours, 15–20 tasks, each on a live cluster. Knowing Kubernetes is half of it. The other half is **working fast and checking your work**. This article collects the habits behind the solutions in `exercise/ckad` and `exercise/killer-sh`.

## Before the first question

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

## The per-question routine

1. **Switch context.** Copy and paste the `kubectl config use-context …` line from the question. Do it every time.
2. **Note the namespace.** Put `-n <ns>` on every command, or `kubectl config set-context --current --namespace=<ns>`.
3. **Generate, don't type**: `k create … $do > qN.yaml`.
4. **Find the missing snippet** with `kubectl explain` or the docs search (keywords below).
5. **Validate**: `k apply -f qN.yaml --dry-run=server` catches schema errors too.
6. **Apply, then verify**: `get`, `describe`, `logs`, a `curl` from a temp Pod.
7. **Save files where asked.** Many questions also ask you to save YAML or output to a specific path.

## Imperative commands worth memorising

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

## Docs search keywords

kubernetes.io is allowed in the exam. These searches lead straight to a ready-to-copy YAML snippet:

| You need | Search |
|----------|--------|
| ConfigMap as volume / env | `configmap` → "Configure a Pod to Use a ConfigMap" |
| Secret as env | `secret` → "Distribute Credentials Securely Using Secrets" |
| Probes | `liveness` → "Configure Liveness, Readiness and Startup Probes" |
| Sidecar / init container | `sidecar`, `init containers` |
| PV / PVC / Pod | `persistent volume` → "Configure a Pod to Use a PersistentVolume for Storage" |
| StorageClass | `storageclass` |
| NetworkPolicy | `network policy` |
| SecurityContext | `security context` |
| Resource limits | `assign memory resources` |
| Deployment strategy | `maxSurge` |
| Ingress | `ingress` |

## Common mistakes (and how to spot them)

**1. Editing immutable Pod fields.**
`kubectl edit pod` → "Forbidden: pod updates may not change fields other than …" (see [`frequent-error.md`](../exercise/ckad/frequent-error.md)). Run `k replace --force -f /tmp/kubectl-edit-*.yaml`, or delete and re-apply.

**2. Wrong names in references.**
`secretKeyRef.name`, `configMap.name` or `claimName` doesn't match the real object, and the Pod hangs in `CreateContainerConfigError`/`ContainerCreating`. `k describe pod` names the missing object.

**3. Labels in the wrong place.**
A Service or NetworkPolicy selects **Pod** labels. Labels on Deployment metadata don't count. Check with `k get pods --show-labels` and `k get ep <svc>`.

**4. Forgetting the namespace.**
The object exists, but in `default`. The checker looks in `moon`. Run `k get <kind> -A | grep <name>`.

**5. Ignoring the default StorageClass.**
The PVC binds to a dynamically provisioned volume instead of your PV, or stays `Pending`. Use `storageClassName: ""` (or the class name the question gives).

**6. Readiness vs liveness confused.**
Re-read the question. Readiness is about traffic, liveness restarts the container.

**7. Testing from the wrong place.**
ClusterIPs only answer from inside the cluster. Use a temporary Pod:

```bash
k run tmp --image=nginx:alpine --restart=Never --rm -i -- curl -s -m 2 svc.ns:80
```

On Minikube with the Docker driver on macOS, NodePorts aren't reachable from the host. Use `minikube ssh` and curl from there.

**8. NetworkPolicy "not working" in your lab.**
The CNI doesn't enforce it. Start Minikube with `--cni=calico`.

**9. YAML indentation.**
`volumeMounts` belongs under the container, `volumes` under `spec`, and `fsGroup` under the **Pod** `securityContext`. `k apply --dry-run=server` finds most of these, and `k explain pod.spec.containers.volumeMounts` confirms where a field goes.

**10. Typos in flags.**
`--from-liberal`, `--iamge` and `-from` (single dash) all appear in real notes. Use tab completion, and read the error. kubectl usually tells you exactly which flag it doesn't know.

## Practice map

| Exercise | Topic | Article |
|----------|-------|---------|
| Task 1, killer Q14 | Secret → env | [5](05-configmaps-and-secrets.md) |
| Task 2 | Label selectors, output formatting | [4](04-pods-and-deployments.md) |
| Task 3, killer Q15 | ConfigMap → volume | [5](05-configmaps-and-secrets.md) |
| Task 4, killer Q16/Q17 | Sidecar and init containers | [4](04-pods-and-deployments.md) |
| Task 5, killer Q6 | Probes | [12](12-observability.md) |
| Task 6, killer Q8 | Rolling update and rollback | [4](04-pods-and-deployments.md), [10](10-deployment-strategies.md) |
| Task 7, killer Q18 | Services and troubleshooting | [7](07-services-and-networking.md) |
| Task 8 | NetworkPolicy | [7](07-services-and-networking.md) |
| Task 9, killer Q12/Q13 | Volumes, PV/PVC, StorageClass | [6](06-storage.md) |
| Task 10 | Helm | [9](09-helm-and-kustomize.md) |
| Task 11 | Requests/limits, `kubectl top` | [11](11-security-and-resources.md), [12](12-observability.md) |
| Task 12 | Canary deployment | [10](10-deployment-strategies.md) |
| Task 13 | SecurityContext | [11](11-security-and-resources.md) |
| Task 14, killer Q11 | Build/save images | [1](01-containers-and-images.md) |
| Task 15 | ServiceAccount | [11](11-security-and-resources.md) |

## Final advice

- **Skip and come back.** A 2% question that takes 15 minutes is a bad trade. Flag it and move on.
- **Verify every answer.** "Applied" is not "working". A 10-second `get`/`curl` check catches most lost points.
- **Practise under time pressure.** Rebuild your lab (`minikube delete && minikube start --cni=calico`) and redo the tasks in `exercise/` until each takes only a few minutes.

Good luck, and happy shipping.
