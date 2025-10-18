# Lesson 13: Deploying Applications the DevOps Way

13.1 DevOps and GitOps
13.2 Blue / Green Deployments
13.3 Canary Deployments


## Blue/Green Deployments

- Blue/Green Deployments are a way of working to ensure that applications can safely be upgraded to a new version.
- The essence of Blue/Green Deployments is that the new version of the application can already be tested while the old version is still being used.
- There are many different ways in which Blue/Green Deployments can be implemented:
  - Using Kubernetes Services
  - Using Kubernetes Ingress
  - Using advanced resources such as Istio

## DevOps and GitOps

- DevOps is a lot. While working with Kubernetes, DevOps focuses on a few items:
  - Configuration as Code (YAML files)
  - A methodology that can easily be reproduced
  - Continuous access to applications
  - Zero-downtime application updates
- GitOps brings a higher level of automation to DevOps:
  - YAML files are provided by a Git repository.
  - A GitOps operator picks up changes and applies them to the Kubernetes cluster in an automated way.

## Service versus Ingress Blue/Green

- Service-based Blue/Green is happening at a lower level.
- Existing client connections may get disturbed.
- For better support of existing connections, Ingress can be used.
- Notice that Ingress only works for HTTP/HTTPS-based applications.

## Demo: Ingress Based Blue/Green Deployments

### Part 1: Creating the Blue application
- cd ~/ckad/kustomize-bluegreen/blue
- cat *
- kubectl apply -k .
- kubectl get deploy,pods, svc,cm,ing

### Part 2: Testing application access
- sudo sh -c "echo $(minikube ip)   myapp.local >> /etc/hosts"
- In browser: http://myapp.local

### Part 3: Creating the Green application
- cd ~/ckad/kustomize-bluegreen/green
- cat*
- kubectl apply -k .

### Part 4: Making the Switch
- sed -i -e's/blue-svc/green-svc/' myapp-ing.yaml
- kubectl apply -f myapp-ing.yaml
- curl http://myapp.local
- kubectl scale deploy blue-deploy --replicas=0

## Demo: Service Based Blue/Green Deployments

- kubectl create deploy blue-nginx --image=nginx:1.14 --replicas=3
- kubectl expose deploy blue-nginx --port=80 --name=bgnginx
- kubectl get deploy blue-nginx -o yaml > green-nginx.yaml
  - Clean up dynamic generated stuff
  - Change Image version
  - Change "blue" to "green" throughout
- kubectl create -f green-nginx.yaml
- kubectl get pods
- kubectl delete svc bgnginx; kubectl expose deploy green-nginx --port=80 --name=bgnginx
- kubectl delete deploy blue-nginx

## Understanding Canary Deployments

- A Canary Deployment upgrade strategy will expose a new version of the application to a limited number of users before completing the migration to the new version.
- This allows user exposure with a minimized risk.
- If things don't work out well, it's easy to revert to the previous situation by just removing the new application instance(s).

## Service versus Ingress Canary

- Canary Deployments can be configured based on Services or Ingress.
- Using Ingress is preferred as the application picks up the change without reconnecting.
- Service-based Canary Deployments are configured to use a common selector label on the old as well as the new applications.
- Ingress-based Canary Deployments are using two Ingress resources pointing to the same Ingress virtual host.
- Canary Deployment solutions are also offered by alternative ecosystem solutions.

## Demo: Ingress Canary Deployments

- echo new-version > index.html
- kubectl create cm new-version --from-file=index.html
- echo old-version > index.html
- kubectl create cm old-version --from-file=index.html
- kubectl apply -f canary.yaml
- kubectl expose deploy old-nginx --port=80 --type=NodePort
- sed -i -e 's/old/new/' canary.yaml
- kubectl apply -f canary.yaml
- kubectl expose deploy new-nginx --port=80 --type=NodePort
- kubectl get deploy,pods, svc

## Demo: Ingress Canary Deployments

- sudo sh -c "$(minikube ip) theapp.info >> /etc/hosts"
- kubectl create ing old-version --rule="theapp.info/=old-version:80"
- curl theapp.info # all traffic goes to old-version only
- cat new-ing.yaml # notice the annotations
- kubectl apply -f new-ing.yaml
- curl theapp.info # repeat at least 15 times

## Demo: Service-based Canary Deployments

- echo "old nginx" > index.html
- kubectl create cm old --from-file=index.html
- echo "new nginx" > index.html
- kubectl create new --from-file-index.html
- cat canary.yaml
- kubectl apply -f canary.yaml
- sed -i -e 's/old/new/' canary.yaml
- sed -i -e 's/replicas: 3/replicas: 1/' canary.yaml
- kubectl apply -f canary.yaml

## Demo: Service-based Canary Deployments

- kubectl expose deploy old-nginx --name=theapp --port=80 --selector type=canary --type=NodePort
- kubectl get svc
- curl $(minikube ip):<nodeport> # repeat at least 10 times