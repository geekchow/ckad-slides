# killer.sh exercise



## Question 6

![question 6](<q-06.png>)

```bash
k run tmp --image=busybox:1.31.0 --restart=Never -it /bin/sh

k run pod6 --image=busybox:1.31.0 --command "touch /tmp/ready" --dry-run=client -o yaml > q-06-pod.yaml

k apply -f q-06-pod.yaml
```

## Question 8 

```bash
# check all revisions of a deployment
kubectl rollout history deployment dp1

# recover to previous revision
kubectl rollout undo deployment dp1
```

![Question 08](q-08.png)

![roll out history](q-08-rollout-history.png)

## Question 9

![question 09](q-09.png)

```shell
# delete pods immediately 
kubectl delete pod pod1 --foce --grace-period=0
```

## Question 10

![Question 10](q-10.png)

![Answer](q-10-answer.png)

![Run tmp pod](q-10-tmp-pod.png)

## Question 11

Docker image operations

![Question 11](q-11.png)

![Q11-part one](q-11-build-docker-image.png)

![Q11-part two](q-11-podman-image.png)

![q 11 podman check](q-11-podman-check.png)

![q 11 podman logs](q-11-podman-logs.png)

## Question 12 

![Question 12](q-12.png)

![Persistent Volume](q-12-pv.png)

![Persistent Volume Claims](q-12-pvc.png)

![create dp from cmd](q-12-dp-cmd.png)

![deploy use pvc](q-12-dp-yaml.png)

## Question 13

![Question 13](q-13.png)

```bash
k create ns moon
# create storage class
# search storageclass
k apply -f q-13-sc.yaml

k get sc -n moon

# search 'persistentvolumeclaim'
k apply -f q-13-pvc.yaml

k get pvc -n moon

```

## Question 14

![question 14](q-14.png)

```bash
k create secret generic -h | less

k create secret -n moon generic secret1 --from-literal=user=test  --from-literal=pass=pwd

k get secrets -n moon secret1 -o yaml 

# search 'secret'
k apply -f q-14-pod.yaml

k get pods -n moon 

k -n moon describe pod-14

k exec -it -n moon pod-14 -- /bin/sh

printenv | grep SECRET
```