Service discovery tool helps to find which processes are listening at which IP address for which service.
Domain Name Service (DNS) is a service discovery solution.


Let's describe what is the problem and then why we need service discovery.
In Kubernetes networking, every pod get its own IP address (which is different from node IP address). Pods can talk to each other directly using these IP addresses.

To distinguish pod IP from node IP here are some features to let you know how they are different:
- Pod IP
	- Unique in cluster
	- Temporary
	- Changes if pod restarts
- Node IP
	- Stable
	- Represents the machine

Now if you have 3 replicas of an application you will have 3 IP address for 3 pods. Assume you have a FastAPI deployment with 3 replicas, then you will have 3 IP addresses such as:
- 3 FastAPI Pods
	- 10.244.1.5
	- 10.244.1.6
	- 10.244.2.3

If another service wants to call your FastAPI app which IP address it should use?
Here is the problem. Pods are dynamic, they constantly being destroyed and new pods will have different IP addresses. Clients need something **stable** to connect to pods. Kubernetes uses `Service` to solve this.

A Service:

- Gets a stable virtual IP (ClusterIP)
    
- Gets a DNS name
    
- Automatically routes traffic to matching Pods
    
- Load balances across them

When a client Pod wants to talk to another application (e.g., FastAPI), it does **not** use a Pod IP. Instead, it uses the Service name: http://fastapi-app

#### Step 1 — DNS Resolution

- The client Pod sends a DNS query to **CoreDNS**
    
- CoreDNS resolves `fastapi-app`
    
- DNS returns the **ClusterIP** of the Service  
    Example: `10.96.15.23`
    

At this point, the client only knows the ClusterIP.

---

#### Step 2 — Traffic Interception by kube-proxy

- The client sends the request to `10.96.15.23`
    
- On the node, **iptables/IPVS rules** (managed by kube-proxy) match this ClusterIP
    
- kube-proxy rewrites the destination IP to one of the Pod IPs
    

Example Pod IPs:

- 10.244.1.5
    
- 10.244.1.6
    
- 10.244.2.3
    

kube-proxy chooses one (usually round-robin, a load balancing engine).

# Setup 
Let's first create some Kubernetes objects for experimentation.
Kubernetes has a concept called deployment which you can think of it as an instance of microservice. Each deployment can have many pods and they will be managed by Kubernetes.

Let's create two deployments.
```shell
kubectl create deployment deployment-foo --image="nginx:alpine"

kubectl scale deployment deployment-foo --replicas=3
```

```shell
kubectl create deployment deployment-bar --image="nginx:alpine"

kubectl scale deployment deployment-bar --replicas=2
```

After creating deployments we can check and see we have 5 pods:
![[Pasted image 20260304091054.png]]

And we use `kubectl expose` to create a service:
```shell
kubectl expose deployment deployment-foo

kubectl expose deployment deployment-bar
```

Then we can check services `kubectl get svc`
![[Pasted image 20260304091618.png]]

# How the test DNS inside the cluster

Run a temporary test pod
```shell
kubectl run dns-test --image=busybox:1.35 --rm -it -- sh
```

Inside the pod look for the service:
```shell
nslookup deployment-foo
```
![[Pasted image 20260304065049.png]]

I can see the **A record**:
```yaml
Name: deployment-foo.default.svc.cluster.local
Address: 10.98.195.23
```

You can also lookup for full DNS name:
![[Pasted image 20260304065547.png]]


# Readiness Check
One of the things the service object can do is to track which pods are ready using **readiness check**.

# Access to Pods from Outside of the Cluster
Lets's first delete the created (exposed) service (or you can edit it using `kubectl edit` but we try to make things easier):
```shell
kubectl delete service deployment-foo
```
![[Pasted image 20260307131532.png]]

and then re-expose the deployment using `--type=NodePort`
```shell
kubectl expose deployment deployment-foo --type=NodePort
```
![[Pasted image 20260307131650.png]]

	Now use `kubectl describe` to get details of the service
```shell
kubectl describe svc deployment-foo
```
![[Pasted image 20260307131735.png]]

As you see the information for service:
- NodePort: 30597
- Pod Endpoint: 10.244.0.36:80

To access this first we need the ip of the cluster. In case of minikube you can get the ip using
```shell
minikube ip
```
![[Pasted image 20260307132309.png]]

For some minikube based on the driver type you can access the service using `http://<minikube_ip>:port` but if that is not reachable try this:

```shell
minikube service deployment-foo
```
![[Pasted image 20260307132758.png]]

You see the `http://<minikube_ip>:port` which in this case is `http://192.168.49.2:30597` us accessible using tunnel in this url: `http://127.0.0.1:60174`
![[Pasted image 20260307132946.png]]

# Load Balancer
So far we are familiar with these service types:
- NodePort
- ClusterIP

There other type of services is:
- LoadBalancer
which you can use if your cluster is configured to integrate with external load balancer.