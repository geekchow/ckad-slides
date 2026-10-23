# 3. Setting Up a Lab and Driving It with kubectl

> Series: [Working with Kubernetes](README.md) · Previous: [How Kubernetes Works](02-kubernetes-architecture.md) · Next: [Pods and Deployments](04-pods-and-deployments.md)

## Choose a lab environment

For learning, you want a cluster that is cheap, quick to rebuild and close to vanilla Kubernetes. Managed cloud clusters (EKS, AKS, GKE) are not a good fit here. They hide or replace pieces you need to learn, such as ingress, network policy and storage, with cloud-specific ones.

The recommended options are:

1. **Minikube in an Ubuntu VM** (VirtualBox, VMware, or an Ubuntu VM in any cloud).
2. **Minikube in WSL2** on Windows, with Docker Desktop.

### Minikube in an Ubuntu VM

```bash
sudo apt install git
git clone https://github.com/sandervanvugt/ckad
cd ckad
./minikube-docker-setup.sh          # installs docker, kubectl and minikube
minikube start --cni=calico
kubectl get all
```

### Minikube on WSL2

```powershell
wsl -l -v          # make sure your Ubuntu distro runs WSL version 2
```

Then, inside Ubuntu, run the same setup script and start Minikube with the Docker driver:

```bash
minikube start --driver=docker --cni=calico
```

### Why `--cni=calico`?

Minikube's default network setup does **not** enforce NetworkPolicy. Calico does. If you skip it, your NetworkPolicy exercises will silently "work" and allow everything (see [article 7](07-services-and-networking.md)).

### Useful Minikube commands

```bash
minikube status
minikube ip                          # node IP, for NodePort / Ingress tests
minikube ssh                         # shell on the node (curl ClusterIPs from here)
minikube addons list
minikube addons enable ingress
minikube addons enable metrics-server
minikube delete && minikube start --cni=calico   # fresh cluster in a minute
```

> **macOS / Docker driver note:** the Minikube node runs inside a Docker network that your Mac cannot reach. `curl $(minikube ip):30080` will hang. Use `minikube ssh` and curl from inside the node, or use `minikube service <name> --url` or `kubectl port-forward`.

## kubectl: your one tool

`kubectl` is the command-line client for the API server. It reads its connection details from `~/.kube/config` (see [article 11](11-security-and-resources.md)).

### Set up your shell first

```bash
source <(kubectl completion bash)       # tab completion (add to ~/.bashrc)
alias k=kubectl
complete -o default -F __start_kubectl k  # completion for the alias too
export do="--dry-run=client -o yaml"    # k run x --image=nginx $do
```

### Get help without leaving the terminal

kubectl has no man pages. `-h` is the manual, and most commands show **examples at the top**:

```bash
kubectl create -h
kubectl create deployment -h
kubectl expose -h | less
kubectl get pods -h | grep -A5 label    # find the flag you need quickly
```

### The everyday verbs

| Goal | Command |
|------|---------|
| Create from scratch | `kubectl create deploy web --image=nginx` / `kubectl run pod1 --image=nginx` |
| Create or update from a file | `kubectl apply -f file.yaml` |
| See things | `kubectl get pods -o wide`, `kubectl get all -n ns`, `-A` for all namespaces |
| See details and **events** | `kubectl describe pod pod1` |
| Logs | `kubectl logs pod1 [-c container] [-f] [--previous]` |
| Shell into a container | `kubectl exec -it pod1 -- sh` |
| Change live | `kubectl edit deploy web`, `kubectl set image ...`, `kubectl scale ...` |
| Remove | `kubectl delete -f file.yaml`, `kubectl delete pod pod1` |

Your first application:

```bash
kubectl create deploy firstapp --image=nginx
kubectl get deploy,rs,pods
```

One command created three objects: a Deployment, which created a ReplicaSet, which created a Pod. That chain is the reconciliation loop from [article 2](02-kubernetes-architecture.md) at work.

## The imperative-to-declarative workflow

Writing YAML from memory is slow and error-prone. The fast workflow is:

1. **Generate** a skeleton with an imperative command and `--dry-run=client -o yaml`.
2. **Edit** the parts the generator can't produce (volumes, probes, env from Secrets …). Copy snippets from kubernetes.io.
3. **Validate** with a dry run.
4. **Apply**.

```bash
kubectl run inpod --image=nginx -n indiana --dry-run=client -o yaml > inpod.yaml
vim inpod.yaml
kubectl apply -f inpod.yaml --dry-run=client     # syntax check
kubectl apply -f inpod.yaml
```

You can also export a live object to YAML, but clean it before reusing it. Remove `status`, `metadata.uid`, `resourceVersion`, `creationTimestamp` and `managedFields`:

```bash
kubectl get deploy mydb -o yaml > mydb.yaml
```

## Some fields can't be changed

Many Pod fields are **immutable**. If you try to `kubectl edit` a Pod's `command`, you get:

```text
The Pod "nevatest" is invalid: spec: Forbidden: pod updates may not change fields other than
`spec.containers[*].image`, `spec.initContainers[*].image`, `spec.activeDeadlineSeconds`,
`spec.tolerations` (only additions ...), `spec.terminationGracePeriodSeconds` ...
```

The fix is to delete and recreate the Pod. `kubectl edit` saves your changed copy to `/tmp/kubectl-edit-xxxx.yaml` when it is rejected, so you can use it:

```bash
kubectl replace --force -f /tmp/kubectl-edit-xxxx.yaml
```

For Deployments this problem doesn't come up. Editing the Pod template simply starts a rollout of new Pods.

## Don't rely on the Dashboard

Many distributions ship a web Dashboard. Don't depend on it for the exam. It may not be installed, and it can do much less than kubectl.

## Key takeaways

- Minikube with `--cni=calico` is the reference lab for this series.
- Set up completion, the `k` alias and a `$do` dry-run variable before anything else.
- `-h`, `kubectl explain` and `kubectl describe` (look at **Events**) answer most questions.
- Generate YAML with `--dry-run=client -o yaml`, edit it, then apply.

Next: [running workloads with Pods and Deployments](04-pods-and-deployments.md).
