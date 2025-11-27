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



