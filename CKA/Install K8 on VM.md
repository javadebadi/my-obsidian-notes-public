Here I show how you can install K8 on a VM in Google Cloud Platform. You can do this to create a lab which will be similar to the environment you will work in CKA exam for example.

# Installation
To install Kubernetes on a VM there are two steps in GCP:

## Create a VM
Assuming you have GCP account and logged with a user that has permissions to create VMs, run the following command to create a VM:
```shell
gcloud compute instances create my-k8-cluster \
--zone=northamerica-northeast1-a \
--machine-type=e2-medium \
--image-family=ubuntu-2204-lts \
--image-project=ubuntu-os-cloud \
--boot-disk-size=10GB \
--tags=k8s-practice
```
![[gcp-create-vm-for-k8.png]]

You can also view the created vm in GCP console: https://console.cloud.google.com/compute/instances?project=main-k8 (replace `project=main-k8` with your own GCP project)
![[Pasted image 20260324170720.png]]

Then SSH in:
```shell
gcloud compute ssh my-k8-cluster --zone=northamerica-northeast1-a
```
![[gcp-vm-ssh-in-entry.png]]

So far, we have created a VM and SSH into that VM. Next, we have to install Kubernetes.
## Install Kubernetes

There are several steps to install Kubernetes in a VM.


### Step 1: Disable Swap
Kubernetes wants to be the one in control of memory management, not the OS. Swap is the OS doing its own thing behind Kubernetes's back. So it should be disabled.

```shell
sudo swapoff -a          # disables swap right now, until next reboot
sudo sed -i '/ swap / s/^/#/' /etc/fstab   # comments it out so it stays off after reboot
```
If the swap is on, then in laters steps when you run `kubeadm init` you will see an error like this:
```shell
[ERROR Swap]: running with swap on is not supported. Please disable swap
```

### Step 2: Load Required Kernel Modules
Kubernetes depends on few Linux modules that should be loaded at start of the systems.
The `/etc/modules-load.d/` is a special directory that the `systemd-modules-load` service watches. On every boot, systemd reads all `*.conf` files in that directory and loads the kernel modules listed in them.
Now for Kubernetes to work the `overlay` and `br_netfilter` should be loaded at the start of they system. For this run:

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
That command will cause the modules to be loaded in system reboot. But what about the current session? To load those modules in current session without reboot run:
```shell
sudo modprobe overlay
sudo modprobe br_netfilter
```
![[gcp-vm-modprobe.png]]

#### What are the two modules for?

**`overlay`** — enables OverlayFS. This is the filesystem trick that containers use to appear to have their own isolated filesystem, while actually sharing the host's files underneath. Without it, containerd can't create or run containers.

**`br_netfilter`** — enables iptables to inspect traffic crossing network bridges. When pods communicate with each other or with services, traffic goes through virtual network bridges. Without this module, Kubernetes networking rules (written in iptables) are invisible to that bridged traffic, so pod networking breaks.

To check the modules are loaded run:
![[gcp-vm-lsmod-check-modules-loaded.png]]

### Step 3:  Set the Kernel Networking Pattern
```shell
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

`sysctl` controls Linux kernel parameters at runtime. This command writes three networking rules into a config file and activates them.

**`net.ipv4.ip_forward = 1`** Allows the Linux kernel to forward packets from one network interface to another. Without this, your node receives a packet destined for a pod on another node and just drops it — "not my address, not my problem." With it on, the node acts as a router and forwards it along. Essential for any pod-to-pod communication across nodes.

**`net.bridge.bridge-nf-call-iptables = 1`** By default, traffic passing through a Linux bridge (which is how container networking works internally) bypasses `iptables` entirely. Kubernetes uses `iptables` heavily — for service routing, network policies, NAT. This setting forces bridged traffic to also go through `iptables` so Kubernetes can intercept and manage it.

**`net.bridge.bridge-nf-call-ip6tables = 1`** Same thing but for IPv6 traffic.

### Step 4: Install Containerd (the Container Runtime)
```shell
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Step 5: Install kubeadm, kubelet, kubectcl
```shell
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Step 6: Bootstrap the Cluster
```shell
sudo apt-get install -y conntrack
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
### Step 7: Set up kubectl Access
```shell
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 8: Install a CNI plugin (needed for the node to go ready)
```shell
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

### Step 9: Verify
```shell
kubectl get nodes
```
![[gcp-vm-verify-kubectl-get-nodes.png]]
