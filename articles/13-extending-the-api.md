# 13. Extending Kubernetes: The API, CRDs and Operators

> Series: [Working with Kubernetes](README.md) · Previous: [Observability](12-observability.md) · Next: [The CKAD Playbook](14-ckad-exam-playbook.md)

## The Kubernetes API

Everything you have done in this series (`kubectl run`, `apply`, `get`, `delete`) was an HTTP call to the **kube-apiserver**. The API is:

- **RESTful**: resources live at URLs, and verbs map to HTTP methods (GET, POST, PUT/PATCH, DELETE).
- **Declarative**: you send desired state as YAML/JSON, and controllers reconcile it.
- **Versioned per group**: `v1` (core: pods, services, configmaps), `apps/v1` (deployments), `networking.k8s.io/v1` (ingress, networkpolicy) and so on.
- **Secured**: TLS plus authentication and authorization ([article 11](11-security-and-resources.md)).
- **Extensible**: you can add your own resource types.

```bash
kubectl version                       # client and server version
kubectl api-versions                  # every group/version
kubectl api-resources -o wide         # every kind, its group, and the verbs it supports
```

The full reference is at `https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.XX/`. Replace `XX` with your server's minor version.

### Talking to the API directly

`kubectl proxy` opens a local, already-authenticated tunnel to the API server, using your kubeconfig credentials. Then any HTTP client works:

```bash
kubectl proxy --port=8001 &
kubectl run proxypod --image=nginx

curl http://localhost:8001/version
curl http://localhost:8001/api/v1/namespaces/default/pods                 # list Pods
curl http://localhost:8001/api/v1/namespaces/default/pods/proxypod        # one Pod
curl -X DELETE http://localhost:8001/api/v1/namespaces/default/pods/proxypod
curl http://localhost:8001/apis/apps/v1/namespaces/default/deployments    # named groups live under /apis
```

> `kubectl proxy` is not `kube-proxy`. `kubectl proxy` is a client-side helper on your machine. `kube-proxy` is the node component that implements Services.

To see the exact calls kubectl makes, add `-v=6` (URLs) or `-v=8` (request and response bodies):

```bash
kubectl get pods -v=6
```

### API deprecations

API versions move from `v1alpha1` to `v1beta1` to `v1`, and old versions are eventually **removed**. A deprecated beta API is served for a set number of releases after deprecation (at least 9 months or 3 releases), then it is gone and manifests that use it are rejected. Well-known examples are `extensions/v1beta1` Deployments and Ingresses.

When you see a deprecation warning, update your manifests:

```bash
kubectl apply -f redis-deploy.yaml
# Warning: ... is deprecated in v1.xx+, unavailable in v1.yy+; use <group>/<version>
kubectl explain deployment --api-version=apps/v1
kubectl explain deploy --recursive | less      # check the field structure for the current version
```

The usual fixes when you migrate an old Deployment to `apps/v1`: change `apiVersion`, and add the now-required `spec.selector.matchLabels`.

## Three ways to extend the API

| Method | Effort | What you get |
|--------|--------|--------------|
| **CustomResourceDefinition (CRD)** | Write some YAML | New resource types stored in etcd and served by the normal API server |
| **Custom controller / operator** | Write code (Go, Python, …) using client libraries | Behavior: something that *acts* on those resources |
| **API aggregation** | Write and run your own API server | Full control. The main API server proxies a path to your server (metrics-server works this way) |

CRDs and controllers usually come together: the CRD defines the *data*, the controller provides the *behavior*.

### Creating a CRD

Defining a custom resource takes two steps: first define the type, then create objects of that type.

**Step 1: define the type** (`crd-object.yaml`):

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.stable.example.com     # <plural>.<group>
spec:
  group: stable.example.com
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    kind: BackUp
    shortNames: [bks]
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              backupType:
                type: string
              image:
                type: string
              replicas:
                type: integer
```

```bash
kubectl create -f crd-object.yaml
kubectl api-resources | grep backup    # backups  bks  stable.example.com/v1  true  BackUp
kubectl explain backup.spec            # your schema is documented automatically
```

**Step 2: create an instance** (`crd-backup.yaml`):

```yaml
apiVersion: stable.example.com/v1
kind: BackUp
metadata:
  name: mybackup
spec:
  backupType: full          # field names must match the schema exactly
  image: linux-backup-image
  replicas: 5
```

```bash
kubectl create -f crd-backup.yaml
kubectl get backups            # or: kubectl get bks
kubectl get bks mybackup -o yaml
```

> Field names are case-sensitive. A field the schema doesn't know (for example `backUpType` vs `backupType`) is **pruned**, meaning silently dropped, or rejected with strict field validation. Check with `kubectl get -o yaml` after creating.

A CRD on its own only **stores** data. Nothing makes backups happen until a controller watches `BackUp` objects and acts on them. Adding custom resources only makes sense if something uses them.

## Controllers and operators

A **controller** is a loop: *observe* the current state, *compare* it with the desired state, *act* to close the gap, repeat. The kube-controller-manager runs the built-in ones (Deployment, ReplicaSet, Job, Endpoints …). You saw them at work in every article.

An **operator** is a controller for one specific application, usually with its own CRDs. It encodes what a human operator knows: "to upgrade this database, do A, then B, then wait for C". With an operator you install a `PostgresCluster` or `Prometheus` object and the operator handles the StatefulSets, Services, backups and failover.

Where to find operators:

- **OperatorHub.io**, plus Helm charts on Artifact Hub;
- examples: the **Prometheus Operator** (monitoring), **Tigera Operator** (Calico networking), **Jaeger** (tracing), **cert-manager** (TLS certificates), database operators (CloudNativePG, Strimzi for Kafka).

You have already used CRD-based extensions in this series: **Gateway API** ([article 8](08-ingress-and-gateway-api.md)) is a set of CRDs plus a controller (NGINX Gateway Fabric).

```bash
kubectl get crd                         # what has been added to this cluster
kubectl get crd | grep gateway
```

## Key takeaways

- kubectl is an HTTP client. `kubectl proxy` + `curl` and `-v=6` show the REST API underneath.
- API versions get deprecated and removed. Update `apiVersion` (and required fields) when warned.
- CRD = new resource type (data). Controller/operator = the behavior that acts on it. Aggregation = your own API server.
- Operators package operational know-how. Many ecosystem tools ship as operators.

Next: [the CKAD playbook](14-ckad-exam-playbook.md).
