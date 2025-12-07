

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

