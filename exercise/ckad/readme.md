
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
Search : > sidecar
```bash
kubectl apply -f sidecar.yaml
```