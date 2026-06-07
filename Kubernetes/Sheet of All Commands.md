```shell
kubectl version

kubectl get nodes

kubectl get componentstatuses
```

```shell
kubectl get pods -n kube-system -l k8s-app=kube-proxy

kubectl get pods -n kube-system -l k8s-app=kube-dns

kubectl get svc -n kube-system
```


```shell
kubectl get pod <pod-name> -n kube-system -o jsonpath='{.spec.containers[*].image}'
```

### Access pod locally for debugging
```shell
kubectl port-forward pod <pod-name> <localhost-port>:<container-port>
```



## Deployments
```shell
# version history
kubectl rollout history deployment <deployment-name>

# revert to older version
kubectl rollout undo deployment <deployment-name> --to-revision=<revision-number>
```

### Update Strategy
```yaml
startegy:
  type: Recreate
```

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: '25%'
    maxUnavailable: '25%'
```


### Pod Creation
```shell
# create a pod with label 
kubectl run mypod --image=nginx:1.17 --port=80 --env="DB_USERNAME=admin" --labels="app=nginx,env=prod"

# create a temporary pod
> kubectl run busybox --image=busybox:1.36.1 --rm -it --restart=Never -- nslookup nginxsvc

# create a pod with command
# The command attribute overrides container ENTRYPOINT
# The args attribute overrides CMD instruanction of the image
# look at YAML to see args and command (following will only add args)
> kubectl run mypd --image=busybox --dry-run=client -o yaml > mypd.yaml -- /bin/sh -c "while true; do date; sleep 10; done;"
❯ kubectl apply -f mypd.yaml
```

### Pods env
```yaml
# no way to add env to existing pod (even by kubectl edit)
# you should recreate pod
# in the yaml for container spec add (or create it imperatively using --dry-run=client)
containers:
  - image: nginx
    env:
      - name: FOO
        value: bar

```

### Pod logs
```shell
# to watch logs
kubectl logs <podname> -f 

# to get past logs (deleted pods)
kubectl logs <podname> -p
```

### Pod execute
```shell
kubectl exec <podname> -- <command>
```

### Pod Volumes
add volume definition: `.spec.volumes[]`
mount volume in containers: `.spec.containers[].volumeMounts[]`
mapping between volume and volumeMounts -> `name` attribute

```yaml
# emtpyDir
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-pod
  name: nginx-pod
spec:
  volumes:
  - name: myapp-data
    emptyDir: {}
  containers:
  - image: nginx
    volumeMounts:
    - name: myapp-data
      mountPath: /myapp-data
    name: nginx-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

### Namespaces
```shell
# get list of namespaces
kubectl list ns

# create a namespace
kubectl create ns foo

# setting a namespace preference
kubectl config set-context --current --namespace=foo

# get pods in namespaces
kubectl get pods -n kube-system

kubectl get pods --namespace default

kubectl get pods --all-namespaces

# delete a namespace (cascadingly deletes all objects in the namespace)
kubectl delete ns foo
```

### jobs
```shell
# create job imperatively
k create job myjob --image=alpine:3.17.3 --dry-run=client -o yaml > job.yaml -- /bin/sh -c 'echo $date'

# print logs of all pods of the job
k get pods | awk '{print $1}' | grep -v "NAME" | xargs -n1 kubectl logs

```
To set `completions` and `parallelism` in  job's `.spec` set:
```yaml
spec:
  ...
  parallelism: 2
  completions: 5
  ...
```

### Cronjobs
```
# create a cronjob
kubectl create cronjob google-ping --image=busybox --schedule="* * * * *" --dry-run=client -o yaml > mycronjob.yaml -- curl google.com
```

To set number of failed or successful jobs to keep:
```yaml
spec:
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3
```

