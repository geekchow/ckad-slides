# 10. Releasing Safely: Rolling Updates, Blue/Green and Canary

> Series: [Working with Kubernetes](README.md) · Previous: [Helm and Kustomize](09-helm-and-kustomize.md) · Next: [Security and Resources](11-security-and-resources.md)

## DevOps and GitOps on Kubernetes

On Kubernetes, "doing DevOps" mostly means:

- **Configuration as code**: everything is YAML in version control.
- **Reproducibility**: the same manifests build the same environment every time.
- **Continuous availability**: users never see a maintenance window.
- **Zero-downtime updates**.

**GitOps** goes one step further. A Git repository is the single source of truth, and an operator in the cluster (Argo CD, Flux) keeps pulling it and applying changes automatically. You release by merging a pull request.

How you replace version 1 with version 2 is your **deployment strategy**. Kubernetes gives you one built in, and you can build the others from Services, labels and Ingress.

| Strategy | Old and new run together? | Traffic shift | Rollback |
|----------|---------------------------|---------------|----------|
| Recreate | No (downtime) | All at once | Redeploy old |
| Rolling update | Briefly, mixed | Gradual, by Pod count | `kubectl rollout undo` |
| Blue/Green | Yes, side by side | All at once, when *you* switch | Switch back |
| Canary | Yes | A small share first, then more | Remove the canary |

## Rolling updates (built in)

This is the default for Deployments, covered in [article 4](04-pods-and-deployments.md). Tune it with `maxSurge` and `maxUnavailable`, then watch it:

```bash
kubectl set image deploy/updates nginx=nginx:1.27
kubectl rollout status deploy/updates
kubectl rollout undo deploy/updates
```

Rolling updates assume v1 and v2 can serve traffic at the same time, and that a **readinessProbe** tells Kubernetes when a new Pod can take traffic ([article 12](12-observability.md)). Without one, Pods receive traffic as soon as the container starts.

## Blue/Green deployments

Run the new version (**green**) next to the current one (**blue**). Test green privately, then switch **all** traffic in one step. Blue stays around for an instant rollback.

### Service-based blue/green

The Service's selector decides which colour gets traffic:

```bash
kubectl create deploy blue-nginx --image=nginx:1.14 --replicas=3
kubectl expose deploy blue-nginx --port=80 --name=bgnginx

# build green from blue
kubectl get deploy blue-nginx -o yaml > green-nginx.yaml
#   - remove status, uid, resourceVersion, creationTimestamp ...
#   - change the image to nginx:1.27
#   - replace "blue" with "green" everywhere (name, labels, selector)
kubectl create -f green-nginx.yaml
kubectl get pods -l app=green-nginx         # test green directly (port-forward, temp Pod ...)

# switch: point the Service at green
kubectl patch svc bgnginx -p '{"spec":{"selector":{"app":"green-nginx"}}}'
# (the slides delete and re-expose the Service instead; patching keeps the same ClusterIP)

kubectl delete deploy blue-nginx            # once you're confident
```

Service-level switching works at the connection level. Long-lived connections to blue may break when you switch.

### Ingress-based blue/green

Switch the **Ingress backend** instead. The proxy handles this more gracefully for HTTP clients. The course demo uses one Kustomize folder per colour:

```bash
cd ~/ckad/kustomize-bluegreen/blue && kubectl apply -k .
echo "$(minikube ip) myapp.local" | sudo tee -a /etc/hosts
curl http://myapp.local                         # blue

cd ../green && kubectl apply -k .               # green runs, no traffic yet
sed -i 's/blue-svc/green-svc/' myapp-ing.yaml
kubectl apply -f myapp-ing.yaml                 # switch
curl http://myapp.local                         # green
kubectl scale deploy blue-deploy --replicas=0   # keep blue's spec for rollback
```

Ingress only handles HTTP/HTTPS. For other protocols, use the Service approach.

## Canary deployments

Send a **small share** of real users to the new version, watch for errors, then increase step by step. If something goes wrong, remove the canary and only a few users were affected.

### Service-based canary: replica ratios

A Service load-balances evenly across **all** Pods that match its selector. So give old and new Pods a **shared label**, select only on that label, and the replica counts set the split.

This is exam task 12 ([`task12-oldbirds.yaml`](../exercise/ckad/task12-oldbirds.yaml), [`task12-newbirds.yaml`](../exercise/ckad/task12-newbirds.yaml)):

```yaml
# oldbirds (newbirds is identical apart from name/app label/image)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oldbirds
  namespace: birds
  labels:
    app: oldbirds
    type: allbirds
spec:
  replicas: 4
  selector:
    matchLabels:
      app: oldbirds
  template:
    metadata:
      labels:
        app: oldbirds
        type: allbirds        # <- shared label, on the Pod template
    spec:
      containers:
      - name: nginx
        image: nginx:1.17
```

```bash
kubectl create ns birds
kubectl apply -f task12-oldbirds.yaml -f task12-newbirds.yaml   # newbirds: replicas 1
kubectl expose deploy oldbirds -n birds --name=allbirds \
  --selector type=allbirds --port=80 --type=NodePort
kubectl describe svc allbirds -n birds      # 5 endpoints: 4 old + 1 new = 20% canary
```

To move forward, scale newbirds up and oldbirds down. To roll back, delete newbirds.

> The shared label must be in the **Pod template** labels. Labels on the Deployment's own `metadata` do not reach the Pods, and the Service selects Pods.

> **Try it:** the *canary 3:1* scenario in the [interactive labs](interactive/k8s-visual-labs.html#services) shows the traffic share for a shared-label Service.

The limit of this approach: the split is only as fine as your replica count (1 in 100 needs 100 Pods), and it is per connection, not per user.

### Ingress-based canary: weights

With ingress-nginx, a second Ingress for the **same host**, marked as a canary, gets a percentage of requests no matter how many replicas exist:

```bash
kubectl create ing old-version --rule="theapp.info/*=old-version:80"
```

```yaml
# new-ing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: new-version
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  ingressClassName: nginx
  rules:
  - host: theapp.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: new-version
            port:
              number: 80
```

```bash
kubectl apply -f new-ing.yaml
for i in $(seq 20); do curl -s theapp.info; done | sort | uniq -c   # ~16 old, ~4 new
```

ingress-nginx can also route canaries by header or cookie (`canary-by-header`, `canary-by-cookie`). That way testers can opt in on purpose.

### Gateway API canary

With Gateway API, weights are part of the standard API. No annotations needed:

```yaml
backendRefs:
- name: old-version
  port: 80
  weight: 80
- name: new-version
  port: 80
  weight: 20
```

For fully automated canaries (shift weight, check metrics, promote or abort), look at **Argo Rollouts** or **Flagger**.

## Key takeaways

- Rolling updates are built in. Pair them with readiness probes and know `rollout undo`.
- Blue/green: two full environments, one switch (Service selector or Ingress backend), instant rollback.
- Canary: a shared label + replica ratio (Service) or a weight (Ingress annotation / Gateway API `weight`).
- Put shared labels on the **Pod template**, because Services select Pods.

Next: [securing workloads and controlling resources](11-security-and-resources.md).
