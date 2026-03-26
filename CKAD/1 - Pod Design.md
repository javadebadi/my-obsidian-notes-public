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
