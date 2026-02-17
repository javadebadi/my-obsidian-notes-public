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

### List nodes
To list all nodes in the cluster run:
```shell
kubectl get nodes
```
![[Pasted image 20260215231726.png]]

There are two kind of nodes in a Kubernetes cluster:
- control-plane: contains container like API server, scheduler, etc
- worker: contains container deployed for applications
## Cluster Components
A Kubernetes cluster itself is made up of many components that are deployed by Kubernetes itself. These components live in `kube-system` namespace.

### Kubernetes Proxy
There is a container called `kube-proxy` that should run on ALL nodes in the cluster.
**kube-proxy** is a network component that runs on **every node** in a Kubernetes cluster and is responsible for making **Services reachable** by routing traffic to the correct Pods. Without kube-proxy, Services stop working on that node because there is no component routing traffic to the right pods.

To check that kube-proxy actually exists in your local Minikube cluster run:
```shell
kubectl get pods -n kube-system -l k8s-app=kube-proxy

```
![[Pasted image 20260215233629.png]]

In this command `-n kube-system` is used to set the namespace to `kube-system` and the `k8s-app` is a label key used on many Kubernetes resources (usually pods). You can think of `k8s-app` label as a way to indicated the name of the component. Using we `k8s-app=kube-proxy` basically we are searching for a resource that's name is `kube-proxy`. 

### Kubernetes DNS
Kubernetes DNS is the internal name system that lets Pods and services find each other using names instead of IP addresses.
For example a pod may send request to `orders-service` and the Kubernetes DNS will change that to the ip-address:
```
orders-service → 10.96.0.15
```
To get pods where `CoreDNS` software is running (the name of software is `CoreDNS` but the label for that is `kube-dns`) run:
```shell
kubectl get services -n kube-system -l k8s-app=kube-dns
```
![[Pasted image 20260215235151.png]]

Also there is a Kubernetes DNS service running for Kubernetes DNS server:
```shell
kubectl get services -n kube-system -l k8s-app=kube-dns
```

![[Pasted image 20260215235454.png]]

The Kubernetes DNS service load balances requests to DNS servers and makes DNS reliable, faster, and resilient inside the cluster.

In the case of above, `10.96.0.10` is the IP address of the DNS service. All pods in the cluster use this IP address to make DNS queries.  They do not know the individual CoreDNS Pod IPs.

Every pod gets its DNS configuration automatically from the kubelet when it starts. This information is stored inside `/etc/resolve.conf` inside the pods:
```conf
nameserver 10.96.0.10
```
