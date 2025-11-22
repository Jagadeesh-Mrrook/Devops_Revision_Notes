# Persistent Storage Overview in Kubernetes (Detailed Explanation Version)

## **Concept / What**

Persistent storage in Kubernetes refers to storage that continues to exist even when Pods are deleted, restarted, or moved to another node. Kubernetes uses a layered approach to storage: Volumes, Persistent Volumes (PV), Persistent Volume Claims (PVC), and StorageClasses.

## **Why / Purpose / Use Case in Real-World**

* Pods are ephemeral; their internal storage is lost on restart.
* Applications require durable storage for databases, logs, media uploads, caches, and shared files.
* Persistent storage ensures data continuity across pod restarts, rescheduling, or scaling.
* Supports stateful workloads such as MySQL, MongoDB, Jenkins, Prometheus, and shared workloads via EFS.

## **How it Works / Steps / Syntax**

1. Application requests storage using a **PVC**.
2. Kubernetes checks for a matching **PV**.
3. If no manual PV exists, a **StorageClass** dynamically provisions a new PV.
4. Storage backend (EBS/EFS) provides actual disk resources.
5. PVC binds to PV → Pod mounts PVC and stores data persistently.

**Flow:** Pod → PVC → PV → StorageClass → AWS Storage Backend

## **Common Issues / Errors**

* PVC stuck in `Pending` state.
* PV and PVC not binding due to mismatched size, StorageClass, AccessMode.
* EBS volume and Pod scheduled in different Availability Zones.
* EBS cannot be attached to multiple Pods (single-node limitation).
* EFS mount permission errors (`permission denied`, read-only filesystem).

## **Troubleshooting / Fixes**

* Check PVC status: `kubectl get pvc`.
* Describe PVC for detailed error: `kubectl describe pvc <name>`.
* Validate StorageClass linked to the PVC.
* Check Pod node AZ using `kubectl get pod -o wide`.
* Verify EBS/EFS configuration in AWS console.
* Ensure EFS security group allows NFS (2049) and correct access point permissions.

## **Best Practices / Tips**

* Use **EBS** for single-node workloads (databases).
* Use **EFS** for multi-node shared storage.
* Prefer **dynamic provisioning** using StorageClasses.
* Default to **gp3** EBS volumes for cost and performance.
* Avoid `hostPath` except for testing.
* Use EFS Access Points for secure directory-level isolation.
* Align workloads with AZ awareness to avoid EBS binding issues.
* Enable volume expansion for future storage growth.

---
---

# Volume Types in Kubernetes (emptyDir, hostPath, EBS, EFS)

## **Concept / What**

Kubernetes supports multiple volume types to handle different storage requirements. These include:

* **emptyDir** – temporary storage that lives only for the lifetime of a Pod.
* **hostPath** – mounts a directory or file from the node’s filesystem into the Pod.
* **EBS Volumes (with PV/PVC)** – AWS block storage attached to a single node in a single Availability Zone.
* **EFS Volumes (with PV/PVC)** – AWS shared file system accessible by multiple Pods across multiple Availability Zones.

These volume types enable Pods to store data temporarily, persistently, or share data across Pods and nodes.

## **Why / Purpose / Real-World Use Cases**

* **emptyDir**: used for caches, temporary working files, or communication between containers inside the same Pod.
* **hostPath**: used by monitoring agents, log collectors, CSI plugins that need access to node-level directories.
* **EBS (Persistent storage)**:

  * Databases like MySQL, MongoDB, PostgreSQL.
  * Jenkins controller home directory.
  * Any single-Pod, single-node stateful application.
* **EFS (Shared storage)**:

  * Shared assets for multiple replicas of an application.
  * Multi-AZ read-write-many use cases.
  * Jenkins agent shared workspace.
  * ML datasets/common application data shared between many Pods.

## **How It Works / Steps / Syntax**

### ### **1. emptyDir**

Created when the Pod starts; deleted permanently when the Pod is removed.

**Pod Manifest with emptyDir:**

```yaml\apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: cache-vol
          mountPath: /cache
  volumes:
    - name: cache-vol
      emptyDir: {}
```

### ### **2. hostPath**

Mounts a directory/file from the worker node into the Pod. Should be used carefully because Pods lose data when moved to another node.

**Pod Manifest with hostPath:**

```yaml\apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]
      volumeMounts:
        - name: host-log
          mountPath: /var/log/node
  volumes:
    - name: host-log
      hostPath:
        path: /var/log
        type: Directory
```

### ### **3. EBS Volumes (Using PV/PVC)**

EBS is a **zonal** block device. An EBS volume exists in a **single Availability Zone** and can attach **only to nodes in that same AZ**.

#### ⚠ Key Properties:

* EBS supports **ReadWriteOnce (RWO)**.
* One EBS volume can attach to **only one node at a time**.
* Multiple Pods using EBS → **multiple EBS volumes** required.
* If PV is in `ap-south-1a` and Pod runs on node in `ap-south-1b`, **volume cannot attach**.
* PVC stays **Pending** and Pod stuck in `ContainerCreating`.

### **Manual PV + PVC (No StorageClass)**

This is used in orgs with strict control over infrastructure.

**PV Manifest (Manual EBS volume pre-created):**

```yaml\apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-ebs-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  awsElasticBlockStore:
    volumeID: vol-0123456789abcdef0
    fsType: ext4
```

**PVC Manifest:**

```yaml\apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: manual-ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeName: manual-ebs-pv
```

### **Pod using EBS PVC:**

```yaml\apiVersion: v1
kind: Pod
metadata:
  name: ebs-app
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: manual-ebs-pvc
```

---

### ### **4. EFS Volumes (Using PV/PVC)**

EFS is a **regional** file system, accessible from **all Availability Zones**.
Supports **ReadWriteMany (RWX)** — multiple Pods in multiple AZs can mount the same EFS volume.

**StorageClass (Optional for dynamic provisioning):**

```yaml\apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-12345678
  directoryPerms: "700"
```

**PVC Manifest:**

```yaml\apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

**Deployment using EFS:**

```yaml\apiVersion: apps/v1
kind: Deployment
metadata:
  name: efs-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: efs-app
  template:
    metadata:
      labels:
        app: efs-app
    spec:
      containers:
        - name: app
          image: nginx
          volumeMounts:
            - name: shared
              mountPath: /shared
      volumes:
        - name: shared
          persistentVolumeClaim:
            claimName: efs-pvc
```

---

## **Common Issues / Errors**

### **EBS**

* Pod stuck in `ContainerCreating` due to AZ mismatch.
* PV/PVC not binding because of mismatched size or access mode.
* EBS volumes cannot attach to multiple Pods.

### **EFS**

* Permission denied on mount due to incorrect access point configuration.
* Missing security group rules for NFS port 2049.

### **hostPath**

* Path doesn’t exist on some nodes → container fails.
* Pod eviction to another node causes data loss.

### **emptyDir**

* Data disappears if the Pod restarts or is rescheduled.

---

## **Troubleshooting / Fixes**

* `kubectl describe pvc <name>` for root cause.
* Verify Pod AZ using `kubectl get pod -o wide`.
* For EBS issues:

  * ensure Pod and PV are in same AZ.
  * check underlying AWS EBS volume health.
* For EFS issues:

  * verify security group rules.
  * fix directory permissions.
* For hostPath:

  * ensure directory exists.
  * use `DirectoryOrCreate`.

---

## **Best Practices / Tips**

* Use **emptyDir** for temporary data only.
* Use **hostPath** only for trusted, node-specific workloads.
* Use **EBS** for single-node databases and stateful apps.
* Use **EFS** for shared storage, multi-AZ, multi-Pod access.
* Understand AZ restrictions: **EBS = AZ-bound**, **EFS = multi-AZ**.
* Use EFS Access Points for secure multi-tenant directory isolation.
* Keep PV/PVC specifications consistent to avoid binding issues.
* Use Retain policy for PVs that store critical data.

---
---

# Persistent Volume (PV) & Persistent Volume Claim (PVC) – Detailed Explanation Version

## **Concept / What**

**Persistent Volume (PV)** is a cluster-level storage resource representing actual storage (EBS, EFS, NFS, etc.). It exists independently of Pods and Nodes.

**Persistent Volume Claim (PVC)** is a request for storage by a Pod. PVC specifies size, access mode, and optionally StorageClass. PVC binds to a matching PV.

When no volumes are defined (no PV/PVC, no emptyDir, no hostPath), Kubernetes stores container data inside the container’s **ephemeral writable layer**, located on the **node’s local filesystem** (e.g., `/var/lib/containerd` or `/var/lib/docker`). This storage is temporary and deleted when the Pod terminates or moves.

---

## **Why / Purpose / Use Case in Real-World**

* Ensure application data persists beyond Pod restarts.
* Keep data safe even if a node fails, restarts, or is replaced.
* Required for stateful applications like MySQL, Jenkins, MongoDB, Prometheus, and applications needing durable file storage.
* Provide controlled and reliable storage independent of node lifecycle.

**Key need:** Pod filesystem is ephemeral → PV/PVC provides durable storage.

---

## **How It Works / Steps / Syntax**

### **1. Manual PV + PVC (No StorageClass)**

Used in strict production environments where storage is managed manually.

### **Persistent Volume (PV)**

A storage admin manually creates an EBS volume and defines a PV pointing to it.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-ebs-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  awsElasticBlockStore:
    volumeID: vol-0abcd1234efgh5678
    fsType: ext4
```

### **Persistent Volume Claim (PVC)**

The application requests storage matching the PV.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: manual-ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeName: manual-ebs-pv
```

### **Pod using PVC**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ebs-backed-app
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: manual-ebs-pvc
```

---

## **Where Data Goes If No PV/PVC or Volumes Are Used**

When no PV/PVC/emptyDir/hostPath is defined:

* Kubernetes stores container data in the **container’s writable layer**.
* This writable layer physically lives on the **node’s local disk**.
* Path (depends on runtime):

  * `/var/lib/containerd/` (containerd)
  * `/var/lib/docker/` (Docker)
  * `/var/lib/kubelet/pods/<pod-id>/volumes/` for internal ephemeral volumes.

This storage is **ephemeral**, removed when the Pod restarts or moves to another node.

---

## **Key AWS EBS Behavior (Critical in Production)**

* EBS volume belongs to **a single Availability Zone**.
* PV backed by EBS can only attach to nodes **in the same AZ**.
* If PVC/PV is in ap-south-1a, Pods must run on nodes in ap-south-1a.
* Pods scheduled in another AZ cannot attach the volume and remain stuck.
* EBS supports **ReadWriteOnce (RWO)** → cannot be shared by multiple Pods.

---

## **Common Issues / Errors**

* **PVC stuck in Pending** → no matching PV specifications.
* **Pod stuck in ContainerCreating** → AZ mismatch between PV’s EBS and Pod’s node.
* **Wrong volumeMode or accessModes** → PVC will not bind.
* **Using PV on multiple Pods** → fails because EBS only supports RWO.
* **Node failure without PV** → ephemeral container data lost.

---

## **Troubleshooting / Fixes**

* `kubectl describe pvc <name>` → binding issues.
* `kubectl describe pv <name>` → verify access modes, capacity, reclaim policy.
* `kubectl get pod -o wide` → check node AZ.
* `aws ec2 describe-volumes` → verify EBS volume AZ and state.
* Ensure Pod and PV are in the same AZ for EBS.
* Use correct filesystem type (`fsType: ext4`).

---

## **Best Practices / Tips**

* Use manual PV/PVC model for strict environments (banks, regulated companies).
* Use **EBS-backed PVs** for single-instance stateful apps.
* Always ensure Pod’s node AZ matches the EBS volume’s AZ.
* Use `Retain` reclaim policy for critical production data.
* Use separate EBS volumes for each Pod (EBS = RWO).
* Understand that node storage is ephemeral for Pods — never store app data there.
* Use EFS for multi-AZ or ReadWriteMany workloads.

---
---

# StorageClass & Dynamic Provisioning – Detailed Explanation Version

## **Concept / What**

A **StorageClass** defines *how* persistent storage should be provisioned dynamically in Kubernetes. It tells Kubernetes which storage backend to use (EBS/EFS), its configuration (type, parameters), and how volumes should be created and bound.

**Dynamic Provisioning** means:

* You create a **PVC**.
* Kubernetes automatically creates **a PV** and the **underlying storage** (EBS/EFS) using the StorageClass.
* No manual EBS or PV creation needed.

---

## **Why / Purpose / Use Case in Real-World**

* Remove manual PV creation.
* Automatically create EBS/EFS with correct size and parameters.
* Prevent AZ mismatch using `WaitForFirstConsumer`.
* Standardize storage (gp3, io2, EFS) across teams.
* Support autoscaling and dynamic workloads.
* Allow on-the-fly volume expansion.
* Clean lifecycle management using reclaim policies.

**Real-world usage:**

* Most production clusters use StorageClass for all persistent storage.
* E-commerce and microservices use StorageClass for temporary shared storage, caching directories, file processing folders, etc.

---

## **How It Works / Steps / Syntax**

### **1. Create StorageClass (EBS example)**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
```

### Key Fields Explained

* **provisioner** – CSI driver that creates volumes (EBS/EFS).
* **parameters.type** – EBS type (gp3, io2, etc.).
* **volumeBindingMode: WaitForFirstConsumer**

  * Ensures EBS volume is created **in the same AZ** as the Pod.
  * Prevents AZ mismatch issues.
* **allowVolumeExpansion** – Allows PVC resizing.
* **reclaimPolicy** – Automatically delete PV/EBS when PVC is deleted.

---

### **2. PVC Using StorageClass**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 20Gi
```

**What happens:**

* Kubernetes looks at StorageClass `gp3`.
* CSI driver creates a **20Gi EBS** volume.
* A PV is auto-created and bound.

---

### **3. Pod Using Dynamically Created PVC**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ebs-app
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: ebs-claim
```

The Pod automatically mounts the newly created EBS volume.

---

## **Real-Time Production Scenarios**

### **1. Preventing AZ Mismatch**

`WaitForFirstConsumer` ensures:

* Pod scheduled first → AZ chosen → EBS created in the same AZ.
* Eliminates Multi-Attach and mount failures.

### **2. Horizontal Scaling**

Each new PVC gets its own EBS volume automatically.
Useful for background workers or one-per-pod storage.

### **3. Shared Storage With Deployment**

When using **EFS StorageClass**, all replicas can mount a single PV.
Essential for:

* Shared processing folders
* E-commerce temporary image processing
* Shared cache or shared output data

### **4. Fast Deployments**

Developers request storage via PVC only. No need to talk to DevOps or AWS.

---

## **Common Issues / Errors**

* **PVC stuck in Pending**

  * Wrong StorageClass name
  * StorageClass deleted or disabled
  * EBS quota exceeded

* **Pod stuck in ContainerCreating**

  * EBS cannot mount (AZ mismatch if not using WaitForFirstConsumer)
  * CSI driver missing or unhealthy

* **Volume expansion not applied**

  * `allowVolumeExpansion` disabled
  * Underlying filesystem not resized

---

## **Troubleshooting / Fixes**

Commands:

```
kubectl get sc
kubectl describe sc <name>
kubectl describe pvc <pvc>
kubectl describe pod <name>
kubectl get pv
```

AWS-side:

```
aws ec2 describe-volumes
```

Checks:

* Validate StorageClass exists.
* Check if EBS volume is in the correct AZ.
* Ensure CSI drivers (EBS/EFS) are installed.
* PVC and PV accessModes must match.

---

## **Best Practices / Tips**

* Prefer **StorageClass** over manual PV creation.
* Always use `WaitForFirstConsumer` to avoid AZ mismatch.
* Use **gp3** as the standard EBS type.
* Use **reclaimPolicy: Delete** for non-critical workloads.
* Use **reclaimPolicy: Retain** when data must be preserved.
* Use **allowVolumeExpansion: true** to support resizing.
* For Deployments → use EFS StorageClass (shared storage).
* For StatefulSets → use EBS StorageClass (1 PVC per replica).
* Tag dynamically created EBS volumes for cost tracking.

---
---

# Volume Expansion & Reclaim Policies – Detailed Explanation Version

## **Concept / What**

**Volume Expansion** allows increasing the size of an existing PersistentVolumeClaim (PVC) without recreating the PV or losing data.

**Reclaim Policies** define what happens to a PersistentVolume (and underlying storage like EBS/EFS) when its PVC is deleted.

Kubernetes supports three reclaim policies:

* **Retain** – keep data and PV
* **Delete** – delete PV and backend storage
* **Recycle** – deprecated

---

## **Why / Purpose / Use Case in Real-World**

### **Volume Expansion** is required when:

* Application needs more storage
* Image processing or temporary files increase
* Batch jobs outgrow existing disk
* e-commerce file processing jobs need more space
* Disk usage warning or threshold reached

### **Reclaim Policy** is required to:

* Preserve critical production data (Retain)
* Automatically clean temp/dev workloads (Delete)
* Control storage lifecycle

---

## **How It Works / Steps / Syntax**

### **1. Volume Expansion (EBS only)**

Prerequisites:

* StorageClass must have:

```yaml
allowVolumeExpansion: true
```

* Underlying storage must support expansion (EBS supports it)

### **StorageClass Example:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### **Update PVC Size:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  resources:
    requests:
      storage: 40Gi
```

Apply:

```
kubectl apply -f pvc.yaml
```

Kubernetes → Resizes EBS → Resizes filesystem.

---

## **2. Expansion Behavior for EFS**

EFS is a serverless filesystem that **auto-expands** without any PVC changes.

* No need for `allowVolumeExpansion`
* No need to edit PVC
* No manual resizing
* Storage grows on-demand

PVC size for EFS is symbolic and not enforced.

---

## **Reclaim Policies (Detailed)**

### **1. Retain**

```yaml
persistentVolumeReclaimPolicy: Retain
```

**Behavior:**

* PVC deleted → PV stays
* Backend volume (EBS/EFS) stays
* Data preserved
* Requires manual cleanup

Used for:

* Critical production data
* E-commerce assets
* Important generated files

---

### **2. Delete**

```yaml
persistentVolumeReclaimPolicy: Delete
```

**Behavior:**

* PVC deleted → PV deleted
* Backend storage deleted (EBS/EFS)
* Data removed

Used for:

* Temporary workloads
* Non-critical environments
* Cache or processing directories

---

### **3. Recycle**

Deprecated. Not used in modern Kubernetes.

---

## **Real-Time Production Scenarios**

### **1. EBS Volume Running Out of Space**

PVC updated → Kubernetes auto-expands PV/EBS → application continues running.

### **2. EFS Auto-Scaling for Shared Storage**

E-commerce image processing folders grow → EFS automatically scales → no manual work.

### **3. Protecting Data from Accidental PVC Deletion**

Use `Retain` for files that should never be lost.

### **4. Temporary Data Workloads**

Use `Delete` for dev/test jobs where cleanup should be automatic.

---

## **Common Issues / Errors**

* StorageClass missing `allowVolumeExpansion`
* CSI driver missing or outdated
* PVC stuck in "Resizing" state
* Filesystem not expanded (requires pod restart)
* PV with `Retain` stuck in "Released" state after PVC deletion

---

## **Troubleshooting / Fixes**

Commands:

```
kubectl describe pvc <name>
kubectl get pv
kubectl describe pv <name>
kubectl get sc
```

AWS-side:

```
aws ec2 describe-volumes --volume-id <id>
```

Fixes:

* Enable expansion in StorageClass
* Restart pod to trigger filesystem resize
* Delete PV manually when using `Retain`

---

## **Best Practices / Tips**

* Use `allowVolumeExpansion: true` for all EBS StorageClasses
* For critical workloads → use `Retain`
* For non-critical or temporary workloads → use `Delete`
* Always use `WaitForFirstConsumer` to avoid AZ mismatch (EBS)
* Monitor disk usage and expand proactively
* Prefer EFS for shared storage with Deployments
* Prefer EBS for StatefulSets with per-pod volumes

---
---

# AWS EBS CSI Driver & AWS EFS CSI Driver – Detailed Explanation Version

## **Concept / What**

The **AWS EBS CSI Driver** and **AWS EFS CSI Driver** allow Kubernetes to provision and manage AWS storage resources natively.

### **EBS CSI Driver**

* Manages **Amazon EBS volumes**.
* Supports **dynamic provisioning**, **attaching**, **detaching**, and **expanding** EBS volumes.
* Provides persistent **block storage** (RWO – ReadWriteOnce).
* Best for **single pod**, **single node**, **stateful workloads**.

### **EFS CSI Driver**

* Manages **Amazon EFS filesystems**.
* Supports **shared storage**, **multi-AZ**, and **dynamic Access Points**.
* Provides persistent **network file storage** (RWX – ReadWriteMany).
* Best for **multiple pods**, **multiple nodes**, **shared data**.

---

## **Why / Purpose / Use Case in Real-World**

### **Why EBS CSI?**

* Needed for workloads that require **local isolated storage**.
* Perfect for **StatefulSet** where each replica gets its own disk.
* Supports volume expansion.
* High performance and cheaper than EFS.

### **Why EFS CSI?**

* Best for workloads that need **shared storage across replicas**.
* Perfect for **Deployments** with multiple pods.
* Required for **e-commerce image processing**, shared cache folders, shared temporary data.
* Multi-AZ and auto-scaling storage.

---

## **How It Works / Steps / Internals**

### **EBS CSI Driver Flow**

1. PVC created → references EBS StorageClass.
2. StorageClass triggers the EBS CSI driver.
3. CSI driver calls AWS API → creates an EBS volume.
4. Volume attached to the node where the pod is scheduled.
5. Volume mounted inside the container.
6. When pod deleted → volume detached.

### **Limitations:**

* **Zonal** only.
* **Single pod** access (RWO).
* Cannot be shared across multiple pods.

---

### **EFS CSI Driver Flow**

1. PVC created → references EFS StorageClass.
2. StorageClass triggers EFS CSI driver.
3. Driver creates an **EFS Access Point** (dynamic provisioning).
4. Mounts EFS into the pod using NFS protocol.
5. Multiple pods across multiple nodes/AZs can access same filesystem.

### **Capabilities:**

* Multi-AZ.
* RWX (multiple pods).
* Auto-expanding filesystem.

---

## **Manifest Examples**

### **1️⃣ EBS StorageClass (gp3)**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### **2️⃣ EFS StorageClass**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-12345678
  directoryPerms: "700"
```

### **3️⃣ EFS PVC (shared)**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

> Note: EFS auto-expands even if PVC says 5Gi.

---

## **Real-Time Production Scenarios**

### **EBS CSI (used when):**

* StatefulSet replicas each need their own disk.
* Single pod stateful workloads.
* High-performance block storage.
* Temporary isolated data for processing.

### **EFS CSI (used when):**

* Deployment with multiple replicas needing shared data.
* E-commerce image processing (shared uploads directory).
* Shared cache/storage directories.
* CI/CD build artifacts shared across pods.
* Cross-AZ access required.

---

## **Common Issues / Errors**

### **EBS CSI Issues:**

* Pod stuck in `ContainerCreating` → AZ mismatch.
* `Multi-Attach` error → trying to attach same EBS to multiple pods.
* PVC expansion not applied → filesystem not resized.
* CSI driver not installed.

### **EFS CSI Issues:**

* `Permission denied` → Access Point permissions incorrect.
* NFS (2049) blocked in security groups.
* Wrong filesystem ID or mount target missing.
* Cross-VPC or cross-account access not configured.

---

## **Troubleshooting / Fixes**

Commands:

```
kubectl describe pvc <name>
kubectl describe pv <name>
kubectl describe pod <name>
kubectl get sc
kubectl get csinodes
```

AWS checks:

```
aws ec2 describe-volumes
aws efs describe-file-systems
```

Fixes:

* Use `WaitForFirstConsumer` for EBS.
* Ensure correct UID/GID for EFS Access Points.
* Open NFS port 2049 between nodes and EFS.
* Ensure IAM roles for CSI drivers are configured.

---

## **Best Practices / Tips**

### **EBS CSI Driver (RWO):**

* Use for StatefulSet pods.
* Avoid using EBS with Deployments.
* Enable `allowVolumeExpansion` for resizing.
* Always use `gp3` for cost/performance.

### **EFS CSI Driver (RWX):**

* Use for Deployments and multi-pod access.
* Use Access Points for per-application isolation.
* Ensure NFS ingress allowed in security groups.
* Use for multi-AZ shared storage.

### **General:**

* Tag all automatically created EBS/EFS resources.
* Use Retain for critical data (EBS).
* Use Delete for temp workloads.
* Monitor disk usage to avoid outages.

---
---
---
