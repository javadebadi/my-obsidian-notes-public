```shell
kubectl version

kubectl get nodes

kubectl get componentstatuses
```

```shell
kubectl get pods -n kube-system -l k8s-app=kube-proxy

kubectl get pods -n kube-system -l k8s-app=kube-nds

kubectl get svc -n kube-system
```

```shell
kubectl get pods -n kube-system

kubectl get pods --namespace default

kubectl get pods --all-namespaces
```

```shell
kubectl get pod kube-proxy-hxk94 -n kube-system -o jsonpath='{.spec.containers[*].image}'
```
