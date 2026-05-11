
A DaemonSet ensures a copy of a Pod is running on every node in a cluster. It is useful for when you want to have an agent running on all nodes.
Use cases:
- Running log collectors
- Running monitoring agents
- Anything that should be run on every node

DaemonSet controller creates a Pod on each node that does not run a copy of that Pod. This is also very useful for clusters with auto-scalar features where nodes are dynamically added or removed.

To create a DaemonSet imperatively use `create deployment` and then modify the manifst.
```shell
kubectl create deployment nginx-app --image=nginx:1.15 --replicas=1 --dry-run=client -o yaml > daemon.yaml
```

Now edit the `daemon.yaml` file by
- Remove the `replicas` field
- Remove the `strategy` field

It will be something like this:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: nginx-app
  name: nginx-app
spec:
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - image: nginx:1.15
        name: nginx
        resources: {}
```

Then apply the manifest:
```shell
kubectl apply -f daemon.yaml
```

When you run in terminal you will see:
```shell
❯ kubectl apply -f daemon.yaml
daemonset.apps/nginx-app created
```

To check the DaemonSet run:
```shell
kubectl describe daemonset nginx-app
```

The result is:
```shell
❯ kubectl describe daemonset nginx-app
Name:           nginx-app
Namespace:      default
Selector:       app=nginx-app
Node-Selector:  <none>
Labels:         app=nginx-app
Annotations:    deprecated.daemonset.template.generation: 1
Desired Number of Nodes Scheduled: 1
Current Number of Nodes Scheduled: 1
Number of Nodes Scheduled with Up-to-date Pods: 1
Number of Nodes Scheduled with Available Pods: 1
Number of Nodes Misscheduled: 0
Pods Status:  1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=nginx-app
  Containers:
   nginx:
    Image:         nginx:1.15
    Port:          <none>
    Host Port:     <none>
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Events:
  Type    Reason            Age    From                  Message
  ----    ------            ----   ----                  -------
  Normal  SuccessfulCreate  8m45s  daemonset-controller  Created pod: nginx-app-csdgp
```

## Updating a DaemonSet
To the `spec` of the DaemonSet add:
```yaml
spec:
  ...
  minReadySeconds: 30
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  ...

```

The `minReadySeconds` determines how much a Pod must be ready before moving to update the next Pod.

## Limiting to Specific Nodes

Sometimes you want to run a version of Pod on selected nodes. For example, on nodes which have GPU. For this you need to label those nodes:
```shell
kubectl label node <that-node> gpu=true
```
To check list of nodes with `gpu=true` label run:
```shell
kubectl get nodes -l gpu=true
```

### Node Selector
Node selectors are defined as part of the Pod spec. To limit this app to only be deployed on nodes with `gpu=true` here is the manifest:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: nginx-app
  name: nginx-app
spec:
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      nodeSelector:
        gpu: "true"
      containers:
      - image: nginx:1.15
        name: nginx
        resources: {}
status: {}
```
Note that `nodeSelector` is added as part of Pod spec.
In case of local minikube since you don't have any `gpu=true` labels on nodes (one node actually) then you should see the Pod was destroyed.
```shell
❯ kubectl get daemonset
NAME        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx-app   0         0         0       0            0           gpu=true        2d11h
```
