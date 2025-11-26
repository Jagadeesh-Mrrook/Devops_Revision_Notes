# Kubernetes Scheduler (Filters & Scoring) — Detailed Explanation

## **Concept / What**

Kubernetes Scheduler decides which node a Pod should run on. It uses two steps: filtering (remove unsuitable nodes) and scoring (rank remaining nodes and choose the best one).

## **Why / Purpose / Use Case in Real-World**

* Ensures pods land on nodes with required resources.
* Schedules workloads based on node labels and constraints.
* Improves availability by spreading pods.
* Ensures correct placement: GPU nodes, high-memory nodes, prod-only nodes.
* Helps avoid resource hotspots.

## **How it Works / Steps / Syntax**

### **1. Filtering**

Scheduler removes nodes that don’t meet pod requirements:

* Insufficient CPU or memory.
* Node not Ready.
* NodeSelector not matched.
* NodeAffinity not matched.
* Node is tainted without toleration.
* Node under disk or memory pressure.

### **2. Scoring**

Scheduler scores remaining nodes and picks the highest:

* Prefers nodes with more free resources.
* Prefers nodes with required images cached.
* Spreads pods across zones/nodes.

### **Example Manifest**

```yaml
a piVersion: v1
kind: Pod
metadata:
  name: scheduler-demo
spec:
  containers:
    - name: app
      image: nginx
  nodeSelector:
    env: prod
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: instance-type
                operator: In
                values:
                  - high-memory
```

### **Field Explanation**

* **nodeSelector.env: prod** → Pod must run on nodes labeled `env=prod`.
* **nodeAffinity.requiredDuring...** → Hard rule, must match.
* **matchExpressions** → Used for matching conditions.
* **instance-type: high-memory** → Ensures scheduling only on specific node types.

## **Common Issues / Errors**

* Pod stuck in Pending due to resource shortage.
* Pod not scheduled because labels don’t match.
* Wrong or missing node labels.
* Node taints block scheduling.
* Scheduler can’t place pods due to strict constraints.

## **Troubleshooting / Fixes**

* Check scheduling errors using `kubectl describe pod`.
* Verify node labels: `kubectl get nodes --show-labels`.
* Check resource usage: `kubectl top nodes`.
* Add correct labels to nodes.
* Remove or adjust scheduling constraints if too strict.

## **Best Practices / Tips**

* Avoid unnecessary hard constraints.
* Use meaningful node labels.
* Monitor cluster resource usage.
* Use affinity/anti-affinity and spread constraints for HA.

---
---

# NodeSelector, Node Affinity & Pod Affinity/Anti-Affinity — Detailed Explanation

## **Concept / What**

Node selectors, node affinity, and pod affinity/anti-affinity control how Kubernetes schedules pods onto nodes. They allow you to choose specific nodes or groups of nodes based on labels.

---

## **Why / Purpose / Use Case in Real-World**

* Ensure pods run on specific types of nodes.
* Isolate workloads (GPU nodes, memory-optimized nodes, spot nodes).
* Control high availability by spreading pods across nodes or zones.
* Keep dependent workloads close for low latency.
* Prevent replicas from running on the same node.

---

## **How it Works / Steps / Syntax**

### **1. Label Nodes (Required Step)**

Before scheduling based on labels:

```
kubectl label node <node-name> env=prod
kubectl label node <node-name> instance-type=high-mem
kubectl label node <node-name> env-        # remove label
kubectl get nodes --show-labels            # verify labels
```

### **2. NodeSelector (Simple, Hard Rule)**

```yaml
spec:
  nodeSelector:
    env: prod
```

Meaning: Pod runs only on nodes labeled `env=prod`.

---

### **3. Node Affinity (Advanced Rules)**

#### **A) Hard Rule: requiredDuringSchedulingIgnoredDuringExecution**

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: env
                operator: In
                values:
                  - prod
```

#### **B) Soft Rule: preferredDuringSchedulingIgnoredDuringExecution**

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 50
          preference:
            matchExpressions:
              - key: instance-type
                operator: In
                values:
                  - spot
```

### **Operators Explained & Weight Meaning (Soft Rule)**

* **Weight** → Applies only in preferredDuringSchedulingIgnoredDuringExecution.

  * Range: **1 to 100**.
  * Higher weight = Higher priority for choosing that node group.
  * Scheduler calculates scores for all preferred rules and sums weights.
  * Node with highest total score wins.

### Example Meaning:

If two preferred rules match:

* Rule A weight = 80
* Rule B weight = 20

A matching node for Rule A gets higher score → more preferred.

**Weight does NOT guarantee placement**. It only influences preference.

---

## **Operators Explained**

* **In** → key exists AND value matches listed values.
* **NotIn** → key exists AND value must not match.
* **Exists** → key must exist (value doesn’t matter).
* **DoesNotExist** → key must not exist.
* **Gt** → numeric value greater than.
* **Lt** → numeric value less than.

---

## **4. Pod Affinity**

Used to schedule pods close to each other.

```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: frontend
      topologyKey: kubernetes.io/hostname
```

Meaning: Pod must run in the same topology group as pods with `app=frontend`.

---

## **5. Pod Anti-Affinity**

Used to keep replicas apart.

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: backend
      topologyKey: kubernetes.io/hostname
```

Meaning: Each replica must run on different nodes.

---

## **6. Topology Key (Mandatory for Pod Affinity/Anti-Affinity)**

Defines the grouping level:

* `kubernetes.io/hostname` → per-node
* `topology.kubernetes.io/zone` → per AZ
* `topology.kubernetes.io/region` → per region
* **Custom keys** → behavior depends on how nodes are labeled

Scheduler groups nodes using topologyKey and applies affinity/anti-affinity rules accordingly.

---

## **Common Issues / Errors**

* Missing node labels → pod stuck in Pending.
* Strict affinity rules → no nodes match.
* Anti-affinity prevents replica placement.
* New nodes missing labels.
* Wrong topologyKey.

---

## **Troubleshooting / Fixes**

* Check scheduling errors:

```
kubectl describe pod <pod-name>
```

* Verify node labels:

```
kubectl get nodes --show-labels
```

* Add or correct node labels.
* Relax required rules if necessary.

---

## **Best Practices / Tips**

* Use nodeSelector for simple, fixed placement.
* Use nodeAffinity for advanced, flexible rules.
* Use podAntiAffinity to spread replicas.
* Use meaningful labels (env, az, instance-type).
* Avoid over-constraining pods.

---
---

# Taints & Tolerations — Detailed Explanation

## **Concept / What**

Taints are rules applied on nodes that prevent pods from being scheduled unless those pods have the matching tolerations. Tolerations are rules on the pod side that allow pods to be scheduled on tainted nodes.

---

## **Why / Purpose / Use Case in Real-World**

* Dedicated nodes (GPU, high-memory, spot instances).
* Prevent regular workloads from entering special nodes.
* Protect system nodes reserved for critical workloads.
* Handle node pressure by evicting non‑tolerating pods.
* Ensure workload isolation by blocking unwanted scheduling.

---

## **How it Works / Steps / Syntax**

### **1. Add a Taint to a Node**

```
kubectl taint nodes <node-name> key=value:NoSchedule
```

### **2. Taint Effects**

* **NoSchedule** → Pods cannot be scheduled unless they tolerate.
* **PreferNoSchedule** → Scheduler avoids placing pods.
* **NoExecute** → New pods won’t schedule, existing pods are evicted.

### **3. Remove a Taint**

```
kubectl taint nodes <node-name> key:NoSchedule-
```

### **4. Pod with Toleration**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: toleration-demo
spec:
  tolerations:
    - key: "team"
      operator: "Equal"
      value: "devops"
      effect: "NoSchedule"
  containers:
    - name: app
      image: nginx
```

### **5. Toleration Operators**

* **Equal** → Key + value must match.
* **Exists** → Only key must match (value ignored).

---

## **Real-Time Scenarios**

### **GPU Nodes**

```
kubectl taint nodes gpu-node gpu=true:NoSchedule
```

Only GPU pods with toleration run here.

### **Spot Nodes**

```
kubectl taint nodes spot-node spot=true:PreferNoSchedule
```

Avoid scheduling non‑spot workloads.

### **Maintenance / Eviction**

```
kubectl taint nodes node1 maintenance=true:NoExecute
```

Evicts pods without the toleration.

---

## **Common Issues / Errors**

* Pod stuck in Pending due to missing toleration.
* Tainted nodes causing scheduling failures.
* Wrong effect applied.
* New nodes missing taints.

---

## **Troubleshooting / Fixes**

* View taints:

```
kubectl describe node <node-name>
```

* View pod tolerations:

```
kubectl describe pod <pod-name>
```

* Fix incorrect taints or tolerations.

---

## **Best Practices / Tips**

* Taint special node groups.
* Combine taints with nodeAffinity.
* Avoid over‑tainting nodes.
* Apply correct effects based on use case.

---
---

# Node Cordoning, Draining & Maintenance — Detailed Explanation

## **Concept / What**

Node cordoning and draining are Kubernetes operations used to safely prepare a node for maintenance. Cordoning stops new pods from being scheduled on the node, while draining evicts existing pods so maintenance can be performed.

---

## **Why / Purpose / Use Case in Real-World**

* OS/kernel/node image upgrades
* Kubelet upgrades
* Hardware replacement
* Disk/CPU/memory issues on a node
* Rolling replacement in Auto Scaling Groups
* Avoid scheduling during maintenance windows

---

## **How it Works / Steps / Syntax**

### **1. Cordon (Mark Node Unschedulable)**

Prevents new pods from scheduling on the node. Existing pods continue running.

```
kubectl cordon <node-name>
```

### **2. Uncordon (Re-enable Scheduling)**

```
kubectl uncordon <node-name>
```

### **3. Drain (Evict Pods Safely)**

Evicts all schedulable pods and prepares the node for maintenance.

```
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

#### **Drain Actions**

* Automatically cordons the node
* Evicts managed pods (Deployment, ReplicaSet, StatefulSet)
* Recreates those pods on other nodes automatically
* Ignores DaemonSet pods (they cannot be evicted)
* Ignores static pods (they remain running)

---

## **DaemonSet Behavior During Drain**

DaemonSet pods are **not evicted** during drain.

* They stay running until the node is rebooted or replaced.
* They do *not* block maintenance.
* After reboot/upgrade, DaemonSet pods are automatically recreated.

This makes drain safe for maintenance even with DaemonSet pods present.

---

## **Important Drain Flags**

* **--ignore-daemonsets** → Required to avoid drain errors
* **--delete-emptydir-data** → Deletes ephemeral data
* **--force** → Force deletes unmanaged pods (not recommended)
* **--grace-period** → Configure shutdown wait time

---

## **Common Issues / Errors**

* PodDisruptionBudgets blocking eviction
* Pods stuck in Terminating due to finalizers
* DaemonSet pods still running (expected behavior)
* Missing flags causing drain failure on emptyDir or local volumes

---

## **Troubleshooting / Fixes**

* Check blocking pods (dry run):

```
kubectl drain <node> --ignore-daemonsets --dry-run=client
```

* Check PDBs:

```
kubectl get pdb
```

* Remove finalizers for stuck pods
* Use `--force` only for unmanaged pods

---

## **Best Practices / Tips**

* Drain one node at a time in production.
* Always use `--ignore-daemonsets`.
* Verify PodDisruptionBudgets before draining.
* After maintenance, always uncordon nodes.
* For ASGs, rely on rolling upgrades when possible.

---
---

# Fargate & Fargate Profiles in EKS — Detailed Explanation

## **Concept / What**

AWS Fargate is a serverless compute engine for running Kubernetes pods without managing worker nodes. In EKS, pods that match a Fargate profile run inside AWS-managed micro-VMs (Firecracker) instead of EC2 nodes. A **Fargate Profile** is an EKS configuration that defines which pods should run on Fargate based on namespace and labels.

---

## **Why / Purpose / Use Case in Real-World**

* Removes the need to manage worker nodes.
* No EC2 node groups required.
* Strong isolation (each pod gets its own micro-VM).
* Pay only for pod runtime.
* Ideal for small services, low-traffic workloads, sensitive workloads, and teams avoiding node management.

Not suitable for:

* DaemonSets
* Privileged pods
* Host networking
* GPU workloads
* Large compute workloads

---

## **How it Works / Steps / Syntax**

### **1. Fargate Profiles Are Created Outside Kubernetes**

They are not Kubernetes objects. They are created in:

* AWS Console
* AWS CLI
* eksctl

Kubernetes does *not* reference or store Fargate profiles.

### **2. Fargate Profile Components**

* **Name** (e.g., `dev-profile`)
* **Pod Execution Role** (IAM role)
* **Selectors**:

  * Namespace
  * Pod labels
* **Subnets** (where Fargate ENIs are created)
* **Security group** for Fargate pods

### **3. How Pods Match a Fargate Profile**

A pod runs on Fargate when:

* Its **namespace** matches the profile selector
* Its **labels** match the profile selector

There is no field inside the Deployment/Pod manifest that explicitly says "use Fargate".
EKS automatically redirects matching pods to Fargate.

---

## **Example Fargate Profile**

```yaml
apiVersion: fargateprofile.eks.amazonaws.com/v1
kind: FargateProfile
metadata:
  name: fp-dev
spec:
  clusterName: my-eks
  podExecutionRoleArn: arn:aws:iam::111111111111:role/eks-fargate-role
  selectors:
    - namespace: dev
      labels:
        run: frontend
```

Meaning: Pods in namespace `dev` and with label `run=frontend` will run on Fargate.

---

## **Example Pod That Runs on Fargate**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: dev
  labels:
    run: frontend
spec:
  containers:
    - name: app
      image: nginx
```

Matching namespace + label → Pod runs on Fargate.

---

## **Creating a Fargate Profile (AWS Console)**

1. Go to **EKS → Your Cluster**
2. Open **Compute** → Click **Add Fargate Profile**
3. Enter profile name
4. Select **Pod Execution Role**
5. Add namespace and label selectors
6. Choose subnets
7. Select security group
8. Create profile (wait until Active)

---

## **Common Issues / Errors**

* Pod stuck in Pending: no matching profile
* Wrong namespace or missing labels
* Incorrect IAM role permissions
* Subnet or SG blocking ENI creation

---

## **Troubleshooting / Fixes**

* Describe pod events:

```
kubectl describe pod <pod-name>
```

* Check labels:

```
kubectl get pods --show-labels
```

* Confirm Fargate profile configuration:

```
aws eks describe-fargate-profile ...
```

* Fix IAM/ECR/CloudWatch log permissions

---

## **Best Practices / Tips**

* Use separate namespaces for Fargate workloads.
* Use clear labels to control which pods run on Fargate.
* Keep small services and low-traffic workloads on Fargate.
* Use EC2 nodes for heavy workloads.
* Always verify that DaemonSets do not target Fargate workloads.

---
---
---


