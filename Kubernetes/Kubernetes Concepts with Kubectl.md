
## Namespace
Namespaces are used to organized objects in a Kubernetes cluster.

To change namespace in kubectl you can use `-n` or `--namespace` flags.
```shell
kubectl get pods -n kube-system
```
![[Pasted image 20260216160548.png]]

```shell
kubectl get pods --namespace default
```

the default value for namespace in using `kubectl` is `default`.

To interact with objects in all namespaces you can use `--all-namespaces` flag:
```shell
kubectl get pods --all-namespaces
```
![[Pasted image 20260216160508.png]]

## Kubectl Configuration
The `kubectl` configuration is stored in `$HOME/.kube/config` file. This configuration contains information about how to find the cluster and how to authenticate to the cluster.

Run this command:
```shell
cat ~/.kube/config
```

Here is the content of that file in my computer:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /Users/javad/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Sat, 14 Feb 2026 23:47:31 MST
        provider: minikube.sigs.k8s.io
        version: v1.38.0
      name: cluster_info
    server: https://127.0.0.1:60382
  name: minikube
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Sat, 14 Feb 2026 23:47:31 MST
        provider: minikube.sigs.k8s.io
        version: v1.38.0
      name: context_info
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
users:
- name: minikube
  user:
    client-certificate: /Users/javad/.minikube/profiles/minikube/client.crt
    client-key: /Users/javad/.minikube/profiles/minikube/client.key
```

This is file is like a **connection profile**, maybe you can think of it as similar to ssh configuration. If you have multiple kubernetes cluster the first section (`cluster`) key of the config file can be like this:
```yaml
clusters:
- cluster:
    server: https://127.0.0.1:60832
    certificate-authority: /Users/javad/.minikube/ca.crt
  name: minikube
- cluster:
    server: https://dev-k8.company.com
    certificate-authority: /path/to/dev/ca.crt
  name: dev
- cluster:
    server: https://prod-k8.company.com
    certificate-authority: /path/to/prod/ca.crt
  name: prod
```
## Contexts
One part of the config file is context. You can change the context if you want to connect to different cluster or have a different default namespace in a cluster.
To create a context with different default namespace run:
```shell
kubectl config set-context my-test-context --namespace=my-test-default
```
![[Pasted image 20260216170432.png]]

After context is created you can check the `$HOME/.kube/config` to see new context is created:
![[Pasted image 20260216170522.png]]

To use the created context run:
```shell
kubectl config use-context my-test-context
```
(Note that the context we created does not have some cluster and user and therefore kubectl will raise errors when you try to use it)

