The ConfigMap is an API resource to store site specific information. There are two use cases for ConfigMaps:
- Variable storage
- Configuration files storage up to a size of `1MiB`
If bigger amounts of data are needed, they should be stored in a Pod Volume.

To create ConfigMaps declaratively use `kubectl create cm` with these two options:
- `--from-literal key=value`
- `--from-env-file=/path/to/file`
And environment file with key-value pairs.

To use the env variable from ConfigMap in your application run:
```shell
kubectl set env --from=configmap/<cm-name> deployment/<deployment-name>
```

In Pod manifests here is where we can define env variables (either directly or using ConfigMap):
- `pod.spec.containers.env.name` defines the variable name
- `pod.spec.containers.env.valueFrom.configMapKeyRef.name` refers to name of the ConfigMap
- `pod.spec.containers.env.valueFrom.configMapKeyRef.key` defines the key in the ConfigMap from which the value must be set to variable

## ConfigMap from Literal
Here is a yaml template to create ConfigMap
```shell
kubectl create cm myappvars --from-literal=DB_PASSWORD=password --dry-run=client -o yaml > myappvars.yaml
```
The content of the file is 
```yaml
apiVersion: v1
data:
  DB_PASSWORD: password
kind: ConfigMap
metadata:
  name: myappvars
```
Now lets apply this template:
![[Pasted image 20260405012332.png]]
![[Pasted image 20260405012412.png]]

To use this variables in Pods (or pods spec of deployments) in Pod spec:
```yaml
- env:
  - name: POSTGRES_DB_PASSWORD
    valueFrom:
      configMapKeyRef:
        name: myappvars
        key: DB_PASSWORD

```
By using this in the pod and env variable named `POSTGRES_DB_PASSWORD` will be availabe in pod environment with value same as the value of `DB_PASSWORD` key in `myappvars` ConfigMap, i.e. `password` in case of above example.


## ConfigMap from a File
### Populating a volume with data stored in a ConfigMap
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox:1.27.2
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # Provide the name of the ConfigMap containing the files you want
        # to add to the container
        name: special-config
  restartPolicy: Never

```

When you create a ConfigMap using:

```
kubectl create configmap my-config --from-file=index.html
```

Kubernetes stores the **file content**, but also keeps the **filename as the key**.

#### What happens when you mount it as a volume?

If you mount that ConfigMap into a Pod, like:

```
volumeMounts:  - name: config-volume    mountPath: /usr/share/nginx/htmlvolumes:  - name: config-volume    configMap:      name: my-config
```

Then inside the container, you’ll get:

```
/usr/share/nginx/html/index.html
```

And yes — the contents of that file will be exactly the same as your original `index.html`.

#### Important nuance

- The **file name becomes the key**
- The **file content becomes the value**
- Each key is turned into a **file in the mounted directory**