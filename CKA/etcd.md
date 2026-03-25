
According to official ETCD [website](https://etcd.io/):
> A distributed, reliable key-value store for the most critical data of a distributed system

## Installation
To install ETCD in your computer look at https://etcd.io/docs/v3.6/install/

### Installation on MacOS
To install on MacOS run:
```shell
brew install etcd
```
![[etcd-install-macos.png]]

Then start etcd as a background process:
```shell
brew services start etcd
```
![[etcd-service-start.png]]

Then check the version of etcd:
```shell
etcd --version
```
![[etcd-version.png]]

The etcd has a client-server architecture:
- etcd -> server (daemon)
- etcdctl -> client (CLI tool)

To check version of etcdctl run:
```shell
etcdctl version
```
![[etcdctl-version.png]]

## CRUD
To insert a record in etcd database use `put` command:
![[etcdctl-put-key1-value1.png]]
To check the value is stored in database use `get`:
```shell
etcdctl get key1
```
![[etcdctl-get-key1.png]]
To update the value for that key in etcd use `put`:
![[etcdctl-put-key1-value2.png]]
To check the update actually happened run:
```shell
etcdctl get key1
```
![[etcdctl-get-key1-2.png]]

To delete a key from db use `del`:
```shell
etcdctl del key1
```
![[etcdctl-del-key1.png]]To check the key is actually deleted run:
```shell
ectdctl get key1
```
![[Pasted image 20260323232658.png]]


### Get list of all keys
To get list of all keys stored in etcd, let's first add few keys to the database:
```shell
etcdctl put name Albert
etcdctl put job Physicist
etcdctl put award Nobel
```

![[etcdctl-put-several-keys.png]]
Then run the following command to get list of all keys in etcd database:
```shell
etcdctl get "" --prefix --keys-only
```
![[etcdctl-get-all-keys.png]]

Or if you want to get list of all keys that start with letter `a` run:
```shell
etcdctl get "a" --prefix --key-only
```
![[etcdctl-get-list-of-keys-starting-with-a.png]]
## Comparison to Other Key-Value Databases
You may have heard about **Redis** which is another key-value database. Here is a table compares etcd with Redis:

| Feature        | Redis                                   | etcd                                      |
|----------------|-----------------------------------------|-------------------------------------------|
| Purpose        | Cache / real-time data                  | Distributed config / coordination          |
| Storage        | In-memory (RAM)                         | Persistent (disk)                          |
| Consistency    | Eventual (clustered)                    | Strong (Raft consensus)                    |
| Speed          | Very fast                               | Slower (consensus overhead)                |
| Data Model     | Rich structures (list, set, hash, etc.) | Simple key-value                           |
| Use Cases      | Caching, sessions, queues               | Config, service discovery, leader election |
| Reliability    | Medium                                  | High (source of truth)                     |

# etcd in Kubernets
Now, that we are familiar with etcd let's investigate etcd and its role in a Kubernetes cluster.
To continue we suggest to follow instruction [[Install K8 on VM|here]] to install a K8 on a VM in Google Cloud Platform

## Task 1: Locate etcd in a running cluster


> [!info]- Question 1.1: Confirm there is a pod for etcd running
> The `etcd` runs as a static pod in `kube-system` namespace. To confirm the pod for etcd is running run:
> ```shell
> kubectl get pods -n kube-system | grep "etcd"
> ```
>  ![[etcd-gcp-vm-get-pod-for-etcd.png]]
>

> [!info]- Question 1.2: Find the etcd static pod manifest
> Similar to other Kubernetes objects the etcd pod is also created by Kubernetes itself. The pod manifest is stored in `/etc/kubernetes/manifests/etcd.yaml`
> ```shell
> sudo snap install yq # install if you want to beautify yaml printed in the screen
> sudo cat /etc/kubernetes/manifests/etcd.yaml | yq
> ```
> ![[etcd-gcp-vm-find-etcd-manifest.png]]
> - **`--data-dir=/var/lib/etcd`** This is where etcd actually stores all cluster data on disk. When you do a restore later, you'll point etcd to a _new_ directory here.
> - **`--listen-client-urls=https://127.0.0.1:2379,https://10.162.0.3:2379`** This is the endpoint your `etcdctl` commands talk to. You'll use `https://127.0.0.1:2379` — the localhost one. Port 2379 is the standard etcd client port.
> - **`--cert-file` and `--key-file`** These are etcd's own TLS certificate and private key — how etcd identifies _itself_ to clients. You'll pass these to `etcdctl` as `--cert` and `--key`.
> - **`--trusted-ca-file`** The CA certificate that etcd uses to verify clients. You'll pass this to `etcdctl` as `--cacert`.
> - **`--peer-*` files** These are for etcd-to-etcd communication in a multi-node cluster. You don't need these for `etcdctl` commands — ignore them for now.
> So when you run `etcdctl` later, the pattern will always be:
> ```shell
> 
> ```

> [!info]- Question 1.3: Locate the data directory of etcd 
> The answer for question 1.2 shows the location of etcd data as `--data-dir=/var/lib/etcd`
> ```shell
> sudo ls -alh /var/lib/etcd
> ```
> ![[etcd-gcp-vm-data-dir.png]]
>

> [!info]- Question 1.4: Find the etcd certificates
> The answer for question 1.2 shows the location of etcd certificates is `/etc/kubernetes/pki/etcd`
> ```shell
> sudo ls -alh /var/kubernetes/pki/etcd
> ```
> ![[etcd-gcp-vm-find-certificates-location.png]]
>
> Note that in next step you will need path of `ca.crt`, `server.crt` and `server.key` to use `etcdctl` to access etcd database.

## Task 2: Run your first etcdctl command

> [!info]- Question 2.1: Install etcdctl client
> First we check if etcdctl is installed
> ```shell
> etcdctl version
> ```
> ![[etcd-gcp-vm-etcdctl-check-version.png]]
> If it is not installed you should install it:
> ```shell
> 	sudo apt-get install etcd-client
> ```
> ![[etcdctl-gcp-vm-install.png]]
> Then check the version again:
> ```shell
> etcdctl --version
> ```
> 
> ![[etcdctl-gcp-vm-version-2.png]]
> To change API version (which we usually need) run:
> ```shell
> ETCDCTL_API=3 etcdctl version
> ```
> ![[etcdctl-gcp-vm-api-3-version.png]]
> 


> [!info]- Question 2.2: Check cluster health using etcdctl
> To check the health you should run
> ```shell
> sudo ETCDCTL_API=3 etcdctl endpoint health \
> --cacert=/etc/kubernetes/pki/etcd/ca.crt \
> --cert=/etc/kubernetes/pki/etcd/server.crt \
> --key=/etc/kubernetes/pki/etcd/server.key \
> 	--endpoints=https://127.0.0.1:2379
> ```
> ![[etcdctl-gcp-vm-etch-health-check.png]]



> [!info]- Question 2.3: Check etcd status using etcdctl and interpret the results
> To check the status you should run
> ```shell
> sudo ETCDCTL_API=3 etcdctl endpoint status \
> --cacert=/etc/kubernetes/pki/etcd/ca.crt \
> --cert=/etc/kubernetes/pki/etcd/server.crt \
> --key=/etc/kubernetes/pki/etcd/server.key \
> --endpoints=https://127.0.0.1:2379
> ```
> ![[etcdctl-gcp-vm-check-status.png]]
> 
> - **`ID`** — unique identifier for this etcd member. In a multi-node cluster each member has a different ID.
> - **`VERSION`** — etcd version `3.5.24`. Useful to know when the exam asks you to confirm etcd is running.
> - **`DB SIZE`** — `3.6 MB` is the current size of the etcd database on disk. After a restore this will change — a good way to confirm the restore worked.
> - **`IS LEADER`** — `true`. In a single-node cluster this is always true. In a 3-node HA cluster only one member is leader at a time — this is the one that processes all writes.
> - **`RAFT TERM`** — `2`. Raft is the consensus algorithm etcd uses. The term increments every time a new leader election happens. `2` means there's been one election since the cluster started (term 1 = bootstrap, term 2 = first normal operation).
> - **`RAFT INDEX`** — `25796`. This is the total number of operations ever committed to etcd. Every time Kubernetes creates, updates, or deletes an object, this number goes up. After a restore from an older snapshot, this number will be lower — which is expected and correct.
> 
> 

## Task 3: Take Snapshot of etcd Database


> [!info]- Question 3.1: Create a sample deployment and create a snapshot of etcd database
>First let's create a deployment
>```shell
>kubectl create deployment deployment-foo --image=nginx:alpine --replicas=3
>```
>![[etcd-gcp-vm-sample-deployment.png]]
>To create snapshot run:
>```shell
>sudo ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
--cert=/etc/kubernetes/pki/etcd/server.crt
>```
>![[etcd-gcp-vm-snapshot-save.png]]
>

> [!info]- Question 3.2: Verify the snapshot is valid
> To verify the created snapshot is valid run
> ```shell
>sudo ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-out=table
>```
>![[etcd-gcp-vm-snapshot-verify.png]]

