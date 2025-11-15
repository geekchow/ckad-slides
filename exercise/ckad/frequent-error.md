# frequent meet errors

## Not all field could be updated.
You cannot modify any field of a existing pod, for eg: command of container.
You have to remove it and re-create a new one.

```bash
The Pod "nevatest" is invalid: spec: Forbidden: pod updates may not change fields other than `spec.containers[*].image`,`spec.initContainers[*].image`,`spec.activeDeadlineSeconds`,`spec.tolerations` (only additions to existing tolerations),`spec.terminationGracePeriodSeconds` (allow it to be set to 1 if it was previously negative)
  core.PodSpec{
  	Volumes:        {{Name: "kube-api-access-6vgdv", VolumeSource: {Projected: &{Sources: {{ServiceAccountToken: &{ExpirationSeconds: 3607, Path: "token"}}, {ConfigMap: &{LocalObjectReference: {Name: "kube-root-ca.crt"}, Items: {{Key: "ca.crt", Path: "ca.crt"}}}}, {DownwardAPI: &{Items: {{Path: "namespace", FieldRef: &{APIVersion: "v1", FieldPath: "metadata.namespace"}}}}}}, DefaultMode: &420}}}},
  	InitContainers: nil,
  	Containers: []core.Container{
  		{
  			Name:       "nevatest",
  			Image:      "busybox",
- 			Command:    nil,
+ 			Command:    []string{"sh", "-c", "sleep infinity"},
  			Args:       nil,
  			WorkingDir: "",
  			... // 19 identical fields
  		},
  	},
  	EphemeralContainers: nil,
  	RestartPolicy:       "Always",
  	... // 28 identical fields
  }


```