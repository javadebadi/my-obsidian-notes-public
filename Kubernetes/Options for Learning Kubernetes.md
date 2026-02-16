If you’re new to Kubernetes, the best way to learn is to use Kubernetes-as-a-Service (KaaS) in cloud providers like AWS, GCP, or Azure. For instance, Google offers Google Kubernetes Engine (GKE), a managed KaaS service. However, it can be costly. To learn Kubernetes using GKE for free, take a course and lab in [Google Cloud Skills Boost](https://www.skills.google/). The website provides free credits to learn concepts and gain hands-on experience with GKE.


| Learning Method                                  | Cost      | Learning Curve | Multi-node |
| ------------------------------------------------ | --------- | -------------- | ---------- |
| Google Cloud - Google Kubernetes Engine          | Expensive | Fast           | True       |
| skills.google website - Google Kubernetes Engine | Free      | Fast           | True       |
| Locally - Minikube                               | Free      | Slow           | False      |


## Install Kubernetes Locally - Minikube
To work with K8 cluster locally you can use `minikube` .

### Install Minikube on MacOS
To install Minikube on MacOS run:
```bash
brew install minikube
```

To check the `minikube` is installed successfully run:
```bash
minikube version
```
![[Pasted image 20260214234253.png]]
Note that `minikube` creates a single node K8 cluster which is very different from multi-node cluster that are used in production.

To create a local cluster run:
```bash
minikube start
```
![[Pasted image 20260214235244.png]]

To stop the cluster once you finished working run:
```shell
minikube stop
```
And to remove the cluster completely run:
```shell
minikube delete
```

## The Kubernetes Client
The official Kubernetes client is `kubectl` a CLI tool to interact with the Kubernetes API.
### Check Cluster Version
To check version of `kubectl` and version of Kubernetes cluster API run:
```shell
kubectl version
```
![[Pasted image 20260215000018.png]]
The results has 3 information:
- Client Version: version of the `kubectl`
- Kustomize Version: version of the Kustomize bundle inside the `kubectl`
- Server Version: version of the Kubernetes in the cluster you are connected to

### Check Cluster Health
To check overall health of the cluster run:
```shell
kubectl get componentstatuses
```
![[Pasted image 20260215000912.png]]
Here we see all components are healthy:
- etcd: storage that stores state of cluster (all objects in the cluster)
- scheduler: responsible for placing pods in nodes
- controller-manager: responsible for running multiple controllers in the cluster to check different things