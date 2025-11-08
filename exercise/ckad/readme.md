
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
