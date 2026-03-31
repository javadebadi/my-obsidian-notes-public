**The most important skill: imperative commands**

The exam gives you 2 hours for 19 questions — you can't afford to write YAML from scratch. You need to generate it fast. The pattern is:
```shell
kubectl run <name> --image=<image> --dry-run=client -o yaml > pod.yaml
```
`--dry-run=client -o yaml` generates the YAML without creating anything. You edit it, then apply. This is the foundation of everything in CKAD.


>[!info]- Task 1: Create a pod imperatively with `nginx` image and `env=prod` and `app=web` labels
> To do this
> ```shell
> kubectl run nginx-pod --image=nginx --labels="app=web,env=prod"
> kubectl get pods --show-labels
> ```
> ![[ckad-task1-create-pod-imperatively.png]]

>[!info]- Task 2: Generate YAML without creating, add annotation and then apply.
> ```shell
> kubectl run annotated-pod --image=nginx --annotations="owner=javad" --dry-run=client -o yaml > annotated-pod.yaml
> 
> kubectl apply -f annotated-pod.yaml
> 
> kubectl get pods
> ```

>[!info]- Task 3: Use labels for filtering.
> ```shell
> kubectl get pods -l "app=web"
> kubectl get pods -l "env=prod"
> kubectl get pods -l "app=web,env=prod"
> ```

>[!info]- Task 4: Create a new namespace `dev` and create a pod there
> ```shell
> kubectl create namespace dev
> kubectl run ns-pod --image=nginx -n dev
> kubectl get pods -n dev
> kubectl get pods --all-namespaces
> ```

> [!DANGER]- In exam use `-A` instead of `--all-namespaces`

You now have the three core skills for every CKAD task:

- ✅ Create pods **imperatively** with labels and annotations
- ✅ **Generate YAML** with `--dry-run=client -o yaml` and apply it
- ✅ **Filter** by namespace and **labels**


# Multiple Containers in a Pod
You cannot create multi-container pods imperatively — you always need YAML. This is one of the few cases where you must write it by hand, so the structure needs to be in your muscle memory.

**The concept:** multiple containers in a pod share the same network namespace (same IP, same localhost) and can share volumes. There are three patterns:

**Sidecar** — a helper container that supports the main container. Example: a log shipper running alongside your app, reading from a shared volume.

**Init container** — runs to completion before the main container starts. Used for setup tasks like waiting for a database to be ready, or pre-populating a volume.

**Ambassador / adapter** — less common on the exam, skip for now.

Create this pod:

```shell
# /tmp/multi-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  initContainers:
  - name: init-setup
    image: busybox
    command: ["sh", "-c", "echo initialized > /shared/init.txt"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  containers:
  - name: main-app
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "while true; do cat /shared/init.txt; sleep 5; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  volumes:
  - name: shared-data
    emptyDir: {}
```

and check logs for each 
![[Pasted image 20260326203729.png]]
`kubectl logs <pod> -c <container>` — always specify `-c` for multi-container pods or you'll get an error saying "specify a container".

`kubectl logs <pod> -c <init-container-name>` — you can still read init container logs after they complete. Useful for debugging failed init containers.


```shell
kubectl exec -it multi-pod -c main-app -- cat /shared/init.txt
```
![[Pasted image 20260326203821.png]]