# Lesson 11: Ingress and Gateway API

> Content

  -  11.1 Managing Incoming Traffic
  -  11.2 Ingress Components
  -  11.3 Installing Ecosystem Ingress Controllers
  -  11.4 Using the Minikube Ingress Controller
  -  11.5 Using Ingress
  -  11.6 Configuring Ingress Rules
  -  11.7 Understanding Gateway API
  -  11.8 Configuring Gateway API
  -  11.9 Using Gateway API to Provide Access to Applications

## Understanding Ingress Controllers

![Ingress network flow](./Ingress-network-flow.jpeg)

- Creating Ingress resources without an Ingress controller has no effect.
- Many Ingress controllers exist:
  - nginx: https://kubernetes.github.io/ingress-nginx/
  - haproxy: https://www.haproxy.com/blog/dissecting-the-haproxy-kubernetes-ingress-controller/
  - traefik: https://docs.traefik.io
  - kong: https://konghq.com/solutions/kubernetes-ingress/
  - contour: https://octetz.com/posts/contour-adv-ing-and-delegation

## Understanding Ingress

- Ingress is used to provide external access to internal Kubernetes cluster resources.
- To do so, Ingress uses an external load balancer.
- This load balancer is implemented by the Ingress controller which is running as a Kubernetes application.
- As an API resource, Ingress uses Services to connect to Pods that are used as a service endpoint.
- To access resources in the cluster, the host name resolution (DNS or /etc/hosts) must be configured to resolve to the Ingress load balancer IP.

## Managing Incoming Traffic

- For a long time, Ingress has been the solution to manage incoming traffic.
- Recently, Ingress has gone into a "feature freeze" and will be replaced by Gateway API.
- Currently, Ingress is still in the exam objectives, this is expected to be replaced with Gateway API in the future.
- For that reason, in this lesson you'll learn about both.

## _*Warning: Do not configure Ingress and Gateway API on the same machine!*_

## Understanding Ingress

- Ingress exposes HTTP and HTTPS routes from outside the cluster to Pods within the cluster.
- Traffic routing is controlled by rules defined on the Ingress resource.
- Ingress can be configured to do the following:
  - Give Services externally-reachable URLs
  - Load balance traffic
  - Terminate SSL/TLS
  - Offer name based virtual hosting

## Ingress Controllers

- From the Kubernetes ecosystem, different Ingress controllers are provided.
- As the CNCF doesn't want to favor specific ecosystem projects, vanilla Kubernetes doesn't come with an Ingress controller.
- Kubernetes distributions normally provide one or more supported Ingress controllers.
- Alternatively, Ingress controllers may be installed manually.
- In vanilla Kubernetes, you have to install an Ingress controller or else you can't use it.

## Demo: Installing an Ingress Controller

- On vanilla Kubernetes only!
  - `helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace`
  - `kubectl get pods -n ingress-nginx`


## Minikube Ingress

- Minikube is a Kubernetes distribution and comes with addons to integrate third-party solutions.
- Use **minikube addons list** to show available addons.
- Use **minikube addons enable** to enable a specific addon.

## Demo: Using the Minikube Ingress Addon

- minikube addons list
- minikube addons enable ingress
- kubectl get ns
- kubectl get all -n ingress-nginx

## Demo: Configuring Ingress Rules

- kubectl create deploy nginxsvc --image=nginx --port=80
- kubectl expose deploy nginxsvc
- kubectl create ingress nginxsvc-ingress --rule="/=nginxsvc:80"
- echo "$(minikube ip) nginxsvc.info" >> /etc/hosts
- kubectl describe ing nginxsvc-ingress
- curl nginxsvc.info

## Ingress Rules

Each Ingress Rule contains the following:

- An optional host to be used as a name-based virtualhost. If no host is specified, the rule applies to all inbound HTTP traffic.
- A list of paths (like /testpath). Each path has its own backend. Paths can be exposed as a regular expression.
- The backend, which is a Service resource. It is common to configure a default backend in an Ingress controller for incoming traffic that doesn't match a specific path.


## Demo: Name-based Virtual Hosting

- kubectl create deploy mars --image=nginx
- kubectl create deploy saturn --image=httpd
- kubectl expose deploy mars --port=80
- kubectl expose deploy saturn --port=80
- Add entries to /etc/hosts
  - $(minikube ip) mars.example.com
  - $(minikube ip) saturn.example.com
- kubectl create ingress multihost --rule="mars.example.com/=mars:80" --rule="saturn.example.com/=saturn:80"
- kubectl edit ingress multihost; change pathType to Prefix
- curl saturn.example.com; curl mars.example.com

## Ingress Paths

- The Ingress pathType specifies how to deal with path requests.
- The Exact pathType indicates that an exact match should occur,
  - If the path is set to /foo, and the request is /foo/ there is no match.
- The Prefix pathType indicates that the requested path should start with,
  - If the path is set to /, any requested path will match.
  - If the path is set to /foo, /foo as well as /foo/ will match.

## Ingress Types

- Ingress backed by a single Service: there is one rule that defines access to one backend Service.
  - kubectl create ingress single --rule="/files=fileservice:80"
- Simple fanout: there are two or more rules defining different paths that refer to different Services,
  - kubectl create ingress single --rule="/files=fileservice:80" --rule="/db=dbservice:80"
- Name-based virtual hosting: there are two or more rules that route requests based on the host header,
  - Make sure there is a DNS entry for each host header.
  - kubectl create ingress multihost --rule="my.example.com/files*=fileservice:80" --rule="my.example.org/data*=dataservice:80"



## Gateway API

![apigateway network flow](./apigateway-network-flow.jpeg)

- In current Kubernetes, Ingress is in feature freeze and no longer developed.
- The replacement is Gateway API.
- Gateway API adds more advanced features to manage incoming traffic:
  - Advanced traffic management
  - More options that are integrated in the API resources
- Gateway API may find its way into future versions of the CKAD exam.


## Gateway API Resources

- Gateway API uses specific API resources which are provided as CRDs:
  - GatewayClass: represents the Gateway Controller
  - Gateway: defines an instance of traffic handling infrastructure
  - HTTPRoute: defines how traffic is routed to one or more Services
- To work with Gateway API, a Gateway API Controller needs to be installed.
- Without this controller, there's nothing actually handling the incoming traffic!

## Gateway API Controller

- Different Gateway API controllers are provided by the ecosystem.
- In this class, we'll use the Nginx Gateway Fabric, which is easily installed with helm.
- Before installing the controller, you must (currently) install the custom resources!

## Resources: GatewayClass

- The GatewayClass resource represents the physical Gateway Controller.
- It uses spec.controllerName to connect to a specific Gateway Controller.
- It has no further configuration, the real configuration is done on the Gateway resource.

## Understanding the Procedure

- First, you'll need to make sure the required custom resources are available.
- Next, install a community Gateway API controller.
- Verify the community Gateway Controller is ready to accept incoming requests.
- Create the Kubernetes application you want to provide access to.
- Configure GatewayClass, Gateway, and HTTPRoute.
- Test by accessing the Service that exposes the community controller.

## Demo: Using Gateway API

- Install CRD's: `kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml`
- Install nginx-gateway-fabric controller: `helm install ngf-oci://ghcr.io/nginxinc/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway`
- Verify: `kubectl get pods, svc -n nginx-gateway`
- At this point, you have a GatewayController. Use `kubectl get gc` and notice the name (nginx).
- Make the gwc service accessible as a NodePort: `kubectl edit -n nginx-gateway svc ngf-nginx-gateway-fabric` and set type to NodePort.

## Demo: Using Gateway API

- Create Kubernetes resources
  - `kubectl create deploy nginxwg --image=nginx --replicas=3`
  - `kubectl expose deploy nginxwg --port=80`
- Open http-routing.yaml from course Git repository and verify
  - gatewayClassname: nginx
  - backendRefs.name: nginxwg
- `kubectl apply -f http-routing.yaml`
- Let's do a first test, using port-forwarding
  - `sudo sh -c "echo 127.0.0.1 whatever.com >> /etc/hosts"`
  - `kubectl -n nginx-gateway port-forward ngf-nginx-gateway-[Tab] 8080:8080 443:443`
  - `curl whatever.com:8080`


> Reference cmds

```shell
# Demo: Using Gateway API
kubectl get pods, svc, -n nginx-gateway

kubectl get svc -n nginx-gateway

kubectl get svc 

kubctl describe svc nginxsvc

kubectl get pods --show-labels
```