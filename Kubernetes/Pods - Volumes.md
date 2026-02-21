Read [[Pods]] before reading this section.

## Mounting
When you plug a USB drive into Linux,  you need to run a command to attach it so you can access the files. The command is:
```shell
mount /dev/sdb1 /mnt/usb
```
This command has two parts:
- storage device: `/dev/sdb1`
- path in you os where device files will appear: `/mnt/usb`

We call this action you mount a disk to `/mnt/usb` and by this action you tell the Linux system show the content of this disk (`/dev/sdb1`) inside this folder (`/mnt/usb`).

Here is the definition of mount you should remember:
**Mount: Attach one filesystem into another filesystem at a specific folder**

```
/
├── home
├── dev
├── var
├── etc
└── mnt   ← you attach things here
```



## Minikube Mounting
To mount a path in minikube to a path in your machine use `minikube mount <OUTSIDE>:<INSDIE>`
```shell
minikube mount /Users/javad/Desktop/myapp-data:/myapp-data
```

You don't need to mount a minikube folder into your OS folder. However, if you want to check the files you create in K8 containers in your minikube cluster actually stored somwhere in your OS then you can do this.
![[Pasted image 20260220082751.png]]

## Persistent Volumes
The pods are ephermal and once they are deleted, all containers and the data inside it will be deleted. If you need to persist data Kubernetes has **Volumes** which is a way to solve this problem.
All containers in a pod are required to mount all volumes defined in the pod.
To add a volume you need to change two sections in Pod's manifest:
- add `volumes` in `spec`:
	- ```
	  volumes:
	    - name: "myapp-data"
	      hostPath:
	        path: "/myapp-data"
	  ```
- mount volumes into containers in containers `volumeMounts`
	- ```
	  volumeMounts:
	    - name: "myapp-data"
	      mountPath: "/data"
	  ```

Here is a Pod manifest with volumes:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  volumes:
    - name: "myapp-data"
      hostPath:
        path: "/myapp-data"
  containers:
    - image: nginx:alpine
      name: myapp-nginx
      volumeMounts:
        - name: "myapp-data"
          mountPath: "/data"
      ports:
      - containerPort: 80
        protocol: TCP
        name: http
      livenessProbe:
        exec:
          command:
            - curl
            - http://localhost:80
        initialDelaySeconds: 5
        periodSeconds: 10
```

Watch [this video](https://drive.google.com/file/d/1bFKcT2_NTTt0NSBsTfUAn2R_5cM_l4td/view?usp=drive_link)



## What we know so far
Here is a manifest that defines a pod with every feature we have learned so far

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  volumes:
    - name: "myapp-data"
      hostPath:
        path: "/myapp-data"
  containers:
    - image: nginx:alpine
      name: myapp-nginx
      volumeMounts:
        - name: "myapp-data"
          mountPath: "/data"
      ports:
      - containerPort: 80
        protocol: TCP
        name: http
      resources:
        requests:
          cpu: "500m"
          memory: "128Mi"
        limits:
          cpu: "1000m"
          memory: "256Mi"
      livenessProbe:
        exec:
          command:
            - curl
            - http://localhost:80
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 5
        timeoutSeconds: 1
        periodSeconds: 10
        failureThreshold: 3
```


## Different Kind of Volumes
| Volume Type                     | Purpose / Problem Solved                                         | Typical Use Cases                                  | Mental Model / Analogy                                  |
| ------------------------------- | ---------------------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------- |
| **emptyDir**                    | Temporary sharing of data between containers in the same Pod     | Cache, logs, scratch space between containers      | Like a shared `/tmp` folder for processes in a Pod      |
| **hostPath**                    | Direct access to the node's filesystem (local debugging/testing) | Access host logs, Docker socket, local experiments | Like giving a process access to `/var/log` on host      |
| **PersistentVolume (PV) + PVC** | Persistent storage that survives Pod restarts                    | Databases, file storage, user uploads              | Like attaching an external USB drive to a process       |
| **ConfigMap / Secret**          | Inject configuration or secrets into containers at runtime       | app.yaml, API keys, SSL certificates               | Like giving a process a read-only config folder         |
| **Cloud / Networked volumes**   | Shared persistent storage across multiple nodes or Pods          | NFS, AWS EBS, GCE Persistent Disk, Ceph            | Like a network drive accessible from multiple computers |
