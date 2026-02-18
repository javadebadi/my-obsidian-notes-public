Pods are group of containers that will be deployed together in a single node and will be act as a single atomic unit. The containers in the pod need to access to shared file systems but at the same time it does not make sense to combine those containers to make them on container (because of resource isolation and different requirements of each service).

Container Foo + Container Bar + Shared File System => Pod

Pods are smallest deployable artifacts in a Kubernetes cluster.

Applications running in a pod have same IP address, same hostname and access to OS level communication and networking. For example container Foo in Pod A can communicate with container Bar in Pod A by sending a request into `http://localhost:8080` if container B is running a web server on port `8080`.

## Pod Design Pattern
One design question in building applications is whether two containers should be inside a Pod or in separate pods. To answer this question here is the critieria:
- Ask yourself: Can these two application work correctly when thy are hosted in different machines?
	- YES -> Two separate pods
	- No -> They should go together in a pod


## Creating a Pod Imperatively 


```shell
kubectl run myapp --image=nginx:alpine --port=80
```

![[Pasted image 20260217215357.png]]

To delete the pod
```shell
kubectl delete pod myapp
```
![[Pasted image 20260217215509.png]]

## Creating a Pod declaratively
Create `myapp.yaml` file with this content:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - image: nginx:alpine
    name: myapp-nginx
    ports:
    - containerPort: 80
      name: http
      protocol: TCP

```

![[Pasted image 20260217220844.png]]


## Get Pod Info
```shell
kubectl get pods
```
![[Pasted image 20260217222237.png]]The status `Running` means the pod is running. If the status is `Pending` it means the pod is submitted to Kubernetes but not been scheduled yet. If there are errors then you see error status in that field.

To get detail information about the pod and its events:
```shell
kubectl describe pod myapp
```
The output will show information about the pod, containers and finally the Events on the pod:
![[Pasted image 20260217222833.png]]

### Access Logs
- `kubectl logs myapp`
- `kubectl exec myapp -- bash`

[Video - Learn Basics of the Pods](https://drive.google.com/file/d/1uJy1eyg5oqa3DzU4B6PpHrKqvXyU7MSb/view?usp=drive_link)
## Delete the Pod
```shell
kubectl delete -f myapp.yaml
```
When you delete a pod it is not being killed immediately, instead the state of the pod will be changed to `Terminated` and it will stop receiving  new requests. There is a grace shutdown period that allows the pod to finish the jobs that were running before being deleted.