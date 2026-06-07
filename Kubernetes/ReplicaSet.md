


To create a simple ReplicaSet run:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  labels:
    app: nginx
    version: "2"
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      version: "2"
  template:
    metadata:
      labels:
        app: nginx
        version: "2"
    spec:
      containers:
        - name: nginx
          image: nginx
```

![[replicaset-template-yaml.png]]


Note that we have several labels in the template and they are necessary to let ReplicaSet manage pods. Without correct labeling the replicaset will not be able to manage pods.

| Location                   | Role                                                       |
| -------------------------- | ---------------------------------------------------------- |
| `metadata.labels`          | Label the ReplicaSet itself                                |
| `selector.matchLabels`     | Define which Pods are managed by the ReplicaSet            |
| `template.metadata.labels` | Define labels for created Pods so ReplicaSet can find them |

There are few things you should know about selection of pods by ReplicaSet.
The Pod must satisfy **all labels** to be selected. So if `selector.matchLabels` has `app:nginx` and `version:2` then those pods will be matched that have this condition:
`app=nginx` **AND** `version=2`.

Based on that argument you should have this relationship:
$$Set(\text{labels in }selectrs.matchLabels) \subset Set(\text{lables in }template.metadata.labels)$$

To create the ReplicaSet run:
```shell
kubectl apply -f nginx-replicaset.yaml

# watch pods being created
kubectl get pods -w

# watch final pods
kubectl get pods
```
![[replicaset-apply-template-watch-pods.png]]

Also you can get more information about the replicaset using `describe`:
```shell
kubectl describe replicaset nginx
```
![[Pasted image 20260330211902.png]]


### Finding a ReplicaSet from a Pod
If you want to find out if a pod is managed by replicaset, then it is helpful to know that ReplicaSet adds `ownerReferences` section into pods metadata. To query for this run:

```shell
kubectl get pod <pod-name> -o jsonpath='{.metadata.ownerReferences[0].name}'
```
![[Pasted image 20260330212342.png]]



## Scaling ReplicaSet

### Scale
to scale you can either edit `replicas=3` in the YAML file and apply the changes (declarative way).
Or imperatively using
```shell
kubectl scale replicaset nginx --replicas=4
```

### Auto-scaling
To create autoscaling (based on cpu or other metrics) run:
```shell
kubectl autoscale rs nginx --min=2 --max=5 --cpu='80%'
```

To view the created `horizontalpodautoscalers` object:
```shell
kubectl get horizontalpodautoscalers nginx
```

![[Pasted image 20260330221452.png]]

or shorter version:
```shell
kubectl get hpa
```
