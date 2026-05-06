By default, there are no restrictions to network traffic in k8s. Pods can always communicate, even if they are in other Namespaces. NetworkPolicies can be used to restrict communication between Pods.

NetworkPolicies need to be supported by the network plugin though:
- The Weave plugin does not support network policies
- The Calico plugin does support NetworkPolicy

If in a policy there is no match, traffic will be denied.
If no NetworkPolicy is used, all traffic is allowed.

## Setup minikube
Your minikube should be started with `Calico` CNI to have network policy. If not, then here is what you should do:
```shell
# stop and delete minikube
minikube stop
minikube delete

# start minikube with Calico CNI
minikube start --cni=calico
```

To confirm run this:
```shell
kubectl get pods -n kube-system | grep -i "calico"
```

And you should see pods with `Calico`:
![[Pasted image 20260505000813.png]]
```shell
❯ kubectl get pods -n kube-system | grep -i "calico"
calico-kube-controllers-565c89d6df-m5pbn   1/1     Running   0          2m9s
calico-node-lfpgp                          1/1     Running   0          2m9s
```



## Example
Here is an example of network policy:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access-nginx: "true"
  policyTypes:
  - Ingress

---
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always

---
apiVersion: v1
kind: Pod
metadata:
  labels:
    access-nginx: "true"
  name: busybox
spec:
  containers:
  - image: busybox
    name: busybox
    command: ['sleep', '3600']
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

after applying this manfiest you should expose the pod (`kubectl expose pod nginx --port=80`)
```shell
❯ kubectl expose pod nginx --port=80
service/nginx exposed
```

Now let's run a command:
```shell
❯ kubectl exec busybox -- wget --spider --timeout=1 nginx
Connecting to nginx (10.108.224.146:80)
remote file exists
```

Now remove the label from `busybox`:
```shell
❯ kubectl label pod busybox access-nginx-
pod/busybox unlabeled
❯ kubectl exec busybox -- wget --spider --timeout=1 nginx
Connecting to nginx (10.108.224.146:80)
wget: download timed out
command terminated with exit code 1
```
