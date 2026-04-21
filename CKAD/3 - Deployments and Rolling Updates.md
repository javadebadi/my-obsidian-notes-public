You already know deployments basics, so we'll focus on the parts that actually appear on the exam — rolling updates, rollbacks, and scaling.

**The key idea:** a deployment manages a ReplicaSet, which manages pods. When you update an image, Kubernetes creates a new ReplicaSet and gradually shifts pods from old to new — that's a rolling update



>[!info]- Task 1 — Create a deployment and scale it:
> In CKAD, you should create deployment imperatively because it is faster:
> ```shell
> kubectl create deployment web --image=nginx:1.19 --replicas=3
kubectl get deployments
kubectl scale deployment web --replicas=5
kubectl get pods
> 
> ```
> Here is the output of the commands:
> ```shell
>❯ kubectl create deployment web --image=nginx:1.19 --replicas=3 deployment.apps/web created
❯ kubectl get deployments
NAME READY UP-TO-DATE AVAILABLE AGE
web 0/3 3 0 7s
❯ kubectl scale deployment web --replicas=5
deployment.apps/web scaled
❯ kubectl get pods
NAME READY STATUS RESTARTS AGE
web-76497c8d96-59rnw 1/1 Running 0 6s
web-76497c8d96-gsw7d 1/1 Running 0 30s
web-76497c8d96-mdf5g 1/1 Running 0 30s
web-76497c8d96-psmrh 1/1 Running 0 6s
web-76497c8d96-rlks7 1/1 Running 0 30s
> ```
>
> 


>[!info]- Task 2 — Perform a rolling update
>Use `set image` to update the image
>
> ```shell
> kubectl set image deployment/web nginx=nginx:1.21
kubectl rollout status deployment/web
kubectl describe deployment web | grep Image
> ```
>
> Here is the output:
> ```shell
> ❯ kubectl set image deployment/web nginx=nginx:1.21
deployment.apps/web image updated
❯ kubectl rollout status deployment/web
Waiting for deployment "web" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 4 of 5 updated replicas are available...
Waiting for deployment "web" rollout to finish: 4 of 5 updated replicas are available...
deployment "web" successfully rolled out
❯ kubectl describe depoloyment web | grep "Image"
error: the server doesn't have a resource type "depoloyment"
❯ kubectl describe deployment web | grep "Image"
Image:         nginx:1.21
>```


>[!info]- Task 3 — Roll back
>Use `rollout undo` to update the image
>
> ```shell
> kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl describe deployment web | grep Image
> ```
> Here is the output of commands:
> ```shell
> ❯ kubectl rollout history deployment/web
deployment.apps/web
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
❯ kubectl rollout undo deployment/web
deployment.apps/web rolled back
❯ kubectl describe deployment web | grep Image
Image:         nginx:1.19
> ```
> Also you can add annotations for change-cause:
> ```shell
> kubectl annotate deployment/web kubernetes.io/change-cause="dwongraded to nginx 1.19"
> ```
> Here is the output of commands:
> ```shell
> ❯ kubectl annotate deployment/web kubernetes.io/change-cause="dwongraded to nginx 1.19"
deployment.apps/web annotated
❯ kubectl rollout history deployment web
deployment.apps/web
REVISION  CHANGE-CAUSE
2         <none>
3         dwongraded to nginx 1.19
> ```


>[!info]- Task 4 — **Pause and resume** (useful for making multiple changes at once):
>Use `rollout pause` and `rollout resume`. The pause/resume pattern is useful when you want to make **multiple changes at once** without triggering a rollout after each change.
>- **Editing YAML** → single rollout automatically, no need to pause
>- **Multiple imperative commands** → use pause/resume to batch them into one rollout- 
>
> ```shell
> kubectl rollout pause deployment/web
kubectl set image deployment/web nginx=nginx:1.25
kubectl rollout resume deployment/web
kubectl rollout status deployment/web
> ```
> Here is the output of commands:
> ```shell
> ❯ kubectl rollout pause deployment/web
deployment.apps/web paused
❯ kubectl set image deployment/web nginx=nginx:1.25
deployment.apps/web image updated
❯ kubectl rollout resume deployment/web
deployment.apps/web resumed
❯ kubectl rollout status deployment/web
Waiting for deployment "web" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 4 of 5 updated replicas are available...
Waiting for deployment "web" rollout to finish: 4 of 5 updated replicas are available...
deployment "web" successfully rolled out
> ```

