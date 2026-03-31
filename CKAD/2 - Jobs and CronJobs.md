
**The concept:**

A regular pod restarts if it fails. Jobs are different — they run a task to **completion** and stop. CronJobs schedule Jobs on a time basis.

Three things the exam tests here:

**Job** — run something once (or N times). Example: database migration, batch processing.

**CronJob** — run something on a schedule. Example: nightly report, cleanup task.

**Key fields to know:**

- `completions` — how many times the job must succeed
- `parallelism` — how many pods run simultaneously
- `backoffLimit` — how many retries before marking as failed


>[!info]- Task 1 — Create a Job imperatively from `busybox` image that prints `hello from job` in the stdout
> To do this
> ```shell
kebectl create job hello-job --image=busybox -- echo "hello from job"
kubectl get jobs
kubectl get pods
kubectl logs -l job-name=hello-job
> ```
> ![[ckad-2-jobs-create-job-imperatively.png]]
> If you wanted a bit complicated command then you can write something like this:
> ```shell
> kubectl create job hello-job --image=busybox -- sh -c "echo start && sleep 20 && echo end"
> ```
> I have used `sleep 20` so it is a good example to use `-f` for streaming of logs:
> ![[ckad-2-jobs-create-job-imperatively-2.png]]
> 

>[!info]- Task 2 — Create a Job with multiple completions
> To do this we need to use imperative command with `--dry-run` to create the YAML template for the job and then edit the YAML by adding `parallelism` and `completions` fields.
> ```shell
kubectl create job hello-job --image=busybox --dry-run=client -o yaml -- echo "hello from job" -> multi-job.yaml
> ```
> ![[ckad-2-job-multi-completions-1.png]]
> Then we need to edit the `.spec` of the job:
> ```yaml
> spec:
>   parallelism: 3
>   completions: 9
> ```
> ![[ckad-2-job-multi-completions-edit-yaml.png]]
> Then we apply the manfiest and check for jobs status using `-w`:
> ```shell
> kubectl apply -f multi-job.yaml
> # check for jobs
> kubectl get jobs -w
> ```
> ![[ckad-2-job-multi-completions-apply-yaml.png]]
> 

>[!info]- Task 3 — Create a cron job imperatively
> 
> ```shell
kubectl create cronjob cron-date-job --image=busybox --schedule="* * * * *" -- date
> ```
> and to view the logs run:
> ```shell
> kubectl get pods | grep cron-date-job | awk '{print $1}' | xargs -I{} kubectl logs {}
> ```
> and to view the logs of jobs run:
> ```shell
> kubectl get jobs | grep cron-date-job | awk '{print $1}' | xargs -I{} kubectl logs job/{}
> ```
> ![[Pasted image 20260328224433.png]]


