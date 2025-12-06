

## Question 6

```bash
k run pod6 --dry-run=client  --image=busybox:1.31.0 -o yaml -- "touch /tmp/ready && sleep 1d" > 6.yaml

k apply -f 6.yaml
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

![alt text](image.png)

```shell
# delete pods immediately 
kubectl delete pod pod1 --foce --grace-period=0
```