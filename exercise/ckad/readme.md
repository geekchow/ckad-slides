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

kubectl get pods -A -l tier=control-plane -o Name    

kubectl get pods -A -l tier=control-plane -o Name | sed 's/^.*\///' > /tmp/pods.txt
```


## Mount volume from ConfigMap
config map store file and mount volume from configmap.

```bash

#kubectl create (no pod)
#kubectl run pod 
echo "welcome phil" > index.html

kubectl create configmap task3cm --from-file=index.html=./index.html

kubectl run oregonpod --image=nginx --dry-run=client -o=yaml > oregonpod.yaml
(Be careful , remove useless fields. May result in errors)

kubectl exec oregon -it -- /bin/bash
curl localhost/index.html

```

## pod with SideCar
when `restartPolicy: Always` of initContainer, it's a sidecar Pod.
Search : > sidecar
```bash
# difficulty:
# convert deploy to pod , 
# container  -wrapping-> pod -wrapping-> deployment 
# compare sidecar-orign to sidecar.yaml
kubectl apply -f sidecar.yaml

kubectl get pod
# keep get the pod status. 
# you will the initContainer keep restarting every 15 sec.
```

## Task 5 using probes
```bash
kubectl run probepod --image=busybox -n probes --dry-run=client -o yaml > probepod.yaml

# edit the probpod.yaml , add sleep infinity.
```

> caution if your cmd `kubectl exec probepod -n probes -- /bin/bash`
get error 

```bash
OCI runtime exec failed: exec failed: unable to start container process: exec: "/bin/bash": stat /bin/bash: no such file or directory: unknown
command terminated with exit code 126
```
That means to bash is installed on this container. You have to re-built the image to install the bash.

## Task 6 create a deployment 

```bash
kubectl create deployment updates --image=nginx:1.17 --replicas=3 --dry-run=client -o yaml > task6.yaml

# get all resources with label: app=updates
kubectl get all -l=app=updates 

# get all history of deployment : updates
kubectl rollout history deployment updates

# check the revision: 1 of deployment updates
kubectl rollout history deployment updates --revision=1

kubectl rollout undo deployment updates --to-revision=1

```

## Task 7 Exposing application

```bash
# expose the updates deployment 
kubectl expose deployment updates --type=NodePort --name=task7svc --port=80 --target-port=80

kubectl get svc task7svc
# check the NodePort
# you can use https://$(minikube ip):{node-port} to access the service.
# Caution, it only works for Linux system.
# For mac you have to use 
minikube svc task7svc 
# To re-mapping the port.

```

1️⃣ If driver = docker (default on macOS nowadays)
👉 The Minikube node runs inside a Docker container, not a full VM.
This means its “Node IP” (192.168.49.2) exists only inside Docker’s network, not directly reachable from your Mac host.

## Task 8 

```bash
kubectl run nevaginx --image=nginx --dry-run=client -o yaml > nevaginx.yaml

kubectl run nevatest --image=busybox --dry-run=client -o yaml > nevatest.yaml

kubectl expose deployment 

kubectl exec nevatest -it -- ls /bin
# let find the shell cmd here 
# it could be /bin/sh /bin/bash
kubectl exec nevatest -it -- /bin/sh

 wget --timeout=1 10.244.0.3 -S -O -
# 10.244.0.3 ip of nevaginx




```