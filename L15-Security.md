# Lesson 15: Security

## 15.1 Authentication and Authorization

### Understanding Authentication
- Authentication is about where Kubernetes users come from.
- In vanilla Kubernetes and Minikube, a local Kubernetes admin account is used for authentication.
- In more advanced setups, you can create your own user accounts (covered in CKA).
- The kubectl config specifies to which cluster to authenticate.
  - Use `kubectl config view` to see current settings.
- The config is read from `~/.kube/config`.

### Understanding Authorization
- Authorization is what these users can do.
- Behind authorization, there is Role Based Access Control (RBAC) to take care of the different options.
- Use `kubectl auth can-i ...` (like `kubectl auth can-i get pods`) to find out what you can do.
- Use `kubectl auth can-i get pods --as=system:serviceaccount:bellevue:viewer -n bellevue`

### Demo: Showing Current Authorizations
- `kubectl auth can-i get pods`
- `kubectl auth can-i get pods --as bob@example.com`

## 15.2 API Access and ServiceAccounts

### Understanding ServiceAccounts
- All actions in a Kubernetes Cluster need to be authenticated and authorized.
- ServiceAccounts are used for basic authentication from within the Kubernetes cluster.
- RBAC is used to connect a ServiceAccount to a specific Role.
- Every Pod uses the default ServiceAccount to contact the API server.
- This default ServiceAccount allows a resource to get information from the API server, but not much else.
- Each ServiceAccount uses a Secret to automount API credentials.

### Custom ServiceAccount Use Case
- Most Pods do fine with the default ServiceAccount.
- If a Pod needs access to resources in the cluster, a custom ServiceAccount that uses a RoleBinding to connect to a specific Role is needed.
- For instance, this is needed for network plugins, monitoring software and other additional components installed in Kubernetes.

### Demo: Exploring ServiceAccounts
- `kubectl describe pod anypod #look for the ServiceAccount`
- `kubectl get sa -n default`
- `kubectl describe pod coredns -n kube-system #look for ServiceAccount`
- `kubectl get sa -n kube-system`

## 15.3 Role Based Access Control (RBAC)

### Demo: Configuring RBAC
- `kubectl create ns bellevue`
- `kubectl create role viewer --verb=get --verb=list --verb=watch --resource=pods -n bellevue`
- `kubectl create sa viewer -n bellevue`
- `kubectl create rolebinding viewer --serviceaccount=bellevue:viewer --role=viewer -n bellevue`
- `kubectl create deploy viewginx --image=nginx --replicas=3 -n bellevue`
- `kubectl set serviceaccount deployment viewginx viewer -n bellevue`
- `kubectl auth can-i get pods --as=system:serviceaccount:bellevue:viewer -n bellevue`

### Demo: Exploring RBAC Usage
- `kubectl describe serviceaccount coredns -n kube-system`
- `kubectl describe clusterrolebinding system:coredns`
- `kubectl describe clusterrole system:coredns`


## 15.3 SecurityContext

### Using SecurityContext

- Notice that SecurityContext can be applied to Pods as well as containers.

- When SecurityContext prevents a Pod from running successfully, use kubectl describe to get additional information from the events.

- Expect Pods that fail because of SecurityContext restrictions to show a status of Pending or failed.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
  fsGroup: 2000
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
  volumeMounts:
  - name: sec-ctx-vol
    mountPath: /data/demo
  securityContext:
    allowPrivilegeEscalation: false
```

## Understanding Resources

- Resource requests can be set for containers in a Pod to ensure that the Pod is only scheduled on cluster nodes that meet the resource requests.
  - Use pod.spec.containers.resources.requests to set

- Resource limits can be set for Pods to maximize the use of system resources.
  - Use pod.spec.containers.resources.limits to define

- Quota are restrictions that can be set on a Namespace to maximize the availability of resources within that Namespace.

- To set resource requests and limits you don't have to use Quota.

- If a Namespace has Quota, all Pods running in that Namespace must have resources set.

## Understanding Resource Limitations

- Memory as well as CPU limits can be used.

- CPU limits are expressed in millicore or millicpu, 1/1000 of a CPU core.
  - So, 500 millicore is 0.5 CPU

- When being scheduled, the kube-scheduler ensures that the node running the Pods has all requested resources available.

- If a Pod with resource limits cannot be scheduled, it will show a status of Pending.

- Use kubectl set resources … to apply resource limits to running applications in deployments.

## Understanding Quota

- Quota are restrictions that are applied to Namespaces.

- If Quota are set on a Namespace, applications started in that Namespace must have resource requests and limits set.

- Use kubectl create quota ... -n mynamespace to apply Quota

## Demo: Using Resource Requests and Limits

- kubectl create -f frontend-resources.yaml
- kubectl get pods
- kubectl describe pod frontend
- kubectl delete -f frontend-resources.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```