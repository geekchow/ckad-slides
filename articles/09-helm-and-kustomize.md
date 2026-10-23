# 9. Packaging Applications: Helm and Kustomize

> Series: [Working with Kubernetes](README.md) · Previous: [Ingress and Gateway API](08-ingress-and-gateway-api.md) · Next: [Deployment Strategies](10-deployment-strategies.md)

A real application is not one YAML file. It is a Deployment, a Service, a ConfigMap, a Secret, an Ingress, maybe a PVC and RBAC objects, and every environment needs slightly different values. Kubernetes has two common answers:

| | Helm | Kustomize |
|-|------|-----------|
| Idea | **Templates** + values → manifests | Plain YAML + **patches/overlays** |
| Distribution | Charts in repositories (like apt/yum for Kubernetes) | Folders in Git |
| Tracks installs | Yes: releases, history, rollback | No: it just renders and applies |
| Built in to kubectl | No (separate `helm` binary) | Yes (`kubectl apply -k`) |
| Best for | Installing third-party software; shipping your app to others | Tweaking your own (or vendored) YAML per environment |

## Helm

### Vocabulary

- **Chart**: a package of templated manifests plus a default `values.yaml`.
- **Repository**: a place that hosts charts. Helm has **no default repo**. Find charts on https://artifacthub.io.
- **Release**: one installed instance of a chart, with a name you choose. You can install the same chart many times under different release names.

### Setup and searching

```bash
source <(helm completion bash)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update                       # refresh the index, like apt update
helm repo list
helm search repo nginx                 # search repos you have added
helm search repo nginx --versions      # every available chart version
helm search hub wordpress              # search Artifact Hub
```

> Bitnami changed its free catalog in 2025, and many images moved or became limited. If a Bitnami chart's Pods sit in `ImagePullBackOff`, check the chart's notes on Artifact Hub, or practise with another chart. The Helm commands are the same.

### Installing, inspecting and removing

```bash
helm install mysql1 bitnami/mysql                 # release name + chart
helm install bitnami/mysql --generate-name        # let Helm pick a name
helm install web bitnami/nginx -n web --create-namespace --version 15.0.0

helm list                     # releases in the current namespace
helm list -A                  # all namespaces
helm status mysql1
kubectl get all -l app.kubernetes.io/instance=mysql1

helm uninstall mysql1         # removes everything the release created
```

Before installing, read what you are getting:

```bash
helm show chart bitnami/mysql      # metadata
helm show readme bitnami/mysql
helm show values bitnami/nginx     # every configurable value (long!)
helm template web bitnami/nginx | less   # render manifests locally without installing
```

### Customising with values

A chart's templates get their settings from `values.yaml`. Override them in one of two ways:

```bash
# 1. A values file (preferred: reviewable, repeatable)
helm show values bitnami/nginx | grep -E 'replicaCount|commonLabels'
cat > values.yaml <<'EOF'
replicaCount: 3
commonLabels:
  type: helmapp
EOF
helm install mynginx bitnami/nginx --values values.yaml

# 2. Individual --set flags (handy for one-offs)
helm install mynginx bitnami/nginx --set replicaCount=3
```

```bash
helm get values mynginx          # only your overrides
helm get values mynginx --all    # the full merged set
helm get manifest mynginx        # the YAML that was applied
```

### Upgrades and rollbacks

```bash
helm install bitginx bitnami/nginx --set ingress.enabled=false
helm upgrade bitginx bitnami/nginx --set ingress.enabled=true --reuse-values
helm history bitginx              # revision list
helm rollback bitginx 1           # back to revision 1
helm upgrade --install app ./mychart -f values.yaml   # idempotent "install or upgrade"
```

> On `helm upgrade`, values you passed before are **not** kept automatically. Pass your `-f values.yaml` again, or use `--reuse-values`. Forgetting this quietly resets settings to the chart defaults.

## Kustomize

Kustomize starts from plain, valid Kubernetes YAML and applies **transformations** described in a `kustomization.yaml`. Nothing is templated, so the base files stay usable on their own. This is ideal for YAML you don't control, such as upstream manifests that change with each release.

### A basic kustomization

```text
myapp/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

```yaml
# kustomization.yaml
resources:            # which files to include
- deployment.yaml
- service.yaml
namePrefix: test-     # every object name gets this prefix
namespace: testing    # every object goes in this namespace
labels:               # added to every object (and to selectors)
- pairs:
    environment: testing
  includeSelectors: true
```

```bash
kubectl kustomize .          # render and print, don't apply
kubectl apply -k .           # render and apply
kubectl get all -n testing   # test-myapp deployment, test-myapp service ...
kubectl delete -k .          # remove everything it created
```

Older examples use `commonLabels: {environment: testing}`. It still works but is deprecated in favour of `labels`.

### Bases and overlays

The real strength of Kustomize is one **base** with small **overlays** per environment:

```text
base/
  deployment.yaml
  service.yaml
  kustomization.yaml        # resources: [deployment.yaml, service.yaml]
overlays/
  dev/kustomization.yaml
  staging/kustomization.yaml
  prod/kustomization.yaml
```

```yaml
# overlays/prod/kustomization.yaml
resources:
- ../../base
namePrefix: prod-
namespace: production
labels:
- pairs:
    environment: production
replicas:
- name: myapp
  count: 5
images:
- name: nginx
  newTag: "1.27"
patches:
- path: resources-patch.yaml     # strategic-merge patch with prod CPU/memory limits
```

```bash
kubectl apply -k overlays/dev
kubectl apply -k overlays/prod
```

Kustomize can also generate ConfigMaps and Secrets (`configMapGenerator`, `secretGenerator`). It adds a content hash to their names, so changing the config automatically rolls out the Deployments that use it. The blue/green demo in [article 10](10-deployment-strategies.md) uses a Kustomize folder for each colour.

## Which one should I use?

- To **install someone else's software** (ingress controllers, databases, monitoring), use **Helm**.
- To manage **your own manifests across environments**, especially with GitOps tools like Argo CD or Flux, use **Kustomize**.
- They also combine: render a Helm chart and post-process it with Kustomize.

## Key takeaways

- Helm: `repo add` → `repo update` → `show values` → `install -f values.yaml` → `list`/`status` → `upgrade`/`rollback`/`uninstall`.
- Pass your values again on every `helm upgrade`.
- Kustomize: `resources` + transformers (`namePrefix`, `namespace`, `labels`, `images`, `patches`), applied with `kubectl apply -k`.
- Use bases and overlays for dev/staging/prod without copying YAML.

Next: [releasing safely with rolling, blue/green and canary deployments](10-deployment-strategies.md).
