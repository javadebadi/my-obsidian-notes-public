Regular pods run forever and if they fail Kubernetes will restart them. This is appropriate for long-running processes such as web server or database applications.
For one-off tasks Kubernetes has **Job** object which creates a pod that is short lived and will be terminated after a successful completion. If the pod fails before a successful termination then Job controller will create another pod based on the template.

## Job Patterns
There are two parameters for jobs:
- `completions`: number of job completions
- `parallelism` : number of pods to run in parallel

Here are job patterns that you can choose based on nature of the workload:

| Type                                  | Use case                                                | Behavior                                                                                                                  | completions | parallelism |
| ------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ----------- | ----------- |
| One shot                              | Database migrations                                     | A single pod running once until successful termination - recreated in case of failure (node failure - application error ) | 1           | 1           |
| Parallel fixed completions            | Multiple pods processing a set of work in parallel      | One or more pods running one or more times until reaching a fixed completion count                                        | 1+          | 1+          |
| Work queue: parallel jobs<br><br><br> | Multiple pods processing from a centeralized work queue | One or more Pods running once until successful termination                                                                | 1           | 1+          |



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



### Parallelism

We use `--dry-run` to create a job template and then we can set `parallelism` parameters in the template.
Here we run a job that is meaningful to be parallelized: finding number of prime numbers. Note that to do this we have used global variable `JOB_COMPLETION_INDEX` which is injected by Kubernetes to the pod it creates for a job when the job has `completionMode: Indexed`.  

```shell
kubectl create job prime-calculator \
  --image=python:3.11-slim \
  --dry-run=client \
  -o yaml \
  -- python -c "
import os, math
task_id = int(os.environ.get('JOB_COMPLETION_INDEX', 0))
start = task_id * 1000 + 2
end = start + 1000
primes = [n for n in range(start, end) if all(n % i != 0 for i in range(2, int(math.sqrt(n))+1))]
print(f'Task {task_id}: Found {len(primes)} primes between {start} and {end}')
print(f'First few: {primes[:5]}')
" > parallel-job.yaml
```
the `parallel-job.yaml` content is like this when created using `--dry-run=client`:
![[k8-job-parallelism-dry-run.png]]We need to add three keys to `.spec` of the template:
```
parallelism: 5
completions: 10
completionMode: Indexed
```
![[k8-job-parallelism-dry-run-edit-spec.png]]
and now lets apply `kubectl apply -f parallel-job.yaml`  and immediately check for pods `kubectl get pods --watch`:
![[Pasted image 20260327203923.png]]
If finally we check the pods you can see `10` pods were created:
![[Pasted image 20260327204041.png]]
and if we check the job itself `kubectl get jobs/prime-calculator`
![[Pasted image 20260327204123.png]]

to check logs from all pods of the job run:
```shell
kubectl logs -l job-name=prime-calculator --prefix=true
```
![[Pasted image 20260327204519.png]]

### Work queue: parallel jobs
This kind of work load uses parallelism but to complete a single job.
![[producer_workqueue_consumer.svg]]
**Producer** pushes all tasks into the queue upfront using `RPUSH`. This could be your app, a script, or a Kubernetes init job.

**Work Queue (Redis)** holds all pending tasks as a list. The key property is that `LPOP` is **atomic** — even if 3 workers call it simultaneously, each gets a unique task with no duplicates.

**Workers (consumers)** each pop one task, process it, then immediately pop the next — so the fastest worker naturally does the most work. When the queue is empty, all workers exit cleanly and Kubernetes marks the Job as complete.

This is fundamentally different from an Indexed job where each worker has a pre-assigned task — here the work is **dynamically distributed** at runtime.
#### Indexed Job vs Work Queue Job

|                   | Indexed Job                 | Work Queue Job                            |
| ----------------- | --------------------------- | ----------------------------------------- |
| `completionMode`  | `Indexed`                   | `NonIndexed`                              |
| `completions`     | Fixed (e.g. 10)             | Often not set (or set to 1)               |
| `parallelism`     | Fixed                       | Fixed                                     |
| How pods get work | Via `JOB_COMPLETION_INDEX`  | Pull from a queue (Redis, RabbitMQ, etc.) |
| Pod stops when    | Its index task is done      | Queue is empty                            |
| Real World        | Partition/Shared Processing | Dynamic task distribution                 |
In these kind of job you should have these keys for job template `.spec`:
```yaml
parallelism: 5
# No completions set
# No completionMode
```

Here we implement an example for this use case. 

#### Step 1: Work queue (deploy Redis)
First we need a work queue pod that stores the pod. The publisher will push data to this work queue.
```shell
kubectl run redis --image=redis:7 --port=6379
kubectl expose pod redis --port=6379 --name=redis-service
```
![[k8-job-work-queue-work-queue-pod-and-svc.png]]

#### Step 2: Populate the queue with tasks
Now, we have the queue and it is exposed with name `redis-service` in the Kubernetes cluster and we should push some data to it.
```shell
# Push 10 tasks into Redis queue
kubectl run queue-filler \
--image=redis:7 \
--restart=Never \
--rm -it \
-- redis-cli -h redis-service RPUSH work-queue \
"image_001.jpg" "image_002.jpg" "image_003.jpg" \
"image_004.jpg" "image_005.jpg" "image_006.jpg" \
"image_007.jpg" "image_008.jpg" "image_009.jpg" \
"image_010.jpg"
```
![[k8-job-work-queue-populate-queue.png]]

#### Step 3: The worker script
Each worker pod will:

1. Connect to Redis (`r = `)
2. **Pop** a task from the queue (atomic — no two pods get same task)
3. Process it
4. Repeat until queue is empty
5. Exit cleanly → Kubernetes marks pod as complete

```python
# worker.py
import redis
import time
import os

r = redis.Redis(host='redis-service', port=6379)

print(f"Worker started, connecting to queue...")

while True:
    # LPOP atomically pops one task - no duplicate processing!
    task = r.lpop('work-queue')
    
    if task is None:
        print("Queue is empty, worker exiting successfully.")
        break  # Exit cleanly → pod completes successfully
    
    task_name = task.decode('utf-8')
    print(f"Processing: {task_name}")
    
    # Simulate work (e.g. resize image, run ML inference, etc.)
    time.sleep(30)
    
    print(f"Done: {task_name}")

print("Worker finished.")
```


#### Step 4: Build a docker file for worker container
We create a Dockerfile
```shell
cat <<EOF > Dockerfile
FROM python:3.11-slim
RUN pip install redis
COPY worker.py .
CMD ["python", "worker.py"]
EOF

docker build -t my-worker:v1 .
```

![[Pasted image 20260327232141.png]]

#### Step 5: Push to docker hub
Next step is to push docker image to a registry so K8 can pull that to build the container of the pod.
If you are not logged in to your docker account run:
```shell
docker login
```
![[Pasted image 20260327232724.png]]

Build docker image using your docker username:
```shell
# Build with your username in the tag
docker build -t yourusername/my-worker:v1 .
```
![[Pasted image 20260327232851.png]]

Then we push to docker hub:
```shell
# Push to Docker Hub
docker push javadebadi/my-worker:v1
```
![[Pasted image 20260327233031.png]]

#### Step 6: Create the work queue job
Similar to what we already showed using `--dry-run` you can create a job template like this:
```yaml
# work-queue-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: image-processor
spec:
  parallelism: 3        # ✅ 3 pods run simultaneously
  # No completions set  # ✅ Job ends when all pods exit successfully
  # No completionMode   # ✅ Defaults to NonIndexed
  template:
    spec:
      containers:
      - name: worker
        image: javadebadi/my-worker:v1
        env:
        - name: REDIS_HOST
          value: "redis-service"
      restartPolicy: Never
```
![[Pasted image 20260327233436.png]]

Watch for job being processed
```shell
kubectl get pods -l job-name=image-processor -w
```
![[Pasted image 20260327233654.png]]


or you can stream the logs
```shell
kubectl logs -l job-name=image-processor --prefix=true -f
```
![[Pasted image 20260327233753.png]]
![[Pasted image 20260327233840.png]]


## CronJob
CronJob is an object that creates Jobs in regular intervals:
```
CronJob
  │
  │  every * * * * *
  │  creates a new ──►  Job
  │                       │
  │  every * * * * *      │  creates ──►  Pod(s)
  │  creates a new ──►  Job               │
  │                       │               │  runs your
  │  every * * * * *      │  creates ──►  Pod(s)
  │  creates a new ──►  Job               │  container
                                          │
                                       completes ✓
```
To create a cron job run:
```shell
kubectl create cronjob date-printer \
  --image=busybox \
  --schedule="* * * * *" \
  -- /bin/sh -c "echo Hello from $(hostname) at $(date)"
```
This will print data every minute (`* * * * *`).
![[Pasted image 20260328000310.png]]


If now you check pods you will see every 1 minute a pod is created:
![[Pasted image 20260328000959.png]]
Note that K8 only keeps last 3 successful pods as can be seen in the screenshot above where I have run `kubectl get pods` few times.

Also you can use `kubectl get pods --watch`
![[Pasted image 20260328001130.png]]

and `kubectl get jobs --watch` shows new jobs are created every minute:
![[Pasted image 20260328001429.png]]