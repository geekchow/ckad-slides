### Understanding the API

- The Kubernetes API provides a way to interact with Kubernetes.
- It provides RESTful endpoints that allow the users to perform operations on the cluster.
- The API uses resources to represent components of the Kubernetes cluster.
- It supports a declarative configuration model, where users define the desired state of the cluster in YAML or JSON manifests.
- It allows for authentication and authorization, using different solutions.
- The API is extensible, which allows for addition of new resources to the Kubernetes environment.

### Understanding the API

- The main API documentation is here:
  https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/
- Replace v1.30 with the current version as shown by kubectl version
- Each API group can have its own version number.
- See here for more information: https://kubernetes.io/docs/reference/using-api/api-overview/
- Use kubectl api-resources for information about resource types.
- Or kubectl api-versions for resource and version information.

### Terminal Commands

```shell
kubectl api-resources | less

kubectl api-versions
```

### Communicating with the API

- The kube-apiserver provides access to the API.
- The kubectl client is the main tool to communicate with the API.
  - It uses the ~/.kube/config file to use the TLS keys for secure communication.
- The kube-proxy can be used as a proxy that uses the .kube/config TLS keys for secure communication.
- This allows utilities like curl to communicate with the API in a non-secured way.

### Client Configuration

- Use kubectl config view to view current kubectl client configuration.
- This command reads the ~/.kube/config file, which contains the following elements:
  - Cluster: certificates and API endpoint needed to contact the kube-apiserver process
  - Users: the TLS certificates that make up the user account
  - Context: the combination of user and cluster, to which a default Namespace is added

### Connecting to the API

- To access the API using curl, start the kube-proxy on the Kubernetes user workstation.
  - kubectl proxy --port=8001&
  - curl http://localhost:8001
- This shows all the available API paths and groups, providing access to all exposed functions.

### Demo: Using curl to Access API Resources

- On the host that runs kubectl: kubectl proxy --port=8001 &
- kubectl run proxypod --image=nginx
- curl http://localhost:8001/version
- curl http://localhost:8001/api/v1/namespaces/default/pods-> shows the Pods
- curl http://localhost:8001/api/v1/namespaces/default/pods/proxypod/shows direct API access to a Pod
- curl -XDELETE http://localhost:8001/api/v1/namespaces/default/pods/proxypod will delete the httpd Pod

### API Deprecations

- With new Kubernetes releases, old API versions may get deprecated.
- If an old version gets deprecated, it will be supported for a minimum of two more Kubernetes releases.
- When you see a deprecation message, make sure to take action and change your YAML manifest files!

### Demo: Dealing with Deprecations

- kubectl create -f redis-deploy.yaml
- kubectl api-versions
- kubectl explain --recursive deploy

## CustomResourceDefinitions

- A CustomResourceDefinition (crd) is an API resource that makes adding your own API resources easy.
- The crds are integrated with the Kubernetes API server and follow the API server update mechanism.
- Using crd is common, as it provides an easy way to add resources without any need to program them.

## API Aggregation

- In API aggregation, the Kubernetes API is extended with additional API servers.
- API aggregation enables a higher level of customization, but API servers need to be programmed.
- The aggregated API server can integrate resources from multiple sources.

## Extending the API

- The Kubernetes API can be extended in different ways,
  - Using the CustomResourceDefinition API resource
  - Using Custom Controllers
  - Using API Aggregation

## Custom Controller

- A controller is a process that watches for changes in the Kubernetes API.
- When changes occur, the controller takes action to ensure that the desired state of the resources is maintained.
- Controllers allow you to automate common tasks within the Kubernetes cluster.
- Custom controllers communicate with the Kubernetes API by using client libraries.

## Understanding CustomResourceDefinitions

- CustomResourceDefinitions allow users to add custom resources to clusters.
- Doing so allows anything to be integrated in a cloud-native environment.
- The crd allows users to add resources in a very easy way
  - The resources are added as extension to the original Kubernetes API server.
  - No programming skills required.
- Adding custom resources only makes sense if you have an application that's using them!

## Creating Custom Resources

- Creating Custom Resources using crds is a two-step procedure.
  - First, you'll need to define the resource, using the CustomResourceDefinition API kind.
  - After defining the resource, it can be added through its own API resource.

## Demo: Creating Custom Resources

- cat crd-object.yaml
- kubectl create -f crd-object.yaml
- kubectl api-resources | grep backup
- cat crd-backup.yaml
- kubectl create -f crd-backup.yaml
- kubectl get backups

## Understanding Operators and Controllers

- Operators are custom applications, based on CustomResourceDefinitions.
- Operators can be seen as a way of packaging, running, and managing applications in Kubernetes.
- Operators are based on Controllers, which are Kubernetes components that continuously operate dynamic systems.
- The Controller loop is the essence of any Controllers.
- The Kubernetes Controller manager runs a reconciliation loop, which continuously observes the current state, compares it to the desired state, and adjusts it when necessary.
- Operators are application-specific Controllers.

## Understanding Operators and Controllers

- Operators can be added to Kubernetes by developing them yourself.
- Operators are also available from community websites.
- A common registry for operators is found at operatorhub.io.
- Many solutions from the Kubernetes ecosystem are provided as operators:
  - Prometheus: a monitoring and alerting solution
  - Tigera: the operator that manages the calico network plugin
  - Jaeger: used for tracing transactions between distributed services

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.stable.example.com
spec:
  group: stable.example.com
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
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    shortNames:
    - bks
    kind: BackUp
```

```yaml

apiVersion: "stable.example.com/v1"
kind: BackUp
metadata:
  name: mybackup
spec:
  backUpType: full
  image: linux-backup-image
  replicas: 5
```