
## Init Container
An init container is a special case of a multi-container Pod where the init container runs to completion before the main container is started.
If the init container fails, then main container will never be started.

[Here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/) is a good example of init container from Kubernetes documentation:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  # These containers are run during pod initialization
  initContainers:
  - name: install
    image: busybox:1.28
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - http://info.cern.ch
    volumeMounts:
    - name: workdir
      mountPath: "/work-dir"
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}


```

Watch [this video](https://drive.google.com/file/d/1bes2N154OgbHwJwYAQCwrz3bIix1Llxf/view?usp=drive_link)

Also if you look at the documentation there are good examples showing other use-cases of init containers using linux `until` command that constantly check if the condition is ready for the main container to start.
Note that after init container is done its job it will be terminated:
```shell
kubectl describe pod init-demo
```
![[Pasted image 20260401181454.png]]


## Sidecar Containers
A **sidecar** container is an `initContainer` that has `restartPolicy` field set to `Always`. The sidecar container will be started before the main pod is started and is typically used to repeatedly run a command.
The sidecar container must complete once before the main Pod is started. 
[Here](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/) is a YAML for sidecar container example from Kubernetes documentation:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: alpine:latest
          command: ['sh', '-c', 'while true; do echo "logging" >> /opt/logs.txt; sleep 1; done']
          volumeMounts:
            - name: data
              mountPath: /opt
      initContainers:
        - name: logshipper
          image: alpine:latest
          # Setting restartPolicy: Always makes this a sidecar container.
          restartPolicy: Always
          command: ['sh', '-c', 'tail -F /opt/logs.txt']
          volumeMounts:
            - name: data
              mountPath: /opt
      volumes:
        - name: data
          emptyDir: {}
```


## restartPolicy
The pod `restartPolicy` spec determines what will happen if a container that is managed by pod crashes. The `restartPolicy` does not affect the state of the entire pod. If a pod is killed or stopped that parameter will no restart the pod itself.
[Watch this video](https://drive.google.com/file/d/1sQWH-G8h5mpYIkMnzGjgeTDm-2Wfv0RH/view?usp=drive_link)


Note that `restartPolicy` is defined on pod level not container level. Therefore all containers (main, init container, sidecar container) shared that value. But note that recently Kubernetes has introduced `restartPolicy` in `initCotainer` spec as we showed above for sidecar containers. This is different from pod level `restartPolicy`.

### 1. Pod-level `restartPolicy`

spec:  
  restartPolicy: Always | OnFailure | Never

- Applies to:
    - all **regular containers** (`containers:`)
    - all **init containers by default**

---

### 2. Per-container `restartPolicy` (⚠️ ONLY for initContainers)

You can now do:

initContainers:  
  - name: sidecar  
    image: busybox  
    restartPolicy: Always

But:

> ❗ This is **NOT a general per-container feature**

---

### 🚫 What is NOT allowed

You **cannot** do this:

containers:  
  - name: app  
    image: my-app  
    restartPolicy: Always   # ❌ INVALID

Regular containers still **do NOT support** their own restart policy.
