**Labels**: Key/value pairs attached to Kubernetes objects and can be used to identify resource
**Annotations**: Key/value pairs attached to Kubernetes objects for non-identifying purposes

## Labels

Let's create some deployments and add labels:
```shell
kubectl create deployment nginx-deployment-1 \  
--image=nginx:alpine \  
--replicas=2
```
![[Pasted image 20260221124258.png]]
Then add label using
```shell
kubectl label deployment nginx-deployment-1 version=1 env=prod
```

![[Pasted image 20260221124530.png]]

To check the deployment has labels run:
```shell
kubectl get deployment nginx-deployment-1 --show-labels
```
![[Pasted image 20260221124741.png]]

Note that we have 3 labels. The `app-nginx-deployment-1` label was automatically set when used `kubect create` command.

Let's create another deployment similarly:
![[Pasted image 20260221125024.png]]

Now lets check all deployments with labels:
```shell
kubectl get deoployments --show-labels
```
![[Pasted image 20260221125122.png]]


### View Label as a Columns
To view a specific label as column in the `kubectl` output use `-L` with key of the label:
```shell
kubectl get deployments -L version
```
![[Pasted image 20260221125422.png]]

Another example to see `k8s-app` label on **pods** of `kube-system` namespace:
```shell
kubectl get pods -L k8s-app -n kube-system
```

![[Pasted image 20260221125615.png]]


### Label Selectors
Label selectors are used to filter Kubernetes objects based on labels. To do this we need to use `--selector` flag.

To filter all deployments with label `version=1`
```shell
kubectl get deployments --selector="version=1"
```

![[Pasted image 20260221131527.png]]

To filter all deployments with label `env=prod`:
```shell
kubectl get deployments --selector="env=prod"
```
![[Pasted image 20260221131504.png]]


You don't only have the `--selector="key=value"` option. Here is the table of selector operators to filter:

| Operator                   | Description                                                      |
| -------------------------- | ---------------------------------------------------------------- |
| key=value                  | is set to value                                                  |
| key!=value                 | is not set to value                                              |
| key in (value1, value2)    | key is one of value1 or value2                                   |
| key notin (value1, value2) | key is not any of value1 or value2                               |
| key                        | key is set (to check a label is applied regardless of the value) |
| !key                       | key is not set                                                   |

Here are few examples:
![[Pasted image 20260221132228.png]]

And using `,` we can have logical `AND` to combine two or many selectors:
```shell
kubectl get deployments --selector="env!=prod,version in (1)"
```
![[Pasted image 20260221132357.png]]



Kubernetes labels are useful for organizing applications and objects by users. However, they are also necessary to relate objects by Kubernetes itself. Many Kubernetes constructs used selector queries to find an object in Kubernetes cluster.


## Annotations
Annotations are mostly used to provide extra information such as build hash or commit hash of image or code, icons, etc.
One of the ways they are used in Kubernetes is they can be used to track rollout information.
