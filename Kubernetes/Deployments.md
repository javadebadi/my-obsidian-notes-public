The Deployment object manages the release of new versions of an application. It is very close to daily routine of deployment by developers to rollout a new version and rollback to previous versions in case of errors.
Kubernetes is an online, self-healing system. The top-level Deployment object is managing the ReplicaSet. The ReplicaSet manages the Pods. If we you delete a Pod manually the ReplicaSet controller will notice and create a new pod. If you scale down the ReplicaSet then the Deployment controller will notice and will re-scale the ReplicaSet.
Similar to `ReplicaSet` the deployment can be scaled up or down using `kubectl scale deployment <deployment-name> --replicas=<new-count>`
## Deployment Updates
Deployments make applications updates easier. To manage how applications are updated the update strategy is used. Default strategy is rolling update which updates application in batches.
```
strategy.type.rollingUpdate
```
To ensure there is 0 downtime. As a result of `rollingUpdate` different version of the application will be running. 
The `recreate` strategy will be bring down all previous version of the application and then new versions will be up. But it will be a downtime.

### Recreate Strategy
First create a deployment template
```shell
kubectl create deployment nginx-deployment --image=nginx:1.14 --replicas=6 --dry-run=client -o yaml > nginx-deployment.yaml
```
Then we edit the `nginx-deployment.yaml` by adding the `strategy`:
```yaml
  strategy:
    type: Recreate

```

![[Pasted image 20260403225029.png]]

Then we apply the template:
```shell
kubectl apply -f nginx-deployment.yaml
```
![[Pasted image 20260403225146.png]]
![[Pasted image 20260403225213.png]]

### Rollout to a New Version
Now, let's update the deployment by updating the nginx image from `1.14` to `1.15` and then apply the template and immediately check the resources:
![[Pasted image 20260403225406.png]]
![[Pasted image 20260403230129.png]]
we see 2 ReplicaSets are available and only the new one has running pods.

### Rollout History
Using the `kubectl rollout history` we can check available versions of the deployment:
```shell
kubectl rollout history deployment nginx-deployment
```
![[Pasted image 20260403230326.png]]

### Rollout Undo
To undo the update we can change to a revision using `kubectl rollout undo`:
![[Pasted image 20260403230517.png]]

and now if we check revisions:
![[Pasted image 20260403230613.png]]
![[Pasted image 20260403230655.png]]
Now, you can see the old replica is the one running pods.

Also it is worth to not that the replica sets has an annotation that keeps the version:
![[Pasted image 20260403230950.png]]

the annotation is `"deplyment.kubernetes.io./revision": "3"`.

> [!WARNING] Best Practice for Rollback (Rollout Undo)
> A preferable way to undo a rollout is to revert the YAML template file and apply the previous version. This way the tracked changes in source code of your K8 files will match with what is really running in the K8 cluster.


### RollingUpdate Strategy
We can change the update strategy like this:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: '25%'
    maxUnavailable: '25%'
  
```

![[Pasted image 20260403231511.png]]

and then we apply the template. 

![[Pasted image 20260403231813.png]]


> [!NOTE] Blue/Green Deployment using RollingUpdate Startegy
> Setting `maxSurge` to `100%` is equivalent to a blue/green Deployment. There will be only one version of the application running all the time not mixed of two versions. The Deployment controller first scales the new version up to 100% of the old version. Once the new version is running and healthy, it immediately scales the old version down to 0%.

## Health of Pods
Kubernetes makes sure new pods are healthy by using readiness checks.
There is a parameter in Kubernetes spec that you can use to set the time it should wait until make sure the pod is ready:
```yaml
spec:
  minReadySeconds: 60
```

and sometime your app my encounter an error like deadlock and it will stuck forever in that state. In these cases having a timeout is useful to make deployment fail and stop the rollout. To do this there is a parameter in deployment spec:
```yaml
spec:
  progressDeadlineSeconds: 600
```
