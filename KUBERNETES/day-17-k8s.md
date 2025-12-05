# **ETCD Backups in Managed EKS — Detailed Explanation Notes**

## **1. Concept / What**

ETCD is the distributed key-value store Kubernetes uses to store the entire cluster state (pods, deployments, services, secrets, RBAC, etc.). In **Amazon EKS**, ETCD and the entire control plane are **fully managed by AWS**, meaning customers cannot access, back up, or restore ETCD directly. AWS performs all ETCD backups internally and handles all restoration in case of control-plane failures.

---

## **2. Why / Purpose / Use Case in Real-World**

Although customers don’t control ETCD in EKS, AWS maintains ETCD backups for the following internal purposes:

* Ensuring **high availability** of the control plane.
* Recovering from **control-plane corruption**.
* Handling **multi-AZ failures** impacting ETCD replicas.
* Supporting **AWS-managed upgrades** and maintenance.

From a customer standpoint:

* ETCD backups do **not** help with workload recovery.
* You must still back up **application objects** and **persistent data** separately.

---

## **3. How It Works / Steps / Syntax**

### **How AWS Manages ETCD Backups**

* AWS stores ETCD data on **multi-AZ replicated storage**.
* AWS takes **automatic ETCD backups** for internal recovery.
* AWS restores the control plane *only when AWS causes or detects a failure*.

### **What Customers Can Do**

Customers **cannot**:

* Take ETCD snapshots.
* Restore ETCD snapshots.
* Roll back cluster ETCD state.
* Migrate ETCD to another region.

### **Cluster Recovery From Customer Perspective**

If customers break the cluster state, they must:

1. Create a **new EKS cluster**.
2. Re-deploy resources using IaC tools (Terraform, GitOps), Helm, or YAML manifests.
3. Restore Persistent Volume data using **EBS snapshots** or **Velero backups**.
4. Re-create or attach node groups as needed.

### **Useful Commands (Workload Backups Only)**

Export all cluster objects:

```bash
kubectl get all --all-namespaces -o yaml > backup-cluster.yaml
```

Export CRDs:

```bash
kubectl get crds -o yaml > crds-backup.yaml
```

Restore them:

```bash
kubectl apply -f backup-cluster.yaml
```

---

## **4. Common Issues / Errors**

* **Accidentally deleted deployments/namespaces** → ETCD backup cannot help.
* **PV data loss** → ETCD stores metadata only, not EBS volume contents.
* **Add-ons break after recreating cluster** → Add-ons must be reinstalled.
* **Cluster misconfiguration** → Cannot revert using ETCD backup.
* **Customer confusion** thinking ETCD backup is for workload recovery (it is not).

---

## **5. Troubleshooting / Fixes**

### Issue: Application or namespace deleted

**Fix:** Reapply manifests, restore data from Velero/EBS.

### Issue: Persistent volume data missing

**Fix:** Use **EBS snapshots** or Velero Restic backups.

### Issue: Cluster is unhealthy due to AWS fault

**Fix:** AWS automatically restores control plane using internal ETCD backup.

### Issue: Need cross-region migration

**Fix:** Create new cluster → deploy workloads → restore data (ETCD backup cannot be used).

---

## **6. Best Practices / Tips**

* Assume **ETCD backups in EKS are NOT part of your DR strategy**.
* Always store manifests in **GitOps / Terraform** for easy redeployment.
* Use **Velero** for backing up Kubernetes objects + PV data.
* Use **EBS snapshots** for storage-level backups.
* Maintain **multi-AZ node groups** for availability.
* Never depend on AWS ETCD backup for restoring your workloads.
* Treat EKS clusters as **rebuildable**, and use IaC for reproducibility.

---
---

# **Velero – Backups and Restores (Detailed Explanation Notes)**

## **1. Concept / What**

**Velero** is an open-source backup and restore tool for Kubernetes clusters. It backs up Kubernetes API objects (Deployments, Services, Secrets, ConfigMaps, CRDs, etc.) and persistent data (Persistent Volumes) using cloud-native snapshots or file-level Restic backups. It enables cluster recovery, migration across clusters/regions, scheduled backups, and disaster recovery.

Velero consists of:

* **Velero CLI** (installed locally) – used to run backup/restore commands.
* **Velero server components** (Deployment + optional Restic DaemonSet) – installed inside the Kubernetes cluster.
* **Backup storage location** (e.g., S3 bucket) – where cluster backup data is stored.

---

## **2. Why / Purpose / Use Case in Real-World**

Organizations use Velero when they need:

* Backup of **Kubernetes objects** (in case of accidental deletion).
* Backup of **Persistent Volumes** (stateful applications).
* **Disaster recovery** for cluster crashes.
* **Cluster migration** across clusters, regions, or cloud providers.
* **Compliance** requiring periodic backups.

Velero is especially useful for clusters running:

* Databases or stateful apps inside Kubernetes.
* Operators and CRDs with complex runtime state.
* Workloads needing full restore, not just re-deploy.

---

## **3. How It Works / Steps / Syntax**

### **Backup Storage Setup (S3 for AWS)**

Velero stores backup data in an S3 bucket:

```bash
aws s3api create-bucket --bucket velero-backup-bucket --region ap-south-1
```

This bucket holds all YAML backups, metadata, logs, and snapshot info.

### **Install Velero CLI (locally)**

```bash
curl -L https://github.com/vmware-tanzu/velero/releases/download/v1.13.0/velero-v1.13.0-linux-amd64.tar.gz
```

The CLI runs backup and restore commands.

### **Install Velero in the Kubernetes Cluster**

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.6.0 \
  --bucket velero-backup-bucket \
  --backup-location-config region=ap-south-1 \
  --snapshot-location-config region=ap-south-1
```

This installs:

* `velero` namespace
* Velero deployment
* Backup/Restore CRDs
* Restic DaemonSet (optional for PV file backups)

### **Backup Examples**

Full cluster backup:

```bash
velero backup create full-backup
```

Backup a namespace:

```bash
velero backup create dev-backup --include-namespaces=dev
```

Backup specific resource types:

```bash
velero backup create deploy-backup --include-resources=deployments
```

### **Restore Examples**

Restore entire backup:

```bash
velero restore create --from-backup full-backup
```

Restore namespace:

```bash
velero restore create dev-restore --from-backup dev-backup
```

### **Schedule Backups**

```bash
velero schedule create daily-backup --schedule="0 3 * * *"
```

### **Backup CR Example (Manifest)**

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: dev-backup
  namespace: velero
spec:
  includedNamespaces:
    - dev
  snapshotVolumes: true
  ttl: 720h
```

### **Restore CR Example (Manifest)**

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: dev-restore
  namespace: velero
spec:
  backupName: dev-backup
```

---

## **4. Common Issues / Errors**

* Backups stuck due to **insufficient IAM permissions** for S3/EBS.
* PVs not restoring because **Restic or snapshots not enabled**.
* CRDs missing because they were not backed up before backing up resources.
* Restore failures due to **resource conflicts** or webhook mutations.
* Version mismatches between Velero and the cloud provider plugin.

---

## **5. Troubleshooting / Fixes**

* Fix IAM issues by attaching correct S3 + EBS snapshot permissions.
* Enable **Restic** or ensure EBS volume snapshot support for PV backups.
* Back up CRDs separately:

  ```bash
  kubectl get crds -o yaml > crds-backup.yaml
  ```
* Resolve restore conflicts using:

  ```bash
  velero restore create --from-backup full-backup --restore-volumes=false
  ```
* Upgrade Velero and AWS plugin together to avoid compatibility issues.

---

## **6. Best Practices / Tips**

* Use **EBS snapshots** for persistent data and Restic for non-snapshot storage.
* Configure **scheduled backups** for production clusters.
* Replicate S3 bucket across regions for DR.
* Always back up **CRDs** before CR instances.
* Monitor Velero logs:

  ```bash
  kubectl logs -n velero deploy/velero
  ```
* Test restore procedures regularly.
* In stateless, GitOps-based environments with externalized state (RDS, Secrets Manager), Velero may not be required.

---
---

# **EBS Snapshot–Based Backups (Detailed Explanation Notes)**

## **1. Concept / What**

Amazon **EBS Snapshots** are block-level backups of EBS volumes. In Kubernetes (specifically EKS), Persistent Volumes (PVs) backed by EBS volumes can be backed up and restored using snapshots. These backups capture the entire state of the volume and allow restoring data by creating new volumes from the snapshots.

Snapshots are:

* Incremental and cost-efficient.
* Stored in Amazon S3 internally (managed by AWS).
* The recommended DR method for stateful workloads running on EBS-backed PVs.

Kubernetes integrates with AWS snapshots using the **EBS CSI Driver** and **VolumeSnapshot APIs**.

---

## **2. Why / Purpose / Use Case in Real-World**

EBS snapshots are used when Kubernetes stores persistent data inside EBS volumes. Common scenarios:

* Backing up stateful workloads such as MySQL, PostgreSQL, ElasticSearch, Kafka, Redis, Jenkins, Prometheus, etc.
* Recovering from corruption or accidental deletion of PVs.
* Migrating workloads across clusters or AWS regions.
* Creating rollback points during risky deployments.
* Meeting compliance requirements for daily/weekly/monthly backups.

Snapshots allow:

* Fast restoration.
* Point-in-time recovery.
* Replication to different regions.

---

## **3. How It Works / Steps / Syntax**

### **EKS PV to EBS Binding**

In EKS:

```
PVC → PV → EBS Volume
```

The EBS volume stores the actual persistent data.

### **Snapshot Creation via Kubernetes (CSI Snapshot Controller)**

Create a `VolumeSnapshot` to back up a PV:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-db-snapshot
spec:
  volumeSnapshotClassName: csi-aws-ebs-snapclass
  source:
    persistentVolumeClaimName: my-db-pvc
```

This triggers AWS to create an EBS snapshot.

### **Snapshot Restore via Kubernetes**

Create a PVC from the snapshot:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-db-pvc
spec:
  storageClassName: gp3
  dataSource:
    name: my-db-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

A new EBS volume is created with the snapshot's data.

### **Snapshot Management via AWS CLI**

Create snapshot:

```bash
aws ec2 create-snapshot --volume-id vol-12345 --description "backup"
```

Restore snapshot:

```bash
aws ec2 create-volume --snapshot-id snap-12345 --availability-zone ap-south-1a
```

---

## **4. Common Issues / Errors**

* **Snapshot creation fails** → EBS CSI driver missing or misconfigured.
* **Restore fails** → Wrong VolumeSnapshotClass or insufficient IAM permissions.
* **Pod stuck in Pending** → Restored PVC does not match Pod's requirements.
* **Snapshot across regions** → Snapshots must be manually copied across regions.
* **Volume size mismatch** → Restored PVC size must be ≥ original volume size.

---

## **5. Troubleshooting / Fixes**

* Check EBS CSI driver logs:

  ```bash
  kubectl logs -n kube-system deploy/ebs-csi-controller
  ```
* Ensure IAM permissions for:

  * `ec2:CreateSnapshot`
  * `ec2:DescribeSnapshots`
  * `ec2:CreateVolume`
  * `ec2:DeleteSnapshot`
* For cross-region restore, copy the snapshot manually.
* Update PVC definitions to match required access modes and storage size.
* Delete orphaned PVC/PV objects before restore to avoid conflicts.

---

## **6. Best Practices / Tips**

* Always enable **EBS CSI driver** for snapshot support.
* Use **AWS Backup** to automate scheduled EBS snapshots.
* Tag snapshots for cost management and retention enforcement.
* Regularly test the **restore process**.
* Use `Retain` policy in `VolumeSnapshotClass` for safer recovery.
* Avoid placing core business databases inside Kubernetes; prefer RDS/Aurora.
* For high-performance workloads, use gp3 volumes with proper baseline settings.

---
---

# **Cluster Migration & Recovery (Detailed Explanation Notes)**

## **1. Concept / What**

**Cluster Migration & Recovery** refers to the process of moving workloads from one Kubernetes cluster to another or rebuilding a cluster after failure. This includes:

* Migrating clusters across regions, cloud providers, or VPCs.
* Recreating clusters after corruption, misconfiguration, or disaster.
* Restoring workloads, operators, PV data, secrets, and networking components.

Migration & recovery ensure business continuity and allow controlled transitions between environments.

---

## **2. Why / Purpose / Use Case in Real-World**

Organizations perform cluster migration/recovery for:

* **Version upgrades** (major Kubernetes upgrades).
* **Region migrations** for latency, cost, or DR.
* **Cluster failures** caused by misconfigurations or human error.
* **Disaster events** (AZ outages, corruption, security incidents).
* **Architecture restructuring** (new VPC, new node groups, new add-ons).
* **Testing DR readiness** for compliance.

These processes ensure that workloads remain resilient and portable.

---

## **3. How It Works / Steps / Syntax**

There are two major migration/recovery models depending on how the cluster is designed.

---

### **A. Stateless / Externalized State Migration (Modern Preferred Model)**

Used when:

* Applications inside Kubernetes are stateless.
* Databases live in RDS/DynamoDB/other managed services.
* Secrets live in AWS Secrets Manager.
* Frontend lives in S3/CloudFront.

**Steps:**

1. Recreate the cluster using Terraform.
2. Deploy core add-ons (CNI, CoreDNS, metrics-server, etc.).
3. Install operators (Prometheus Operator, External Secrets, etc.).
4. Redeploy workloads using GitOps or kubectl.
5. Validate health.
6. Update DNS or switch traffic.
7. Decommission old cluster.

No backup/restore inside Kubernetes is required.

---

### **B. Stateful Cluster Migration (PV + In-cluster State)**

Used when applications store data inside PVs (EBS).

**Steps:**

1. Back up Kubernetes objects (Velero or GitOps).
2. Back up PVs (EBS snapshots or Restic).
3. Create a new cluster.
4. Restore workloads.
5. Restore PVs and bind PVCs.
6. Validate app connectivity.
7. Switch traffic.

This is more complex than stateless migration.

---

### **Common Tools Involved**

* **Terraform** → recreate infrastructure & clusters.
* **GitOps (ArgoCD/Flux)** → redeploy manifests.
* **Velero** → backup/restore objects & PVs (optional).
* **EBS snapshots / AWS Backup** → handle PV backup.
* **External Secrets Operator** → sync secrets from AWS Secrets Manager.

---

## **4. Common Issues / Errors**

* **CRDs missing** → CRDs restored after CRs cause failures.
* **PV not attaching** → AZ mismatch or wrong StorageClass.
* **Ingress not coming up** → missing ALB/NGINX controller.
* **Secrets missing** → External Secrets not synced yet.
* **Operator state lost** → operator-managed objects not recreated.
* **ClusterIP changes** → service IPs differ across clusters.
* **IAM roles missing** → IRSA roles not recreated.

---

## **5. Troubleshooting / Fixes**

* Restore **CRDs first**:

  ```bash
  kubectl apply -f crds.yaml
  ```
* Ensure PV restores match **AZ + StorageClass**.
* Reinstall Ingress controller before restoring Ingress objects.
* Re-run External Secrets Operator to sync secrets.
* Verify IAM policies for IRSA.
* Delete stale PVCs/PVs if they conflict with restored ones.
* Validate health using readiness/liveness probes.

---

## **6. Best Practices / Tips**

* Treat clusters as **disposable** and recreate them using IaC.
* Keep state **outside Kubernetes** using RDS/S3/Secrets Manager.
* Use **EBS snapshots** for internal PVs (Prometheus, Grafana, caches).
* Use Velero only when Kubernetes stores critical state.
* Automate DR testing quarterly.
* Use multi-AZ node groups for stability.
* Keep all manifests version-controlled (GitOps).
* Document restore steps thoroughly.

---
---

# **Disaster Recovery Strategies (Detailed Explanation Notes)**

## **1. Concept / What**

**Disaster Recovery (DR)** is the set of strategies and processes used to restore applications, infrastructure, and data after failures such as region outages, cluster corruption, accidental deletions, or security incidents.

DR covers:

* Backups (objects + data)
* Failover mechanisms
* Cluster recreation
* Application redeployment
* Data restoration
* Traffic redirection

DR is not just taking backups—it includes the entire recovery workflow.

---

## **2. Why / Purpose / Use Case in Real-World**

Organizations need DR to ensure:

* **Business continuity** during failures.
* **Protection from accidental deletions** (namespaces, PVs, objects).
* **Recovery from infrastructure failures** (AZ outage, node failures).
* **Protection of application data** (database corruption, PV loss).
* **Compliance** (PCI, SOC2, HIPAA require DR planning).

Common DR events:

* Region failure
* Cluster corruption
* Human errors
* Misconfigurations
* Security breaches

---

## **3. How It Works / DR Strategy Components**

A complete Kubernetes DR strategy spans multiple layers:

### **A. Application Layer DR**

* Applications redeployed from Git or CI/CD.
* Immutable container builds.
* Multi-AZ deployments.
* Autoscaling.

### **B. Data Layer DR**

AWS-managed backups depend on where state lives:

* RDS automated backups + snapshots.
* DynamoDB PITR.
* EBS snapshots for PVs.
* S3 versioning and replication.
* ElastiCache backups.

### **C. Cluster Layer DR**

* Recreate EKS using Terraform/IaC.
* Reinstall add-ons (CNI, kube-proxy, CoreDNS).
* Reinstall operators (Prometheus Operator, External Secrets, etc.).

### **D. Networking Layer DR**

* ALB recreation.
* Route 53 failover.
* CloudFront distribution updates.
* VPC routing validation.

### **E. Failover Strategy**

Different failover models:

* **Backup & Restore** (most common for stateless).
* **Warm Standby**.
* **Active/Passive Multi-Region**.
* **Active/Active Multi-Region**.
* **Pilot Light**.

---

## **4. Types of Kubernetes DR Strategies**

### **1. Backup & Restore (Most common for stateless clusters)**

* Terraform recreates cluster.
* CI/CD or GitOps redeploys workloads.
* Data restored via AWS (RDS, EBS snapshots).
* Easy, low-cost, suitable for stateless workloads.

### **2. Warm Standby**

* Secondary cluster runs minimal capacity.
* Scaled up during disaster.
* Faster recovery.

### **3. Active/Passive Multi-Region**

* One region active, one region ready for failover.
* Data replicated cross-region.
* Fast failover.

### **4. Active/Active Multi-Region**

* Both regions active.
* Real-time traffic split.
* Most expensive but zero-downtime.

---

## **5. Real-World DR Scenarios**

### **Scenario 1 — Region Failure**

* Route 53 switches traffic to DR region.
* RDS read replica promoted.
* EKS recreated via Terraform.
* Workloads redeployed via GitOps/CI-CD.
* PVs restored from snapshots.

### **Scenario 2 — Namespace Deletion**

* Redeploy from GitOps/CI-CD.
* Restore secrets via ESO.
* Reconnect to RDS.

### **Scenario 3 — PV Corruption**

* Restore PV from EBS snapshot.
* Rebind PVC to restored PV.

### **Scenario 4 — Cluster Corruption**

* Destroy cluster.
* Recreate via Terraform.
* Reinstall add-ons and operators.
* Redeploy workloads.
* Restore PV snapshots if needed.
* Update Route 53.

---

## **6. CI/CD vs GitOps in DR**

**CI/CD (Jenkins/GitLab)** and **GitOps (ArgoCD/Flux)** do not change DR strategy.

Key point:

> DR depends on **where state lives**, not on the deployment method.

As long as workloads and manifests exist in Git:

* CI/CD can redeploy after DR.
* GitOps can resync automatically.

No need for Velero unless the cluster stores internal state.

---

## **7. Common Issues During DR**

* Missing CRDs → CRs fail to restore.
* PVs fail to attach → AZ mismatch.
* Secrets missing → External Secrets not yet synced.
* Ingress fails → controller not installed.
* IRSA roles missing → pods fail to access AWS.
* Apps crash-looping → ConfigMaps/Secrets missing.

---

## **8. Troubleshooting DR Failures**

* Restore **CRDs first** before restoring CR instances.
* Verify PV snapshot AZ and StorageClass.
* Reinstall Ingress controller before restoring Ingress.
* Run ESO sync to restore secrets.
* Recreate IRSA roles and IAM policies.
* Test workload health with readiness/liveness probes.

---

## **9. Best Practices / Tips**

* Keep clusters stateless; store data in RDS/S3.
* Use AWS-native backup tools (RDS backup, EBS snapshots, S3 replication).
* Use GitOps/CI-CD for workload redeployment.
* Recreate clusters using Terraform.
* Automate DR tests every quarter.
* Distribute workloads across multiple AZs.
* Document all DR procedures.
* Only use Velero when Kubernetes stores important internal state.

---
---




