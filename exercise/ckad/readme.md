# Exam



## Task1 refer secret from pod
```shell
kubectl create secret -h | less

kubectl create secret generic insecret --from-literal=COLOR=blue -n indiana

kubectl run inpod --image=nginx -n indiana --dry-run=client -o yaml  > taks1-inpod.yaml

# append env var refer secret part.
# verify the syntax before apply
kubectl apply --dry-run=client -f task1-pod.yaml

kubectl explain deployment

kubectl explain deployment.spec

kubectl explain pod

kubectl explain pod.spec.containers

kubectl explain secret

kubectl apply -f taks1-inpod.yaml
```

[complete code of pod](./task1-inpod.yaml)

## Task2 find pods by label

```shell
kubectl get pods -h | grep label -C 5


kubectl get pods -A -l tier=control-plane

kubectl get pods -A --selector tier=control-plane

kubectl get pods -A -l tier=control-plane -o Name    

kubectl get pods -A -l tier=control-plane -o Name | sed 's/^.*\///' > /tmp/pods.txt
```

## Task 3 Creating a configmap 

```bash
# 1. create a cm from a file 
echo "welcome to the task3 webserver" > index.html

# find the cmd to creat cm from file
kubectl create cm -h | grep file -C 5

kubectl create configmap  task3cm --from-file=index.html=./index.html 

# 2. create a pod runing nginx 
# search "configmap" in kubernetes.io find the yaml and edit in task3-pod.yaml

kubectl apply -f task3-pod.yaml

kubectl exec -it oregonpod -- /bin/sh

curl localhost
#verify the content.
```

## Task 4 SideCar 

```bash
#search keyword "sidecar" in kubernetes.io 
# you will find the deployment for sidecar , you have to modify it to pod 
kubectl apply -f task4-pod.yaml --dry-run=client

kubectl apply -f task4-pod.yaml

kubectl get pod 
# you will see the sidepod has two containers, one of the two keep restarting every 15 sec.

```

## Task 5 probe 

run a pod , the health check is to check the k8s api healthz endpoint.
```bash
minikube ssh 
# get ip of the minikbue host 
hostname -i 
# curl k8s api healthz endpoint.
curl -i https://192.168.49.2:8443/readyz


kubectl create ns probes

# search health from kubernetes.io find a doc for health check
kubectl apply -f task5-pod.yaml --dry-run=client

kubectl apply -f task5-pod.yaml 

kubectl get pods -n probes

kubectl describe pod probepod -n probes
```


## Task 6 Create a deployment 

```shell
# search maxSurge in kubernetes.io
# find the deployment yaml
# set nginx:1.17
kubectl apply -f task6-dp.yaml --dry-run=client

kubectl get all -l type=prod
# see three pods and a replicasets

kubectl edit deployment updates
# update nginx:latest

kubectl get all -l type=prod
# see three pods and two replicasets

#rollback to last deployment
kubectl rollout -h | less
kubectl rollout undo deployment/updates

```

## Task 7 Exposing Applications

```bash
# check help of expose cmd
kubectl expose -h | less

kubectl expose deployment updates --port=80 --target-port=80 --name=task7svc

kubectl get svc
# find the service task7svc and its ip

minikube ssh 

curl 10.101.162.103
# get the defaul enginx response.


```

## Task 8 NetworkPolicy


```bash

minikube start --network-plugin=cni --cni=calico
# by default minikube doesn't enable networkplicy 

kubectl run nevaginx --image=nginx 

kubectl label pods nevaginx type=webapp

kubectl expose pod nevaginx --port=80 --target-port=80

# search networkpolicy

kubectl apply -f task8-np.yaml 

kubectl run nevatest --image=busybox -- sleep infinity

kubectl exec -it nevatest -- wget --spider --timeout=1 nevaginx
# it will failed, since the nevatest pod doesn't have label type=tester

kubectl label pods nevatest type=tester

# remove label type and retry .
kubectl label pods nevatest type-

```

## Task 9 Persistent Volumn

```bash

minikube ssh 

echo "welcome to store pod" > /home/docker/webapp/index.html

# search Persistent Volumes
kubectl apply -f task9-pv.yaml

kubectl expose pod storepod --type=NodePort

# edit the svc storepod , modify the node-port to 32032
kubectl edit svc storepod 

kubectl get all -l=type=store   
# get the ip the storepod service 

minikube ssh 
curl 10.100.119.18:32032

#caution won't work on mac, since the network on mac of docker is isolated from the host.
curl $(minikube ip):32032 

```

## Task 10 Use HELM 

```bash
brew install helm

helm -h | grep repo -C 5

helm repo add bitnami https://charts.bitnami.com/bitnami

helm install mysql1 bitnami/mysql 
```


## Task 11 managing resource restriction 
```bash
kubectl create ns nabraska 

# search memory
kubectl apply -f task10-mem.yaml 


# enable metrics-server addon
minikube addons enable metrics-server    
# verify if metrics-server pods are up 
kubectl get deployment metrics-server -n kube-system

# need to enable  metrics-server
kubectl top pod snowdeploy  -n nabraska

# or use imperative cmd
kubectl create deploy snowdeploy --image=nginx -n nabraska

kubectl set resources -h | less
#  kubectl set resources deployment nginx --limits=cpu=200m,memory=512Mi --requests=cpu=100m,memory=256Mi

kubectl set resources deployment snowdeploy --limits=memory=128Mi --requests=memory=64Mi

```

## Task 12 Create Cannary Deployment 

```shell
kubectl create namespace birds

kubectl create deploy oldbirds --image=nginx:1.17 -n birds --dry-run=client -o yaml > task12-oldbirds-dp.yaml
# caution , you need to attach label type=allbirds to both deploy & pod level.
kubectl create deploy newbirds --image=nginx:1.17 -n birds --dry-run=client -o yaml > task12-newbirds-dp.yaml


kubectl expose deployment -n birds  oldbirds --selector type=allbirds --type=NodePort --port 80

kubectl edit service -n birds oldbirds 
# update the nodePort field to 32323
kubectl describe -n birds service oldbirds
# will see the pords 80:32323

# but minikube on mac, its network is isolated. 
# so we have to go to minikube host
minikube ssh

curl localhost:32323
#or use ip of the service
curl 10.110.227.250

kubectl get pods -n birds --show-labels


kubectl describe -n birds service oldbirds

# to let 20% to new , 80 % to old , we can juset set replicas of old deploy to 4
kubectl scale -n birds deployment oldbirds --replicas=4 

kubectl describe -n birds service oldbirds
# you will see ips of oldbirds are attached to the service.
# Endpoints:                10.244.0.48:80,10.244.0.49:80,10.244.0.51:80 + 1 more...

```

## Task 13 Security Context 

```bash
#search securitycontext in kubernetes.io

vi task13-security.yaml

kubectl apply -f task13-security.yaml --dry-run=client

kubectl apply -f task13-security.yaml
```

## Task 14 Using Docker file 

```bash
docker build --help | less

docker build -f task14-dockerfile -t myapp:1.0 .

docker save --help | less

docker save myapp:1.0 -o /tmp/myapp.tar
```

## Task 15 User Service Account 

```bash
kubectl create ns oklahoma

kubectl create sa secure -n oklahoma

kubectl run pod securepod -n oklahoma --image=nginx:latest --dry-run=client -o yaml

kubectl explain pod.spec | less
 
#edit the task15.yaml file append serviceAccountName property

```
