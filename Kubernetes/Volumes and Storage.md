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
![[Pasted image 20260402180829.png]]

And to create `PersistentVolumeClaim`:
```yaml
apiVersion: v1
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
]![[Pasted image 20260402182936.png]]


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


