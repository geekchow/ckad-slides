# 5. Configuring Applications: Environment Variables, ConfigMaps and Secrets

> Series: [Working with Kubernetes](README.md) · Previous: [Pods and Deployments](04-pods-and-deployments.md) · Next: [Storage](06-storage.md)

An image should be the same in dev, staging and production. What changes between them is **configuration**: database URLs, feature flags, passwords, config files. Kubernetes keeps that configuration in the cluster as API objects, separate from both the image and the node.

## Plain environment variables

`kubectl create deploy` has no `--env` flag. Add variables afterwards with `kubectl set env`:

```bash
kubectl create deploy mydb --image=mariadb
kubectl logs deploy/mydb          # mariadb refuses to start: no root password set
kubectl set env deploy mydb MARIADB_ROOT_PASSWORD=password
kubectl get pods                  # new Pods roll out and start fine
```

`kubectl run` does accept `--env`, but it creates a bare Pod:

```bash
kubectl run mydb --image=mysql --env="MYSQL_ROOT_PASSWORD=password"
```

Hard-coding values like this is fine for experiments. For *configuration as code* you want the values in their own object.

## ConfigMaps

A **ConfigMap** holds non-sensitive configuration, up to **1 MiB**. It has two uses:

1. **Variables**: key/value pairs that become environment variables.
2. **Files**: each key is a file name and its value is the file content. You mount them as a volume.

### Creating ConfigMaps

```bash
kubectl create cm app-vars --from-literal=COLOR=blue --from-literal=MODE=prod
kubectl create cm app-env  --from-env-file=app.env          # KEY=value lines
kubectl create cm myindex  --from-file=index.html            # key = file name
kubectl create cm myindex  --from-file=home.html=./index.html  # custom key
kubectl create cm nginx-conf --from-file=./conf.d/           # one key per file in the dir
kubectl get cm app-vars -o yaml
```

Add `--dry-run=client -o yaml` to any of these to get a manifest instead of creating the object.

### Using a ConfigMap as variables

The quick way is to import every key as a variable, with an optional prefix:

```bash
kubectl create deploy mydb --image=mariadb --replicas=3
kubectl create cm mydbvars --from-literal=ROOT_PASSWORD=password
kubectl set env deploy/mydb --from=configmap/mydbvars --prefix=MARIADB_
kubectl get deploy mydb -o yaml | grep -A5 env
# -> MARIADB_ROOT_PASSWORD from configMapKeyRef mydbvars/ROOT_PASSWORD
```

In YAML you have two forms:

```yaml
containers:
- name: app
  image: nginx
  env:
  - name: APP_COLOR                # one variable, picked from one key
    valueFrom:
      configMapKeyRef:
        name: app-vars
        key: COLOR
  envFrom:                         # all keys at once
  - configMapRef:
      name: app-vars
    prefix: CFG_                   # optional
```

### Using a ConfigMap as files

Mount the ConfigMap as a volume. This is exam task 3 and killer.sh Q15: serve a custom `index.html` from nginx.

```bash
echo "welcome to the task3 webserver" > index.html
kubectl create configmap task3cm --from-file=index.html
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oregonpod
spec:
  containers:
  - name: oregonpod
    image: nginx
    volumeMounts:
    - name: config-vol
      mountPath: /usr/share/nginx/html   # every key becomes a file here
  volumes:
  - name: config-vol
    configMap:
      name: task3cm
      # optional: pick keys and rename them
      # items:
      # - key: index.html
      #   path: index.html
```

```bash
kubectl apply -f oregonpod.yaml
kubectl exec -it oregonpod -- curl -s localhost
# welcome to the task3 webserver
```

Note that the mount **hides** whatever the image had at that path. To add a single file without hiding the folder, use `subPath`. But a `subPath` mount does not receive ConfigMap updates.

**Updates:** when you change a ConfigMap, mounted files are refreshed after a short delay (up to about a minute). **Environment variables are not refreshed.** They are read once, when the container starts. Restart the Pods with `kubectl rollout restart deploy/<name>`.

## Secrets

A **Secret** works like a ConfigMap, but it is meant for sensitive data. Values are **base64-encoded, not encrypted**:

```bash
kubectl create secret generic dbpw --from-literal=ROOT_PASSWORD=password
kubectl get secret dbpw -o yaml               # data: ROOT_PASSWORD: cGFzc3dvcmQ=
echo cGFzc3dvcmQ= | base64 -d                 # password
kubectl get secret dbpw -o jsonpath='{.data.ROOT_PASSWORD}' | base64 -d
```

Anyone who can read the Secret object can read the value. Secrets still help, because:

- the value is not in your Deployment manifest or Git repository;
- access can be limited separately with RBAC (see [article 11](11-security-and-resources.md));
- clusters can enable encryption at rest in etcd, and external secret stores can plug in.

### Secret types

| `kubectl create secret ...` | Use for |
|-----------------------------|---------|
| `generic` | Passwords, tokens, SSH keys, any file: `--from-literal`, `--from-file` |
| `tls` | A certificate/key pair: `--cert=tls.crt --key=tls.key` (used by Ingress) |
| `docker-registry` | Credentials for pulling from a private registry |

```bash
kubectl create secret generic my-ssh-key --from-file=ssh-privatekey=$HOME/.ssh/id_rsa
kubectl create secret tls my-tls --cert=tls/my.crt --key=tls/my.key
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com --docker-username=me --docker-password=...
```

### Using Secrets as variables

Exam task 1 and killer.sh Q14 ([`q-14-pod.yaml`](../exercise/killer-sh/q-14-pod.yaml)):

```bash
kubectl create secret generic secret1 -n moon --from-literal=user=test --from-literal=pass=pwd
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-14
  namespace: moon
spec:
  containers:
  - name: pod-14
    image: nginx
    env:
    - name: SECRET1_USER
      valueFrom:
        secretKeyRef:
          name: secret1      # must be the Secret's real name
          key: user
    - name: SECRET1_PASS
      valueFrom:
        secretKeyRef:
          name: secret1
          key: pass
```

```bash
kubectl exec -n moon pod-14 -- printenv | grep SECRET
```

> A common slip: the `name` under `secretKeyRef` must match the Secret you created. In task 1 the Secret was `insecret`, so a reference to `my-secret` copied from the docs leaves the Pod stuck in `CreateContainerConfigError`. `kubectl describe pod` shows the reason.

`envFrom` with `secretRef:` imports every key. `kubectl set env deploy/x --from=secret/dbpw` does the same from the command line.

### Using Secrets as files

```yaml
volumes:
- name: creds
  secret:
    secretName: my-ssh-key
    defaultMode: 0400        # owner read-only, good for keys
containers:
- name: app
  volumeMounts:
  - name: creds
    mountPath: /etc/creds
    readOnly: true
```

Like ConfigMaps, mounted Secrets are refreshed when the Secret changes. Environment variables are not.

### Pulling from a private registry

```yaml
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: registry.example.com/team/app:1.0
```

You can also attach `imagePullSecrets` to a ServiceAccount, so every Pod that uses that ServiceAccount gets them.

## Key takeaways

- Keep configuration out of images: ConfigMaps for normal settings, Secrets for sensitive ones.
- Both can be used as **env vars** (`valueFrom`, `envFrom`, `kubectl set env --from`) or **files** (volumes).
- Mounted files are refreshed when the object changes. Env vars need a Pod restart.
- Secrets are base64, not encryption. Protect them with RBAC and encryption at rest.

Next: [persisting data with volumes and PersistentVolumes](06-storage.md).
