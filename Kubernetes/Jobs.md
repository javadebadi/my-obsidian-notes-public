Regular pods run forever and if they fail Kubernetes will restart them. This is appropriate for long-running processes such as web server or database applications.
For one-off tasks Kubernetes has **Job** object which creates a pod that is short lived and will be terminated after a successful completion. If the pod fails before a successful termination then Job controller will create another pod based on the template.

## Job Patterns
There are two parameters for jobs:
- `completions`: number of job completions
- `parallelism` : number of pods to run in parallel

Here are job patterns that you can choose based on nature of the workload:

| Type     | Use case            | Behavior                                                                                                                  | completions | parallelism |
| -------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------- | ----------- | ----------- |
| One shot | Database migrations | A single pod running once until successful termination - recreated in case of failure (node failure - application error ) | 1           | 1           |


### One shot
Here is an example of one shot workload:
```shell
kubectl create job migrate-users \
  --image=python:3.11 \
  -- python -c "
import time
print('Starting migration...')
time.sleep(60)
print('Migration done!')
"
```
![[k8-job-one-shot-imperative.png]]
Then check `kubectl get jobs` and you can see the job `migrate-users` is created and the `COMPLETIONS` is `0/1`.
![[k8-job-one-shot-get-jobs.png]]

Now, check the pods using `kubectl get pods`. You will see a new pod with name pattern of `migrate-users-xxxxx` is created and it is `Running`.
![[k8-job-one-shot-get-pods.png]]

Wait for few seconds for job to be completed. After few seconds if we run `kubectl get jobs` and `kubectl get pods` again:
![[k8-job-one-shot-completed.png]]
We see that the number of `COMPLETIONS` is now `1/1` and the status of the pod corresponding to the job is `Completed`.

To check the logs run: 
```shell
kubectl logs job/migrate-users
```
![[k8-job-one-shot-logs.png]]

#### Auto delete pod of the job
Check the pods in the cluster and you will see the pod is not deleted:
![[k8-job-one-shot-get-pods-2.png]]

To make this be deleted automatically we need change the YAML template of the pod by adding this: `ttlSecondsAfterFinished: 100`. Edit the job manifest:
```shell
kubectl edit job migrate-users
```
![[k8-job-one-shot-edit-job-template.png]]

Now if you check pods you can see the pod of the job was deleted:
![[k8-job-one-shot-get-pods-3.png]]
Also this will delete the job too:
![[k8-job-one-shot-get-jobs-3.png]]


```shell
kubectl run -i migrate-users \
  --image=python:3.11 \
  --restart=OnFailure \
  --command -- python -c "
import time
print('Starting migration...')
time.sleep(60)
print('Migration done!')
```