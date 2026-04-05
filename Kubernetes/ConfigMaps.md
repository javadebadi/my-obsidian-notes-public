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



