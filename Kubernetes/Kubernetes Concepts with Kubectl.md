 
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

## Kubernetes and RESTful API
- **Kubernetes API Server**: Every cluster exposes a RESTful API at a base URL, e.g., `https://mydev.example.com` or in the above screenshots `http://127.0.01:60382`. 
    
- **kubectl**: A client that communicates with the API server using HTTPS and kubeconfig credentials. Every `kubectl` command is translated into a REST call.

|Resource Scope|REST Path Pattern|Example|
|---|---|---|
|Namespaced|`/api/<version>/namespaces/<namespace>/<resource>/<name>`|Pod `kube-proxy` in `kube-system`: `/api/v1/namespaces/kube-system/pods/kube-proxy-hxk94`|
|Cluster-scoped|`/api/<version>/<resource>/<name>`|Node `node-1`: `/api/v1/nodes/node-1`|
|Custom Resource (CRD)|`/apis/<group>/<version>/namespaces/<namespace>/<resource>/<name>`|MyCRD in `default` namespace|

So when you run:
```shell
kubectl get pod kube-proxy-hxk94 -n kube-system

```

It is getting translated to this request by `kubectl`:
```http
GET /api/v1/namespaces/kube-system/pods/kube-proxy-hxk94
```


## Kubectl View Commands
The are few commands to view objects in the Kubernetes cluster:
- get
- describe

For the get command here is the pattern:
```shell
kubectl get <resource>

# if you want to view a particualr object of a resource type
kubectl get <resource> <name> 
```

There are options to change output format that you can use in kubectl commands:
- `-o wide` -> more details
- `-o json` -> complete object info in JSON fromat
- `-o yaml` -> complete object info in YAML format

If you want to see more than one resource it can be done by adding `,`:
```shell
kubectl get pods,services
```

Similarly the `describe` command gives more information about the object:
```shell
kubectl get <resource> <name> 
```

### Jsonpath
There are many output formats for `kubectl`. One of them is `jsonpath` and it supports querying for a specific field of a Kubernetes resource.
```shell
kubectl get pod kube-proxy-hxk94 -n kube-system -o jsonpath='{.spec.containers[*].image}'
```

![[Pasted image 20260216173934.png]]

Watch this:
https://drive.google.com/file/d/1-DZDvQOAud-IHByWaiSczq1p0THcAvAL/view?usp=drive_link

### Remove Headers
One of the common tasks in working with `kubectl` is to pipe results to tools such as `awk` . The `--no-headers` removes headers from the printed results of `kubectl` which is makes it suitable for these kind of tasks.

### Continuous Watching
Using `--watch` with `kubectl get` command you can continuously observer the state of a Kubernetes resource until the event you are looking for starts to happen.

### Explain
To see list of all field available on a Kubernetes resource kind use `kubectl explain <resource>`

```shell
kubectl explain pods
```


## Kubectl CRUD commands
There are few commands to create, update or delete resource in Kubernetes:
- `kubectl apply -f <file-name.yaml>`
- `kubectl edit <resource> <name>`
- `kubectl delete <resource> <name>`

## Labeling and Annotating
Labels and annotations are tags for Kubernetes objects.
- `kubectl label pods bar color=red`
- `kubectl label pods bar color=blud --overwrite`
- `kubectl label pods bar color-`

The `annotate` has similar syntax.

## Debugging Commands
- `kubectl logs <pod-name> -c <container-name>` (viewing logs)
- `kubectl logs <pod-name> -c <container-name> -f` (streaming logs)
- `kubectl exec -it <pod-name> -c <container-name> -- bash` (interactive shell inside the container)
- `kubectl attach -it <pod-name> -c <container-name>` (attach terminal to the container)
- `kubectl cp <pod-name>:</path/to/remote/file> <path/to/local/file>` (copy file from container to local)
- `kubectl port-forward <pod-name> 8080:80` (open a connection that forwards traffic from local machine on port 8080 to the remote container on port 80)
- `kubectl get events (--watch)` Kubernetes events
- `kubectl top nodes` (resources used by nodes)
- `kubectl top pods` (resources used by pods)
- 