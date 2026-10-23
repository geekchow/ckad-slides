# Working with Kubernetes: A Hands-On Article Series

This series turns the lesson slides (`L01`–`L17`) and the exam exercises (`exercise/ckad`, `exercise/killer-sh`) in this repository into a set of articles you can read in order. Each article explains *why* a feature exists, *what* the moving parts are, and then walks through commands you can run on a local Minikube cluster.

The series follows the shape of the CKAD (Certified Kubernetes Application Developer) curriculum. It is also useful for anyone who builds and runs applications on Kubernetes day to day.

## Reading order

| # | Article | Built from |
|---|---------|------------|
| 1 | [Containers and Images: The Building Blocks](01-containers-and-images.md) | L01, L02, exam task 14, killer.sh Q11 |
| 2 | [How Kubernetes Works: Architecture and the Resource Model](02-kubernetes-architecture.md) | L03, L14 |
| 3 | [Setting Up a Lab and Driving It with kubectl](03-lab-setup-and-kubectl.md) | L04, frequent-error.md |
| 4 | [Running Workloads: Pods, Multi-Container Pods and Deployments](04-pods-and-deployments.md) | Exam tasks 2, 4, 6; killer.sh Q6, Q8, Q9, Q16, Q17 |
| 5 | [Configuring Applications: Environment Variables, ConfigMaps and Secrets](05-configmaps-and-secrets.md) | L12, exam tasks 1, 3; killer.sh Q14, Q15 |
| 6 | [Persisting Data: Volumes, PersistentVolumes and StorageClasses](06-storage.md) | Exam task 9; killer.sh Q12, Q13 |
| 7 | [Networking: Services, DNS and NetworkPolicy](07-services-and-networking.md) | L10, exam tasks 7, 8; killer.sh Q18 |
| 8 | [Getting Traffic In: Ingress and Gateway API](08-ingress-and-gateway-api.md) | L11 |
| 9 | [Packaging Applications: Helm and Kustomize](09-helm-and-kustomize.md) | L09, exam task 10 |
| 10 | [Releasing Safely: Rolling Updates, Blue/Green and Canary](10-deployment-strategies.md) | L13, exam tasks 6, 12 |
| 11 | [Securing Workloads: RBAC, ServiceAccounts, SecurityContext and Resource Limits](11-security-and-resources.md) | L15, exam tasks 11, 13, 15 |
| 12 | [Keeping Applications Healthy: Probes, Logs and Metrics](12-observability.md) | L17, exam task 5; killer.sh Q6 |
| 13 | [Extending Kubernetes: The API, CRDs and Operators](13-extending-the-api.md) | L14 |
| 14 | [The CKAD Playbook: Speed, Habits and Common Mistakes](14-ckad-exam-playbook.md) | exercise/ckad, exercise/killer-sh |

## How to use these articles

- **Have a cluster ready.** Article 3 shows how to start Minikube with the Calico network plugin. Most examples assume that setup.
- **Type the commands yourself.** Kubernetes is learned with your fingers. The exam is hands-on too.
- **Read the YAML.** Every manifest in this series is small on purpose. Change a field, re-apply it and see what happens.
- **Use the docs.** `kubectl explain` and https://kubernetes.io/docs are allowed in the exam. The articles show you how to find things quickly.

## Conventions

- `k` is an alias for `kubectl` (`alias k=kubectl`). Some killer.sh examples use it.
- `$(minikube ip)` is the IP of your Minikube node.
- Files in `exercise/` are linked where a full manifest is available.
