
### Restart Behaviour
`spec.backoffLimit`: number of retries a job attempts to successfully finish the command that should be run. 
Job manifest must explicitly declare the restart policy : `spec.template.spec.restartPolicy` . The restart Policy of job can only be `Never` or `OnFailure`. For normal pod it is `Always`.

- `Never` -> job creates new pods in failure
- `OnFailure` -> job restarts the pod in failure

### activeDeadlineSeconds
The `activeDeadlineSeconds` is a **time limit for the entire Job execution**. It tells Kubernetes:
> “No matter what is happening, stop this Job after X seconds.”


#### 📦 Use case (when you actually need it)

##### 1. **Prevent runaway jobs**

If your job has a bug (e.g., infinite loop), it could run forever.

`activeDeadlineSeconds: 300`

➡️ After 5 minutes, Kubernetes **kills all Pods** and marks the Job as failed.

---

##### 2. **Cost control / resource protection**

In cloud environments, long-running jobs = $$$.

- Batch processing
- Data pipelines
- ML training experiments

➡️ You cap the maximum runtime.


