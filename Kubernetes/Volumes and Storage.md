```table-of-contents
title: 
style: nestedList # TOC style (nestedList|nestedOrderedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
include: 
exclude: 
includeLinks: true # Make headings clickable
hideWhenEmpty: false # Hide TOC if no headings are found
debugInConsole: false # Print debug info in Obsidian console
```
A **Persistent Volume** is a specific Kubernetes resource that defines the storage.
Pods connect to a Persistent Volume resource using the **PersistentVolumeClaim** API resource.
The advantage of using Persistent Volume is decoupling. Pods don't care about storage type they just use what is available for them. PersistentVolumeClaim is request for storage like I need 5 GB of storage.

## Persistent Volume and Persistent Volume Claim
[Here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/) is a yaml config to create a Persistent Volume:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"

```
and if we apply this manifest
```shell
❯ kubectl apply -f task-pv-volume.yaml
persistentvolume/task-pv-volume created
❯ k get pv task-pv-volume -o wide
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE   VOLUMEMODE
task-pv-volume   1Gi        RWO            Retain           Available           manual         <unset>                          11s   Filesystem
```


> [!NOTE]- Screenshot
![[Pasted image 20260402180829.png]]

And to create `PersistentVolumeClaim`:
```yaml
apiVersion: v12
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi

```

> [!NOTE]- Screenshot
![[Pasted image 20260402182936.png]]


And then we can create a pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage

```

**`storageClassName`** is just an arbitrary label — `manual` has no special meaning. Its only purpose is to match a PV to a PVC. The string can be anything, as long as PV and PVC have the exact same value. Omitting it sets it to `""`, which has its own matching rules.

**Three fields must match for a PV/PVC to bind:** `storageClassName` (exact string), `accessModes` (PVC mode must be offered by PV), and `capacity` (PVC request ≤ PV size). All three must align — any mismatch leaves the PVC stuck in `Pending`.

---

**Q: `storageClassName` is set on the PVC but no PV matches — what happens?**

The PVC stays in `Pending` indefinitely. No binding occurs. Debug with `kubectl describe pvc <name>` — it will say something like _"no persistent volumes available for this claim."_

There are two ways of storage provisioning in real life:
- **Static provisioning:** you write the PV → PVC binds to it
- **Dynamic provisioning:** you write a StorageClass → PVC triggers it → StorageClass generates the PV → PVC binds to the auto-generated PV
For example in GCP:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd   # GCP's provisioner
parameters:
  type: pd-ssd
```

If you wanted slower but cheaper disk:
```yaml
parameters:
  type: pd-standard  # HDD
```

**If you use GKE** — StorageClasses come pre-installed. Google sets them up for you out of the box. You can see them with `kubectl get storageclass` immediately after cluster creation.

**If you built your own cluster on GCP VMs (kubeadm)** — nothing is pre-installed. You have to manually install the GCP CSI driver, which then makes the StorageClass available. It's a real setup process.

## Persistent Volume Options

### VolumeMode
```shell
❯ k explain pv.spec.volumeMode
KIND:       PersistentVolume
VERSION:    v1

FIELD: volumeMode <string>
ENUM:
    Block
    Filesystem

DESCRIPTION:
    volumeMode defines if a volume is intended to be used with a formatted
    filesystem or to remain in raw block state. Value of Filesystem is implied
    when not included in spec.

    Possible enum values:
     - `"Block"` means the volume will not be formatted with a filesystem and
    will remain a raw block device.
     - `"Filesystem"` means the volume will be or is formatted with a
    filesystem.
```

**`volumeMode`** controls whether Kubernetes gives your app a formatted disk (with an OS-managed index of files and folders) or a raw disk (just numbered blocks, no structure). The default is `Filesystem` and you will almost never change it.

A disk at the hardware level is just a sequence of dumb numbered blocks — no files, no folders. A filesystem is software that sits on top and maintains an index: "file `main.py` lives in blocks 142, 143, 501." Formatting a disk means writing that empty index onto it. `Filesystem` mode lets the OS manage that index. `Block` mode skips the index entirely and hands the raw blocks straight to your app.

|                       | Filesystem                                                                           | Block                                                                          |
| --------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| Analogy               | A library with a catalogue — you ask for a book by name and the librarian fetches it | A warehouse with no catalogue — you manage your own map of where everything is |
| Who manages the index | The OS (ext4, APFS, NTFS)                                                            | Your application itself                                                        |
| Your app sees         | Files and folders at a path like `/data`                                             | Raw bytes, no structure                                                        |
| Who uses it           | Every normal application                                                             | High-performance databases (Oracle, some PostgreSQL configs)                   |
| In your manifest      | Omit it — defaults to `Filesystem`                                                   | `volumeMode: Block`                                                            |


### Access Modes
**`accessModes`** defines how many nodes (or pods) can mount a volume simultaneously and whether they can write. It is a hardware constraint first — the mode you declare must match what your storage backend is physically capable of.

Restriction is per **node** not per pod — except for RWOP which tightens it to per pod.

|Mode|Short|Restriction|Write|Typical backend|
|---|---|---|---|---|
|`ReadWriteOnce`|RWO|1 node|yes|GCE PD, EBS, most block disks|
|`ReadOnlyMany`|ROX|many nodes|no|NFS, shared config|
|`ReadWriteMany`|RWX|many nodes|yes|NFS, GCP Filestore, CephFS|
|`ReadWriteOncePod`|RWOP|1 pod|yes|any backend supporting RWO|

**Key facts to remember:**

- `RWO` is what you use 90% of the time
- `RWX` is not a setting you can just turn on — block disks physically cannot mount on two nodes at once
- `RWOP` is `RWO` but stricter — use it when you need to guarantee only one pod writes even during rolling updates where two pods briefly coexist on the same node
- A PVC's accessMode must be a subset of the PV's accessModes for binding to succeed

### Reclaim Policy
**`reclaimPolicy`** controls what happens to the PV and its underlying storage when the PVC is deleted.

|Policy|What happens|Use when|
|---|---|---|
|`Retain`|PV and data kept, PV goes to `Released` — not claimable again without manual intervention|Terraform manages the disk, or data must survive cluster deletion|
|`Delete`|PV and actual underlying storage (e.g. GCE disk) deleted|Kubernetes manages the disk, ephemeral data|
|`Recycle`|Data wiped, PV made available again|Deprecated — don't use|

**Key decisions:**

- Should this data outlive the Kubernetes cluster? → `Retain`
- Is this ephemeral data tied to the app lifecycle? → `Delete`
- Terraform-managed infrastructure? → always `Retain` to avoid state drift

Default for manually created PVs is `Retain`. Default for dynamically provisioned PVs is `Delete`.

- **Static provisioning** (you manually write PV) → default `Retain`
- **Dynamic provisioning** (PVC + StorageClass, no manual PV) → default `Delete`

## Persistent Volume Claim
**PersistentVolumeClaim** is a request for storage. It doesn't care about the underlying storage type or infrastructure — it just declares requirements and Kubernetes finds a matching PV automatically.

**Matching** is based on `storageClassName`, `accessModes`, and `capacity` — not by name, keeping the PVC decoupled from any specific PV.

**Binding is exclusive** — one PV to one PVC. Once bound, that PV is fully reserved even if the PVC uses only a fraction of the capacity.

**Kubernetes picks the smallest PV** that satisfies the request to avoid wasting storage.


## Storage Class
In Google Cloud Platform, I execute `kubectl get storageclass` and here is what we see there
```shell
sage@cloudshell:~ (main-k8)$ kubectl get storageclass
NAME                        PROVISIONER                        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
dynamic-rwo                 pd.csi.storage.gke.io              Delete          WaitForFirstConsumer   true                   15d
enterprise-multishare-rwx   filestore.csi.storage.gke.io       Delete          WaitForFirstConsumer   true                   94d
enterprise-rwx              filestore.csi.storage.gke.io       Delete          WaitForFirstConsumer   true                   94d
gcsfusecsi-checkpointing    gcsfuse.csi.storage.gke.io         Delete          Immediate              false                  15d
gcsfusecsi-serving          gcsfuse.csi.storage.gke.io         Delete          Immediate              false                  15d
gcsfusecsi-training         gcsfuse.csi.storage.gke.io         Delete          Immediate              false                  15d
parallelstore-rwx           parallelstore.csi.storage.gke.io   Delete          WaitForFirstConsumer   false                  94d
premium-rwo                 pd.csi.storage.gke.io              Delete          WaitForFirstConsumer   true                   94d
premium-rwx                 filestore.csi.storage.gke.io       Delete          WaitForFirstConsumer   true                   94d
regional-rwx                filestore.csi.storage.gke.io       Delete          WaitForFirstConsumer   true                   94d
standard                    kubernetes.io/gce-pd               Delete          Immediate              true                   94d
standard-rwo (default)      pd.csi.storage.gke.io              Delete          WaitForFirstConsumer   true                   94d
standard-rwo-regional       pd.csi.storage.gke.io              Delete          WaitForFirstConsumer   true                   94d
standard-rwx                filestore.csi.storage.gke.io       Delete          WaitForFirstConsumer   true                   94d
zonal-rwx                   filestore.csi.storage.gke.io       Delete          WaitForFirstConsumer   true                   94d
```

> [!NOTE]- Screenshot
![[gcp-kubectl-get-storageclass.png]]

- `pd.csi.storage.gke.io` → GCE Persistent Disk ✓
- `filestore.csi.storage.gke.io` → GCP Filestore — this is **network file storage (NFS)**, not block. Notice all filestore StorageClasses are `rwx` — many nodes can mount simultaneously
- `gcsfuse.csi.storage.gke.io` → you're right, GCS (object storage) ✓

The interesting one is `gcsfuse` — you're correct that GCS is object storage. `gcsfuse` is a tool that **mounts a GCS bucket as if it were a filesystem**. It's a bridge — behind the scenes it's still object storage, but your app sees a normal path.
`accessModes` on the PV defines what's _physically possible_. `RWX` means the underlying storage (Filestore/NFS) physically supports multiple nodes mounting simultaneously.

The PVC binding rule (one PV to one PVC) still holds — only one PVC can claim that PV. But that one PVC can be used by **many pods across many nodes** simultaneously because the underlying storage supports it.

So:

- One PV → one PVC (always)
- One PVC → many pods (if accessMode is RWX)


**StorageClass** is a factory for PVs. Instead of manually creating PVs, you define a StorageClass once and Kubernetes creates PVs on demand when a PVC requests it.

**Key fields:**

- `provisioner` — which plugin creates the storage (e.g. `pd.csi.storage.gke.io` for GCE disk)
- `reclaimPolicy` — default is `Delete` for dynamic provisioning
- `volumeBindingMode` — `Immediate` creates storage right away, `WaitForFirstConsumer` waits until a pod is scheduled (important for zonal storage like GCE disks)
- `allowVolumeExpansion` — whether you can resize the PVC later

**The full chain:**

StorageClass → PVC → Pod

PV is created automatically by the provisioner — you never write it manually.

**Admin vs developer split:**

- Admin defines StorageClasses once
- Developer just picks one by name in their PVC

**Default StorageClass** — if your PVC omits `storageClassName`, it uses the cluster's default. In your GKE cluster that's `standard-rwo`.

### Define Storage Class
When using GKE, Google pre-installs StorageClasses for you. But you can define your own if you need different behavior — same provisioner, different parameters.

Common reason to define a custom one: changing `reclaimPolicy` from `Delete` to `Retain` for data that must survive PVC deletion.

In GKE, Google pre-installs the default StorageClasses for you — nobody on your team wrote them. But you _can_ define your own if the defaults don't fit your needs. For example if you want a StorageClass with `Retain` policy instead of `Delete`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-rwo-retain
provisioner: pd.csi.storage.gke.io
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

Same provisioner as `standard-rwo`, just different reclaim policy. Then developers use `standard-rwo-retain` in their PVC when data must survive PVC deletion.

And in GCP we can create PVC using this:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  storageClassName: standard-rwo-retain
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
```