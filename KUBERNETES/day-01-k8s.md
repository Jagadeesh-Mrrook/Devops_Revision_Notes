# 🧩 Concept: What is Kubernetes & Why It’s Used

## 🧾 Concept / What

Kubernetes (also called **K8s**) is an **open-source container orchestration platform** used to **deploy, manage, scale, and maintain containerized applications automatically**. It ensures applications are running as desired, even if servers or containers fail.

Kubernetes acts like an **operating system for containerized workloads**, managing their lifecycle across multiple servers (called nodes) in a cluster.

---

## 🎯 Why / Purpose / Real-World Use Case

| Purpose                               | Example Use Case                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------- |
| **Automated Deployment & Management** | Deploy an Nginx web app using YAML — Kubernetes handles scheduling & networking |
| **Auto-Scaling**                      | E-commerce site scales up during sales & scales down later                      |
| **Load Balancing**                    | Distributes user traffic evenly across pods                                     |
| **Self-Healing / Fault Tolerance**    | If a node or pod fails, Kubernetes automatically replaces it                    |
| **Rolling Updates & Rollbacks**       | Deploy new app versions with zero downtime                                      |
| **Declarative Infrastructure**        | Define desired state using YAML manifests                                       |

**In short:** Kubernetes provides *high availability, scalability, and automation* for containerized workloads.

---

## ⚙️ How It Works / Steps / Syntax

### High-Level Flow

1. Developer builds & pushes a Docker image.
2. A YAML manifest is applied using `kubectl apply -f`.
3. Kubernetes schedules pods on worker nodes.
4. **Kubelet** runs containers on each node.
5. **Kube-proxy** manages networking and load balancing.
6. **Controller Manager** ensures the desired state matches the actual state.

---

## 🗂 Core Object Example — Pod

A **Pod** is the smallest deployable unit in Kubernetes. It usually runs a single container (but can run multiple containers that share storage/network).

### 📄 Pod Manifest Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

### 🔍 Field-by-Field Explanation

| Field                 | Description                                      |
| --------------------- | ------------------------------------------------ |
| `apiVersion: v1`      | Specifies which API version to use               |
| `kind: Pod`           | Declares this manifest defines a Pod             |
| `metadata.name`       | Pod name (unique within namespace)               |
| `labels`              | Key-value pairs to help identify/select this Pod |
| `spec`                | Contains the specification for the Pod           |
| `containers`          | List of containers to run inside this Pod        |
| `image: nginx:latest` | Pulls Nginx image from Docker Hub                |
| `containerPort: 80`   | Exposes port 80 inside the container             |

---

## 💥 Common Issues / Errors

| Issue                           | Likely Cause                            |
| ------------------------------- | --------------------------------------- |
| Pod stuck in `ImagePullBackOff` | Wrong image name or missing credentials |
| Pod in `CrashLoopBackOff`       | Application keeps crashing              |
| Pod not scheduling              | Node resource shortage or taints        |

---

## 🛠 Troubleshooting / Fixes

| Command                          | Description                     |
| -------------------------------- | ------------------------------- |
| `kubectl describe pod <pod>`     | View events and failure reasons |
| `kubectl logs <pod>`             | Check container logs            |
| `kubectl get events`             | View recent cluster events      |
| `kubectl exec -it <pod> -- bash` | Access pod shell for debugging  |

---

## ✅ Best Practices / Tips

* Pods are **ephemeral** — don’t use standalone Pods for production, use **Deployments**.
* Always define **labels** for selection & management.
* Add **liveness** and **readiness probes**.
* Use **Deployments** for stateless apps, **StatefulSets** for databases.
* Store manifests in Git (GitOps approach) for version control.

---

> 🧠 **Summary:** Kubernetes automates container management — deployment, scaling, recovery, and updates — ensuring consistent, reliable, and efficient application operations across clusters.

---
---

# 🧩 Concept: Kubernetes Architecture (Control Plane & Worker Nodes)

## 🧾 Concept / What

Kubernetes architecture follows a **Control Plane–Worker Node** model (formerly known as Master–Node architecture). The **Control Plane** is responsible for managing and maintaining the desired cluster state, while **Worker Nodes** are responsible for running containerized workloads.

Together, they create a **distributed, self-healing system** that ensures applications are deployed, scaled, and maintained automatically.

---

## 🎯 Why / Purpose / Real-World Use Case

| Purpose                    | Description                                                               |
| -------------------------- | ------------------------------------------------------------------------- |
| **Scalability**            | Add or remove worker nodes based on load.                                 |
| **Separation of Concerns** | Control Plane handles orchestration logic; Worker Nodes handle execution. |
| **Fault Tolerance**        | Control Plane and Worker Nodes can be replicated for high availability.   |
| **Automation**             | Automatically manages scheduling, health checks, and restarts.            |
| **Declarative Management** | Uses YAML manifests to enforce desired state consistently.                |

**Real-World Use:**
In a production cluster, if one worker node fails, the Control Plane automatically schedules affected pods onto other available nodes without downtime.

---

## ⚙️ How It Works / Steps / Syntax

1. **Developer** submits a manifest file to the **API Server** (`kubectl apply -f`).
2. The **API Server** validates and stores the configuration in **etcd** (cluster store).
3. The **Scheduler** decides which worker node to place the pod on.
4. The **Controller Manager** ensures the pod actually runs (matches desired vs actual state).
5. The **Kubelet** on the chosen worker node creates the container via the **Container Runtime**.
6. **Kube-proxy** manages network communication between pods and services.

---

## 🧠 Control Plane Components

| Component                    | Description                                                                         | Example / Real Use                                        |
| ---------------------------- | ----------------------------------------------------------------------------------- | --------------------------------------------------------- |
| **kube-apiserver**           | The front-end of the cluster; handles all external and internal API calls.          | Every `kubectl get pods` request passes through it.       |
| **etcd**                     | A distributed key-value store that saves cluster data and state.                    | Stores deployments, secrets, configs, and pod metadata.   |
| **kube-scheduler**           | Decides which node will run a pod based on resources, policies, and affinity rules. | Assigns a new pod to the node with enough CPU and memory. |
| **kube-controller-manager**  | Watches objects and ensures the desired state is maintained.                        | Recreates a pod if it crashes.                            |
| **cloud-controller-manager** | Integrates with cloud providers like AWS, Azure, or GCP.                            | In EKS, provisions load balancers (ELB) automatically.    |

---

## 💪 Worker Node Components

| Component             | Description                                                                                                 | Example / Real Use                           |
| --------------------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| **kubelet**           | Agent on each node that ensures containers are running properly. Reports node and pod status to API Server. | Restarts a crashed container automatically.  |
| **kube-proxy**        | Handles cluster networking and service routing between pods.                                                | Manages traffic flow using iptables or IPVS. |
| **Container Runtime** | Responsible for running containers. Kubernetes supports multiple runtimes.                                  | Docker, containerd, or CRI-O.                |

---

## 🗂 Example Manifest — Node (for reference)

*(Nodes are usually auto-registered, but this example shows structure)*

```yaml
apiVersion: v1
kind: Node
metadata:
  name: worker-node1
  labels:
    role: worker
spec:
  taints:
  - key: node-role.kubernetes.io/master
    effect: NoSchedule
```

### Field Explanation

| Field           | Description                                                 |
| --------------- | ----------------------------------------------------------- |
| `apiVersion`    | API version for Node object                                 |
| `kind`          | Specifies the Kubernetes object type                        |
| `metadata.name` | Node name in the cluster                                    |
| `labels`        | Used for scheduling and grouping nodes                      |
| `taints`        | Used to prevent pods from being scheduled on specific nodes |

---

## ☁️ EKS Context (Real-Time Use)

In **AWS EKS (Elastic Kubernetes Service)**:

* The **Control Plane** (API Server, etcd, Scheduler, Controller Manager) is **fully managed by AWS**.
* AWS handles availability, backups, and scaling of these components.
* You manage only the **Worker Nodes** (EC2 instances), which connect to the AWS-managed control plane.

---

## 💥 Common Issues / Errors

| Issue                    | Cause                                                    | Fix                                             |
| ------------------------ | -------------------------------------------------------- | ----------------------------------------------- |
| Node in `NotReady` state | Kubelet not running, network issues, or IAM permissions. | Restart kubelet, check logs, validate IAM role. |
| Pods not scheduling      | Scheduler failure or node taints.                        | Check `kubectl describe pod` events.            |
| API server timeout       | etcd or network issue.                                   | Validate API server logs and etcd health.       |

---

## 🛠 Troubleshooting / Fixes

| Command                                       | Purpose                                   |
| --------------------------------------------- | ----------------------------------------- |
| `kubectl get nodes -o wide`                   | View node status and details.             |
| `kubectl describe node <node>`                | Get detailed node conditions.             |
| `kubectl get componentstatuses`               | Check health of control plane components. |
| `systemctl status kubelet`                    | Validate kubelet service on worker node.  |
| `kubectl logs -n kube-system <apiserver-pod>` | Inspect control plane issues.             |

---

## ✅ Best Practices / Tips

* Always run multiple Control Plane nodes for **high availability**.
* Use **taints/tolerations** to control pod placement.
* Secure **etcd** with TLS and regular backups.
* Monitor control plane health with **Prometheus or CloudWatch**.
* Use **managed Kubernetes (like EKS)** to simplify cluster management.

---

> 🧠 **Summary:** Kubernetes architecture separates management (Control Plane) from execution (Worker Nodes). This design provides scalability, high availability, and self-healing capabilities — making Kubernetes the foundation for modern, production-grade container orchestration.

---
---

# 🧩 Concept: EKS Architecture and AWS-Managed Components

## 🧾 Concept / What

Amazon Elastic Kubernetes Service (**EKS**) is a **managed Kubernetes service** provided by AWS. It allows you to run Kubernetes clusters on AWS **without managing the control plane or etcd** yourself.

EKS runs the **same upstream Kubernetes** you’d run on-prem or on EC2, but AWS manages and maintains the control plane. You only manage the **worker nodes**, networking, and workloads.

---

## 🎯 Why / Purpose / Real-World Use Case

| Purpose                         | Description / Example                                                                      |
| ------------------------------- | ------------------------------------------------------------------------------------------ |
| **Fully Managed Control Plane** | AWS handles API Server, etcd, Scheduler, Controller Manager, and Cloud Controller Manager. |
| **High Availability**           | Control Plane runs across 3 Availability Zones automatically.                              |
| **Seamless AWS Integration**    | Integrates with IAM, VPC, ALB, CloudWatch, EBS, and Route53.                               |
| **Security & Reliability**      | AWS applies patches, manages etcd backups, and ensures encryption.                         |
| **Scalable Worker Management**  | You manage EC2/Fargate nodes and workloads easily.                                         |

**In short:** EKS provides **Kubernetes automation** without the burden of managing master components.

---

## ⚙️ How EKS Architecture Works

```
            +-------------------------------------+
            |          AWS Managed Control Plane  |
            |-------------------------------------|
            | API Server | etcd | Scheduler |     |
            | Controller Manager | CCM (AWS)      |
            +-------------------------------------+
                          |
                          |  (Secure TLS Communication)
                          |
            +-------------------------------------+
            |      Your AWS Account (Data Plane)  |
            |-------------------------------------|
            | Worker Nodes (EC2 / Fargate)        |
            | VPC, Subnets, IAM, Security Groups  |
            +-------------------------------------+
```

---

## 🧠 Components in EKS Architecture

### 🏗️ AWS-Managed Components (Control Plane)

| Component                          | Role                                                                   | Managed By |
| ---------------------------------- | ---------------------------------------------------------------------- | ---------- |
| **kube-apiserver**                 | Entry point for all cluster operations (`kubectl`, controllers, etc.). | AWS        |
| **etcd**                           | Key-value store holding cluster data and state.                        | AWS        |
| **kube-scheduler**                 | Assigns pods to worker nodes based on available resources.             | AWS        |
| **controller-manager**             | Ensures desired = actual cluster state.                                | AWS        |
| **cloud-controller-manager (CCM)** | Integrates Kubernetes with AWS services (ELB, EBS, etc.).              | AWS        |

All control plane components are replicated across multiple AZs for high availability.

---

### 💪 Customer-Managed Components (Data Plane)

| Component               | Role                                                           | Example                       |
| ----------------------- | -------------------------------------------------------------- | ----------------------------- |
| **Worker Nodes**        | EC2 instances or Fargate profiles that actually run your pods. | `t3.medium`, `m5.large`       |
| **Node Group**          | Collection of worker nodes managed together.                   | Managed Node Group            |
| **VPC / Subnets / SGs** | Networking layer for worker nodes and pods.                    | Public/private subnets.       |
| **IAM Roles**           | Grants permissions for node and service accounts.              | `AmazonEKSNodeRole`, etc.     |
| **Add-ons**             | Optional components for networking and DNS.                    | CoreDNS, kube-proxy, VPC CNI. |

---

## 🔗 Detailed Communication Flow (Internal Kubernetes Flow)

1. **kubectl apply** → User sends request to **API Server**.
2. **API Server** validates and writes desired state to **etcd**.
3. **Controller Manager** continuously **watches** the API Server. If it detects a mismatch (e.g., desired 3 pods but actual 0), it requests creation of missing objects.
4. **Scheduler** also watches the API Server. It finds unscheduled pods and assigns them to appropriate worker nodes.
5. **Kubelet** on each worker node watches the API Server. When a pod is scheduled to its node, kubelet pulls the image and starts containers using the **container runtime** (Docker/containerd).
6. **Kubelet** reports the pod’s status back to the API Server, which updates etcd.
7. **kube-proxy** configures network rules to route traffic to the new pods.

🧭 **Key Principle:** All communication happens **through the API Server** — components never talk to each other directly.

```
kubectl → API Server → etcd (desired state)
    ↑        ↓
Controller Manager ↔ Scheduler ↔ Kubelet
```

---

## ☁️ AWS EKS Communication Summary

| Direction                       | Description                                         |
| ------------------------------- | --------------------------------------------------- |
| User → API Server               | Sends commands via kubectl or AWS CLI.              |
| API Server → etcd               | Stores desired state (Deployments, Pods, Services). |
| Controller Manager → API Server | Watches for state changes and reconciles them.      |
| Scheduler → API Server          | Assigns pods to available worker nodes.             |
| Kubelet → API Server            | Creates containers and reports pod status.          |

---

## ⚙️ EKS Control Plane Responsibilities

* High availability (multi-AZ replication)
* etcd encryption & periodic backups
* Automatic version upgrades and patching
* TLS-secured API communication
* Cloud integration (via CCM)

**Result:** You don’t manage these — AWS does.

---

## 🧩 EKS Cluster Creation with `eksctl`

`eksctl` is a **CLI tool** developed by Weaveworks and AWS to create EKS clusters easily.
It uses **AWS CloudFormation** in the background to provision all resources.

### Example YAML Config

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-eks-cluster
  region: ap-south-1
  version: "1.30"
managedNodeGroups:
  - name: worker-nodes
    instanceType: t3.medium
    desiredCapacity: 2
```

Run:

```bash
eksctl create cluster -f cluster.yaml
```

This creates:

* EKS Control Plane (AWS-managed)
* Worker Nodes (EC2)
* Networking (VPC, Subnets)
* IAM roles and kubeconfig automatically

---

## 🧱 Terraform vs eksctl (When Using Terraform)

| Feature                  | eksctl                                | Terraform                                      |
| ------------------------ | ------------------------------------- | ---------------------------------------------- |
| **Purpose**              | Simplifies EKS creation (quick setup) | Full IaC control (modular, reusable)           |
| **Underlying Mechanism** | Uses AWS CloudFormation               | Uses Terraform AWS Provider                    |
| **Customization**        | Limited                               | Fully flexible (VPC, IAM, EKS, ALB, RDS, etc.) |
| **Best Use Case**        | Quick test or dev clusters            | Production-grade multi-env infra               |

🧠 **Conclusion:** If using Terraform, you don’t need `eksctl`. Terraform handles all EKS setup using IaC modules.

---

## 💥 Common Issues / Errors

| Issue                    | Likely Cause                             | Fix                             |
| ------------------------ | ---------------------------------------- | ------------------------------- |
| Node not joining cluster | Missing IAM role or wrong security group | Validate `aws-auth` ConfigMap   |
| Pods stuck in Pending    | Subnet or ENI quota issue                | Check VPC CNI settings          |
| Cluster creation failed  | IAM or CloudFormation error              | Check permissions, logs         |
| `kubectl` timeout        | Invalid kubeconfig                       | Run `aws eks update-kubeconfig` |

---

## 🛠 Troubleshooting Commands

| Command                                      | Purpose                            |
| -------------------------------------------- | ---------------------------------- |
| `aws eks describe-cluster --name my-cluster` | Check control plane health         |
| `kubectl get nodes -o wide`                  | Verify worker nodes joined cluster |
| `kubectl get pods -A`                        | View all workloads                 |
| `kubectl logs -n kube-system <pod>`          | View system component logs         |
| `eksctl get cluster`                         | List clusters (if using eksctl)    |

---

## ✅ Best Practices

* Use **Terraform** for IaC-driven cluster setup.
* Enable **Cluster Autoscaler** for scaling worker nodes.
* Integrate **CloudWatch** for logs and metrics.
* Use **private cluster endpoints** for security.
* Keep **EKS and node versions** within one minor version.
* Use **IRSA (IAM Roles for Service Accounts)** instead of node-level IAM.

---

## 🧠 Summary

EKS keeps the standard Kubernetes architecture but shifts all **Control Plane responsibilities to AWS** — including API Server, etcd, Scheduler, and Controller Manager. You only manage **worker nodes, networking, and workloads**.

AWS ensures the control plane is **highly available, secure, patched, and backed up**, so DevOps teams can focus purely on deploying and managing applications.

> **In short:** Kubernetes architecture + AWS automation = Amazon EKS.

---
---

# 🧩 Core Component 1: Kube API Server

## 🧾 Concept / What

The **Kube API Server** is the **central management and communication hub** of the Kubernetes cluster. It acts as the **front-end** for the Kubernetes control plane — all other components (controllers, scheduler, kubelet, kubectl, etc.) communicate through it.

Think of it as the **brain–to–body connector** in Kubernetes — every command or status update must pass through it.

---

## 🎯 Why / Purpose / Role

| Purpose                            | Description                                                                   |
| ---------------------------------- | ----------------------------------------------------------------------------- |
| **Central Gateway**                | Every API request (from kubectl, controllers, or kubelets) passes through it. |
| **Validation Layer**               | Validates manifests (YAML/JSON) before storing them in etcd.                  |
| **Authentication & Authorization** | Authenticates users, Pods, and components before allowing actions.            |
| **State Persistence**              | Communicates with etcd to save and retrieve cluster state.                    |
| **Extensibility**                  | Acts as an API endpoint for external automation (Terraform, ArgoCD, Jenkins). |

---

## ⚙️ How It Works (Detailed Flow)

1. A user runs a command:

   ```bash
   kubectl apply -f deployment.yaml
   ```
2. **kubectl** sends an HTTPS REST API request to the **Kube API Server**.
3. The API Server performs:

   * **Authentication:** Who is the requester (IAM user, ServiceAccount, etc.)?
   * **Authorization:** Are they allowed to perform this action? (RBAC, IAM mapping)
   * **Admission Control:** Any policies or validations (e.g., PodSecurity, ResourceQuota).
4. Once validated, the **object is written to etcd** (cluster database).
5. Controllers and Scheduler **watch the API Server** for updates and act accordingly.
6. Worker nodes (kubelets) **report Pod and node status** back to the API Server.

📌 **Important:** All communication (even internal) happens through the API Server — components never talk to each other directly.

---

## 🔁 Communication Pattern

```
[kubectl / Controllers / Kubelets]
          │
          ▼
     [Kube API Server] ←→ [etcd]
          ▲
          │
    [Controllers & Scheduler Watch Updates]
```

---

## 🧠 Related Authentication Mechanisms

The API Server enforces **who** can talk to the cluster and **how**. In EKS, this spans three key mechanisms.

### 🧩 1️⃣ IAM Authentication (Human / Tool → API Server)

Used when **you** or tools (like Jenkins, Terraform) access the cluster.

* EKS integrates AWS IAM with the cluster using the **`aws-auth` ConfigMap**.
* IAM roles or users are mapped to Kubernetes RBAC roles.

Example:

```yaml
mapRoles: |
  - rolearn: arn:aws:iam::111122223333:role/DevOpsAdminRole
    username: admin
    groups:
      - system:masters
```

✅ Allows authenticated users to run commands like `kubectl get pods`.

---

### 🧩 2️⃣ ServiceAccount JWT (Pod → API Server)

Used when **Pods inside the cluster** communicate with the API Server.

* Every Pod has a **ServiceAccount** (default or custom).
* The ServiceAccount has a **JWT token** mounted at:

  ```bash
  /var/run/secrets/kubernetes.io/serviceaccount/token
  ```
* The Pod uses that token to authenticate requests to the API Server.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: app-service-account
  containers:
    - name: app
      image: nginx
```

✅ Used for internal communication (Pod → API Server).

---

### 🧩 3️⃣ OIDC + IRSA (Pod → AWS Services)

Used when Pods need to access AWS services like S3, Secrets Manager, or CloudWatch.

* AWS creates an **OIDC provider** URL for the EKS cluster automatically:

  ```
  https://oidc.eks.ap-south-1.amazonaws.com/id/ABCDEF1234567890
  ```
* You create an **IAM Role** that trusts this OIDC provider.
* You link it to a **Kubernetes ServiceAccount** using an annotation:

  ```yaml
  metadata:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/S3AccessRole
  ```

✅ This enables **IRSA (IAM Roles for Service Accounts)** — secure AWS access without credentials.

---

## 🧩 Real-Time Example Flow

```
Human (kubectl)  →  API Server  →  etcd
Controller       →  API Server  →  Watches desired state
Scheduler        →  API Server  →  Assigns Pod → Node
Kubelet          →  API Server  →  Creates containers and reports status
Pod (ServiceAccount) → API Server → Auth via JWT
Pod (IRSA) → AWS STS via OIDC → Temporary AWS creds
```

---

## 💥 Common Issues

| Issue                       | Root Cause                                | Fix                                        |
| --------------------------- | ----------------------------------------- | ------------------------------------------ |
| `403 Forbidden`             | IAM or RBAC misconfiguration              | Check `aws-auth` ConfigMap and RBAC roles  |
| `kubectl` timeout           | Endpoint not reachable / wrong kubeconfig | Verify cluster endpoint and token          |
| API latency                 | etcd overload or high API traffic         | Scale control plane or reduce watcher load |
| Pod cannot reach API Server | ServiceAccount missing or corrupted       | Recreate SA or check network policies      |

---

## 🛠 Troubleshooting Commands

| Command                                       | Purpose                                        |
| --------------------------------------------- | ---------------------------------------------- |
| `kubectl get componentstatuses`               | View control plane health                      |
| `kubectl get --raw='/healthz'`                | API Server health check                        |
| `kubectl logs -n kube-system <apiserver-pod>` | Inspect API Server logs (for self-managed K8s) |
| `aws eks describe-cluster --name <cluster>`   | Check API endpoint and OIDC in EKS             |
| `kubectl config view`                         | View kubeconfig details and contexts           |

---

## ✅ Best Practices

* Never expose the API Server publicly unless necessary (use private endpoints in EKS).
* Use **RBAC** and **IAM roles** to enforce least privilege.
* Rotate **ServiceAccount tokens** regularly (K8s v1.24+ auto rotates).
* Enable **audit logs** for API Server.
* Use **OIDC + IRSA** instead of embedding AWS credentials.
* Monitor API Server performance using **CloudWatch** or **Prometheus**.

---

## 🧠 Summary

| Layer                     | Purpose                         | Auth Mechanism           |
| ------------------------- | ------------------------------- | ------------------------ |
| **Human → Cluster**       | Manage Kubernetes via `kubectl` | IAM + aws-auth ConfigMap |
| **Pod → Kube API Server** | Communicate with cluster APIs   | ServiceAccount JWT       |
| **Pod → AWS Services**    | Access AWS resources securely   | OIDC + IRSA              |

**In short:** The **Kube API Server** is the beating heart of the Kubernetes control plane — all actions flow through it. In EKS, AWS enhances it with IAM and OIDC integration for secure, automated access from both users and workloads.

---
---

# 🧩 Core Component 2: etcd

## 🧾 Concept / What

**etcd** is a **distributed, consistent, and highly available key–value store** used by Kubernetes to store **all cluster data**. It acts as the **database of the Kubernetes control plane**, holding the desired and actual state of every resource in the cluster — such as Pods, Deployments, ConfigMaps, Secrets, and Nodes.

If etcd is lost or corrupted, the cluster loses its state information.

---

## 🎯 Why / Purpose

| Purpose                      | Description                                                                                 |
| ---------------------------- | ------------------------------------------------------------------------------------------- |
| **Cluster State Management** | etcd stores every Kubernetes object as a key-value pair.                                    |
| **Consistency**              | Ensures consistent state across all control plane nodes using the Raft consensus algorithm. |
| **High Availability**        | Replicates data across multiple control plane nodes.                                        |
| **Source of Truth**          | Control plane components always read and write to etcd for accurate cluster data.           |
| **Disaster Recovery**        | A backup of etcd can fully restore a cluster.                                               |

---

## ⚙️ How etcd Works (Step-by-Step)

### 📋 Workflow Between API Server and etcd

1. You run:

   ```bash
   kubectl apply -f deployment.yaml
   ```
2. The **Kube API Server** validates and stores this information in etcd as a key-value pair.
3. etcd **replicates** this data across all master nodes (for HA).
4. Other components (scheduler, controller-manager) **watch the API Server** for updates — the API Server retrieves data from etcd.
5. When a Pod’s status changes, the **Kubelet** reports it back to the API Server, which updates etcd.

📌 **All data flow in Kubernetes passes through the API Server — never directly to etcd.**

---

## 🧠 Data Stored in etcd

Everything in the cluster is stored as a key-value pair, organized by resource type and namespace.

| Object     | etcd Key Path                                 |
| ---------- | --------------------------------------------- |
| Pod        | `/registry/pods/default/nginx-pod`            |
| Deployment | `/registry/deployments/default/web-app`       |
| ConfigMap  | `/registry/configmaps/default/db-config`      |
| Secret     | `/registry/secrets/default/db-password`       |
| Node       | `/registry/minions/ip-10-0-1-10.ec2.internal` |

---

## ⚙️ Internal Architecture

```
[Kube API Server]
      │
      ▼
     etcd
  (Key-Value Store)
```

When you run `kubectl get pods`, the API Server reads the requested data from etcd and returns it to you.

---

## 🧩 Consensus & Reliability

### 🔹 Raft Algorithm

* etcd uses the **Raft consensus algorithm** to ensure consistency among all its members.
* One node acts as a **Leader**; others are **Followers**.
* The Leader handles all writes and replicates changes to the followers.

### 🔹 High Availability Setup

* etcd is deployed as an **odd-number cluster** (3, 5, or 7 nodes).
* A majority (quorum) must agree before any write is committed.
* This design ensures that even if one node fails, etcd continues functioning.

---

## ☁️ etcd in Amazon EKS (AWS-Managed)

In **Amazon EKS**, etcd is **fully managed by AWS** as part of the control plane. You don’t install or maintain it manually.

| Feature                | Description                                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------------- |
| **High Availability**  | AWS automatically deploys 3 control plane (master) nodes across 3 Availability Zones (AZs). |
| **Data Replication**   | etcd data is replicated synchronously across all AZs.                                       |
| **Encryption**         | etcd is encrypted at rest using **AWS KMS**.                                                |
| **Backups & Recovery** | AWS handles etcd backups and restores internally.                                           |
| **Scaling**            | AWS auto-scales the control plane based on request load.                                    |
| **Isolation**          | Control plane runs in an AWS-managed VPC (not visible to your account).                     |

You **cannot directly access or manage etcd** in EKS — you interact with it only through the **Kube API Server** endpoint.

---

## 🧱 AWS Multi-Master Control Plane (EKS Control Plane Management)

### 🧠 How AWS Manages Multi-Master HA

* When you create an EKS cluster, AWS automatically provisions **three control plane nodes** (masters) in **three different Availability Zones**.
* Each master runs:

  * kube-apiserver
  * etcd (replicated)
  * controller-manager
  * scheduler
* These are fronted by an **AWS-managed Network Load Balancer** for high availability.

### ⚙️ Scaling Behavior

* You **don’t configure or see** the control plane nodes directly.
* AWS always maintains 3 masters (minimum) across AZs.
* If API traffic increases, AWS automatically scales the control plane vertically.
* AWS monitors and replaces unhealthy masters automatically.

### 🔐 Security & Reliability

| Feature                    | Description                                         |
| -------------------------- | --------------------------------------------------- |
| **Multi-AZ Redundancy**    | 3 masters in separate AZs.                          |
| **Automatic Failover**     | If one fails, others continue serving API requests. |
| **Managed Patching**       | AWS applies updates and patches.                    |
| **Encrypted etcd**         | All data encrypted with AWS KMS.                    |
| **Load-Balanced Endpoint** | AWS manages endpoint access (public/private).       |

📌 **You manage only the worker nodes (data plane); AWS handles all control plane operations.**

---

## 💾 Backup & Restore (Self-Managed Clusters)

For clusters created using kubeadm or kOps, you must handle etcd backups manually.

### Backup:

```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Restore:

```bash
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd-snapshot.db \
  --data-dir /var/lib/etcd-from-backup
```

---

## 💥 Common Issues

| Issue                           | Root Cause                             | Fix                                   |
| ------------------------------- | -------------------------------------- | ------------------------------------- |
| Cluster read-only               | Lost quorum or leader election failure | Restore etcd quorum or restart nodes  |
| High API latency                | etcd I/O bottleneck or network delay   | Check etcd health and latency metrics |
| API Server “connection refused” | etcd down or cert expired              | Restart etcd, renew certs             |
| Data corruption                 | Disk issues or power loss              | Restore from latest backup            |

---

## 🛠 Troubleshooting Commands (Self-Managed)

```bash
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 --cacert=ca.crt --cert=client.crt --key=client.key

ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 --cacert=ca.crt --cert=client.crt --key=client.key
```

AWS EKS users do **not** run these manually — AWS manages etcd monitoring and health checks.

---

## ✅ Best Practices

* Always **encrypt etcd data at rest** (AWS does this by default).
* **Do not modify etcd directly**; interact via API Server.
* **Snapshot etcd** regularly (if self-managed).
* Run an **odd number of etcd nodes** for quorum.
* Use **dedicated SSD storage** for etcd in self-managed clusters.
* Monitor **etcd latency and leader elections** using metrics or CloudWatch.

---

## 🧠 Summary

| Concept          | Description                                                         |
| ---------------- | ------------------------------------------------------------------- |
| **What**         | Distributed key-value store for all cluster data.                   |
| **Why**          | Ensures consistent and reliable cluster state.                      |
| **How**          | API Server writes and reads data through etcd using Raft consensus. |
| **EKS**          | AWS fully manages etcd (multi-AZ, encrypted, backed up).            |
| **Self-managed** | You must configure, secure, and back it up manually.                |

> 🧩 **In short:** etcd is the **heart of Kubernetes’ data storage**, and in EKS, AWS ensures it’s always replicated, encrypted, and highly available — freeing you from all control plane management.

---
---

# 🧩 Core Component 3: Controller Manager

## 🧾 Concept / What

The **Kubernetes Controller Manager** is the **automation engine** of the control plane. It continuously watches the cluster’s state through the API Server and ensures that the **actual state matches the desired state** stored in etcd. It performs automatic actions to maintain, repair, or scale the cluster as needed.

> Think of it as Kubernetes’ “self-healing and automation brain.”

---

## 🎯 Why / Purpose

| Purpose                     | Description                                                              |
| --------------------------- | ------------------------------------------------------------------------ |
| **Maintain Desired State**  | Detects and fixes mismatches between actual and desired cluster state.   |
| **Automation**              | Automates replication, node lifecycle, and endpoint management.          |
| **Self-Healing**            | Recreates Pods, replaces failed nodes, and updates status automatically. |
| **Controller Coordination** | Hosts multiple built-in controllers (ReplicaSet, Node, Job, etc.).       |

---

## ⚙️ How It Works (Step-by-Step)

1. You apply a manifest defining the **desired state** (e.g., 3 replicas for a Deployment).
2. The **API Server** stores this in etcd.
3. The **Controller Manager** constantly monitors the API Server for resource changes.
4. If it detects a difference (e.g., 2 Pods running instead of 3), it instructs the API Server to fix it.
5. Other components (like the Scheduler and Kubelet) carry out the changes.

✅ This continuous feedback loop maintains Kubernetes’ declarative nature.

---

## 🧩 Common Controllers

| Controller                              | Responsibility                                            |
| --------------------------------------- | --------------------------------------------------------- |
| **Node Controller**                     | Detects and responds to node failures.                    |
| **Replication / ReplicaSet Controller** | Ensures the specified number of Pod replicas are running. |
| **Deployment Controller**               | Handles rolling updates and rollbacks.                    |
| **Job Controller**                      | Manages batch jobs that run until completion.             |
| **DaemonSet Controller**                | Ensures one Pod per node.                                 |
| **Service Controller**                  | Creates or updates LoadBalancers for Services.            |
| **Endpoint Controller**                 | Updates Service endpoints based on Pod IPs.               |
| **Namespace Controller**                | Cleans up resources when a namespace is deleted.          |
| **PersistentVolume Controller**         | Manages PV and PVC bindings.                              |
| **ServiceAccount Controller**           | Creates and manages default ServiceAccounts and tokens.   |

All of these controllers run as part of a single binary process: `kube-controller-manager`.

---

## 🧠 Internal Working Diagram

```
+--------------------+
|    etcd (state)    |
+--------------------+
          ▲
          │
   (watch via API)
          │
+--------------------+
| Controller Manager |
| - Node Controller  |
| - ReplicaSet Ctrl  |
| - Job Controller   |
| - Service Ctrl     |
| - PV Controller    |
+--------------------+
          │
          ▼
   (update via API)
          │
+--------------------+
|   Kube API Server  |
+--------------------+
```

Controllers **watch** the API Server and **act** by sending updates back to it.

---

## ☁️ Controller Manager in Amazon EKS

In **EKS**, the Controller Manager is **AWS-managed**. You don’t deploy or control it directly.

| Function                            | Responsibility |
| ----------------------------------- | -------------- |
| Run Controller Manager              | ✅ AWS-managed  |
| Apply patches/upgrades              | ✅ AWS-managed  |
| Maintain availability               | ✅ AWS-managed  |
| Deploy custom controllers/operators | 🧑‍💻 You      |
| Configure autoscaling (HPA/VPA)     | 🧑‍💻 You      |

So in EKS:

* You **define resources** (Deployments, DaemonSets, Jobs, etc.)
* AWS ensures the controllers act on them automatically.

---

## 🔍 Real-Time Example — Self-Healing

**Scenario:**
You create a Deployment with 3 replicas of an app.

* One Pod crashes.
* The **ReplicaSet Controller** detects actual = 2, desired = 3.
* It creates a new Pod instantly.
* The **Endpoint Controller** updates the Service endpoints.

✅ Result: Application recovers automatically — no manual action needed.

---

## 💥 Common Issues

| Issue                          | Root Cause                            | Fix                                            |
| ------------------------------ | ------------------------------------- | ---------------------------------------------- |
| Replica mismatch               | Controller stalled or API lagging     | Restart kube-controller-manager (self-managed) |
| Pods not recreated             | Node Controller timeout misconfigured | Tune `--node-monitor-grace-period`             |
| Service endpoints not updating | Endpoint Controller delayed           | Check Service selector and Pod labels          |
| PV/PVC stuck in Pending        | Label mismatch or missing claimRef    | Verify PVC and PV selectors                    |

---

## 🛠 Troubleshooting

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl describe deployment <name>
kubectl get replicasets,daemonsets,jobs,statefulsets
```

For self-managed clusters:

```bash
journalctl -u kube-controller-manager
```

In EKS, controller logs are managed internally by AWS.

---

## ✅ Best Practices

* Always use **Deployments** and **ReplicaSets** (avoid standalone Pods).
* Configure **HPA (Horizontal Pod Autoscaler)** for dynamic scaling.
* Set **resource requests/limits** so autoscaling controllers work correctly.
* Use **DaemonSets** for node-wide workloads (like logging agents).
* Keep controller versions consistent during upgrades.

---

## 🧠 Summary

| Concept         | Description                                                          |
| --------------- | -------------------------------------------------------------------- |
| **What**        | Automation engine that maintains the cluster’s desired state.        |
| **Why**         | Ensures self-healing and automation across resources.                |
| **How**         | Watches API Server and acts to align actual and desired states.      |
| **EKS Context** | Fully managed by AWS; you define workloads, AWS handles controllers. |
| **Example**     | ReplicaSet Controller recreates missing Pods automatically.          |

> 🧩 **In short:** The **Controller Manager** is Kubernetes’ internal automation system — always watching, reacting, and repairing so your cluster remains in the desired state.

---
---

# 🧩 Core Component 4: Scheduler

## 🧾 Concept / What

The **Kubernetes Scheduler** is responsible for deciding **which node** a newly created Pod should run on. It continuously monitors the cluster for **unscheduled Pods** and assigns them to nodes based on **resource availability, policies, and constraints**.

> In simple terms: The **Controller Manager** decides a Pod *should* exist; the **Scheduler** decides *where* it should run.

---

## 🎯 Why / Purpose

| Purpose                | Description                                                            |
| ---------------------- | ---------------------------------------------------------------------- |
| **Optimal Placement**  | Assigns Pods to nodes efficiently based on CPU, memory, and resources. |
| **Policy Enforcement** | Enforces affinity, anti-affinity, and taint/toleration rules.          |
| **Resource Fairness**  | Prevents hotspots by balancing workloads across nodes.                 |
| **Resilience**         | Reschedules Pods if nodes fail or become unschedulable.                |

---

## ⚙️ How It Works (Step-by-Step)

1. **Controller Manager creates an unscheduled Pod.**
   The Pod starts in a `Pending` state (no node assigned).

2. **Scheduler watches for Pending Pods.**
   It continuously checks the API Server for Pods without a `spec.nodeName`.

3. **Filtering Phase (Feasibility Check)**
   The Scheduler filters nodes that can run the Pod based on:

   * Node taints/tolerations
   * Node selectors and affinity rules
   * Resource requests/availability
   * Pod anti-affinity rules

4. **Scoring Phase (Ranking)**
   It scores eligible nodes based on factors like:

   * Available CPU/memory
   * Data locality
   * Image locality
   * Topology spread
   * Affinity preferences

5. **Binding Phase**
   The Scheduler updates the Pod’s spec with the chosen node name and sends it to the API Server:

   ```yaml
   spec:
     nodeName: ip-10-0-2-101.ec2.internal
   ```

   The **Kubelet** on that node picks up the Pod and starts it.

---

## 🧠 Internal Architecture

```
[Controller Manager] --> Creates Pending Pod
          │
          ▼
[Scheduler] --> Filters nodes → Scores → Selects one
          │
          ▼
[API Server] --> Updates Pod with nodeName
          │
          ▼
[Kubelet] --> Pulls image, starts containers
```

---

## ⚙️ Scheduling Policies & Rules

| Mechanism                         | Description                                 | Example                                          |
| --------------------------------- | ------------------------------------------- | ------------------------------------------------ |
| **Node Selector**                 | Hard binding based on node labels           | `nodeSelector: { disktype: ssd }`                |
| **Node Affinity / Anti-Affinity** | Preferred or required placement rules       | `requiredDuringSchedulingIgnoredDuringExecution` |
| **Pod Affinity / Anti-Affinity**  | Schedule Pods together or apart             | Web Pod and Cache Pod colocated                  |
| **Taints & Tolerations**          | Control which Pods can run on tainted nodes | `key=value:NoSchedule`                           |
| **Topology Spread Constraints**   | Spread Pods evenly across zones or nodes    | `topologyKey: topology.kubernetes.io/zone`       |
| **Resource Requests/Limits**      | Ensure node has required CPU/memory         | `resources.requests.cpu: 500m`                   |

---

## ☁️ Scheduler in Amazon EKS

In **EKS**, the Scheduler is **fully AWS-managed**.

| Responsibility              | Managed By             |
| --------------------------- | ---------------------- |
| Scheduler process & HA      | AWS                    |
| Scaling and patching        | AWS                    |
| Custom scheduling policies  | You                    |
| Deploying custom schedulers | You (as separate Pods) |

You can’t modify the default EKS Scheduler, but you can **deploy your own custom scheduler** for special placement logic.

---

## 🔍 Real-World Scenarios

| Scenario                        | Description / Fix               |                                    |
| ------------------------------- | ------------------------------- | ---------------------------------- |
| Pod in `Pending`                | No node meets resource requests | Check `kubectl describe pod`       |
| Pod not scheduled due to taint  | Missing toleration              | Add `tolerations:` in YAML         |
| Pod stuck due to affinity rules | Overly strict constraints       | Relax affinity conditions          |
| Pod scheduling delayed          | Cluster full or node pressure   | Add more nodes or scale node group |
| Pods unevenly distributed       | Missing topology spread         | Add zone spread constraints        |

---

## 💥 Common Issues

| Issue                  | Root Cause                      | Fix                                       |
| ---------------------- | ------------------------------- | ----------------------------------------- |
| Pod `Pending` forever  | No suitable node                | Scale nodes or relax resource requests    |
| Scheduler high latency | Too many pending Pods or nodes  | Optimize filters or use custom schedulers |
| Unintended placement   | Incorrect nodeSelector/affinity | Validate YAML configuration               |
| Taint issues           | Missing Pod toleration          | Add toleration to Pod spec                |

---

## 🛠 Troubleshooting Commands

```bash
# Check scheduling errors
kubectl describe pod <pod-name>

# Check node details
kubectl get nodes -o wide
kubectl describe node <node-name>

# View node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# List scheduling events
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## ✅ Best Practices

* Always define **resource requests and limits** in Pod specs.
* Use **affinity and topology rules** for controlled placement.
* Apply **taints/tolerations** for dedicated workloads (e.g., GPU nodes).
* Use **Cluster Autoscaler** for dynamic node scaling.
* Monitor **scheduling latency** via Prometheus or CloudWatch.
* For complex workloads, create **custom schedulers**.

---

## 🧠 Summary

| Concept         | Description                                                        |
| --------------- | ------------------------------------------------------------------ |
| **What**        | Decides which node a Pod should run on.                            |
| **Why**         | Ensures optimal and policy-compliant Pod placement.                |
| **How**         | Filters and scores nodes based on available resources and rules.   |
| **EKS Context** | AWS manages the Scheduler; you define placement logic.             |
| **Example**     | Pending Pod assigned to best-fit node after filtering and scoring. |

> 🧩 **In short:** The **Kube Scheduler** is Kubernetes’ decision-maker — it ensures each Pod lands on the right node efficiently, following resource availability and defined scheduling policies.


---
---

# 🧩 Core Component 5: Kubelet

## 🧾 Concept / What

The **Kubelet** is an **agent that runs on every worker node** in a Kubernetes cluster. Its main job is to communicate with the **Kube API Server** and ensure that the containers described in Pod specifications are running and healthy.

> 🧩 In simple terms: The **API Server** tells *what* should run, and the **Kubelet** makes sure it *is* running.

---

## 🎯 Why / Purpose

| Purpose                          | Description                                                                     |
| -------------------------------- | ------------------------------------------------------------------------------- |
| **Pod Lifecycle Management**     | Ensures that all containers defined in PodSpecs are created and healthy.        |
| **Node-API Communication**       | Acts as a bridge between the worker node and the control plane.                 |
| **Health Monitoring**            | Continuously checks container and node health and reports it to the API Server. |
| **Container Runtime Management** | Interacts with Docker, containerd, or CRI-O to manage containers.               |
| **Self-Healing**                 | Restarts or replaces containers that fail health checks.                        |

---

## ⚙️ How It Works (Step-by-Step)

1️⃣ **Node Registration** – When a node starts, the Kubelet registers itself with the API Server. The node then appears as `Ready` in the cluster.

2️⃣ **Pod Assignment Watch** – Kubelet constantly watches the API Server for Pods assigned to its node (`spec.nodeName`).

3️⃣ **Pod Creation** – Upon receiving a PodSpec, Kubelet:

* Pulls the container images via the **Container Runtime**.
* Creates containers.
* Mounts volumes and sets environment variables.

4️⃣ **Health Checks** – Runs readiness and liveness probes to ensure Pods are healthy.

5️⃣ **Status Reporting** – Continuously sends node and Pod status back to the API Server.

6️⃣ **Cleanup** – Removes terminated Pods and unused images.

---

## 🧠 Internal Architecture

```
[Kube API Server]
       │
       ▼
[Kubelet]
  ├── Watches API Server
  ├── Manages Pods & Containers
  ├── Reports Node Status
       │
       ▼
[Container Runtime (containerd, Docker, CRI-O)]
       │
       ▼
[Running Containers]
```

---

## ⚙️ Kubelet Configuration Example

```bash
/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

Example options:

```bash
ExecStart=/usr/bin/kubelet \
  --kubeconfig=/etc/kubernetes/kubelet.conf \
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --pod-manifest-path=/etc/kubernetes/manifests
```

📌 The `--pod-manifest-path` flag tells Kubelet where to look for **static Pods** (like API Server and etcd on master nodes).

---

## 🧱 Container Runtime Interface (CRI)

The Kubelet uses the **Container Runtime Interface (CRI)** to communicate with container runtimes:

| Runtime                 | Description                                        |
| ----------------------- | -------------------------------------------------- |
| **containerd**          | Default runtime for most EKS and upstream clusters |
| **Docker (dockershim)** | Deprecated after Kubernetes v1.24                  |
| **CRI-O**               | Lightweight alternative used in OpenShift          |

Kubelet → CRI → containerd → runc (creates container processes)

---

## ☁️ Kubelet in Amazon EKS – Installation & Registration Process

### 🧱 1️⃣ Managed Node Groups

* When you create an **EKS Managed Node Group**, AWS automatically launches EC2 instances using an **EKS-Optimized AMI**.
* This AMI already contains:

  * `kubelet`
  * `kube-proxy`
  * `containerd`
  * AWS IAM Authenticator
* On startup, a **user data script** automatically:

  * Starts the Kubelet service.
  * Connects it to the cluster’s API Server.
  * Registers the node.

✅ **No manual installation needed** — AWS handles it.

### 🧱 2️⃣ Self-Managed Node Groups

If you create your own EC2 instances, you can use the same **EKS AMI**, which already includes Kubelet. Run:

```bash
/etc/eks/bootstrap.sh <cluster-name>
```

This script:

* Starts the Kubelet.
* Passes cluster endpoint and certificate details.
* Registers the node automatically.

### 🧱 3️⃣ Fargate

* Fargate uses a **virtual Kubelet**, managed entirely by AWS.
* You don’t see or manage it, but it reports workloads as normal Pods.

### 🧠 Summary

| Node Type              | Who Manages Kubelet        | Visible to You | Notes                         |
| ---------------------- | -------------------------- | -------------- | ----------------------------- |
| **Managed Node Group** | AWS                        | ✅ Yes          | Preinstalled and auto-started |
| **Self-Managed Node**  | You (via bootstrap script) | ✅ Yes          | Preinstalled AMI              |
| **Fargate**            | AWS (virtual)              | ❌ No           | Hidden and fully managed      |

---

## 🔍 Verification Commands

```bash
# Check kubelet status
systemctl status kubelet

# Restart kubelet
sudo systemctl restart kubelet

# View version
kubelet --version
```

---

## 💥 Common Issues

| Issue                            | Root Cause                          | Fix                                   |
| -------------------------------- | ----------------------------------- | ------------------------------------- |
| Node `NotReady`                  | Kubelet not reporting to API Server | Restart kubelet or fix network        |
| Pod stuck in `ContainerCreating` | Image pull issue                    | Check registry access or IAM role     |
| Pod stuck in `Terminating`       | Kubelet failed cleanup              | Restart kubelet or check runtime logs |
| Evicted Pods                     | Disk or memory pressure             | Adjust eviction thresholds            |

---

## 🛠 Troubleshooting Commands

```bash
journalctl -u kubelet -f
kubectl describe node <node-name>
kubectl get pods -o wide
```

---

## ✅ Best Practices

* Use **EKS optimized AMIs** for correct kubelet configuration.
* Set resource reservations with `--kube-reserved`.
* Enable **Node Problem Detector** for node health.
* Enable **Kubelet certificate rotation**.
* Monitor node resource pressure and eviction behavior.

---

## 🧠 Summary

| Concept         | Description                                                                 |
| --------------- | --------------------------------------------------------------------------- |
| **What**        | Node-level agent managing Pods and containers.                              |
| **Why**         | Ensures containers run as per Pod definitions and report health.            |
| **How**         | Watches API Server, manages containers via CRI, reports status.             |
| **EKS Context** | Preinstalled by AWS on managed/self-managed nodes; virtualized in Fargate.  |
| **Example**     | Restarts crashed containers automatically and reports status to API Server. |

> 🧩 **In short:** The **Kubelet** is the heart of the worker node — it keeps the node connected to the control plane and ensures that all assigned Pods are running, healthy, and reported accurately.

---
---

# 🧩 Core Component 6: Kube-Proxy

## 🧾 Concept / What

**Kube-Proxy** is a **networking agent** that runs on every node in a Kubernetes cluster. It maintains network rules to enable communication between Pods and Services, handling load balancing and routing across nodes.

> 🧩 In simple terms: **Kube-Proxy** ensures network connectivity and traffic routing between Pods and Services.

---

## 🎯 Why / Purpose

| Purpose                    | Description                                                           |
| -------------------------- | --------------------------------------------------------------------- |
| **Pod-to-Service Routing** | Forwards traffic from a Service to the correct backend Pods.          |
| **Load Balancing**         | Distributes network requests across multiple Pods.                    |
| **Stable Connectivity**    | Allows communication via Service IPs rather than ephemeral Pod IPs.   |
| **Networking Rules**       | Maintains iptables/IPVS/eBPF rules to control routing and forwarding. |

---

## ⚙️ How It Works (Step-by-Step)

1️⃣ You create a Service:

```bash
kubectl expose deployment nginx --port=80 --type=ClusterIP
```

2️⃣ Kubernetes assigns a **ClusterIP** (virtual IP) to the Service and tracks backend Pod IPs.

3️⃣ Each node’s **Kube-Proxy** updates iptables/IPVS rules:

* Maps Service IP (e.g., 10.96.0.15) to Pod IPs (e.g., 10.244.1.2, 10.244.2.3).

4️⃣ When a request reaches the Service IP, **Kube-Proxy** redirects traffic to one of the backend Pods.

5️⃣ It updates these rules dynamically as Pods scale or restart.

---

## 🧠 Internal Architecture

```
+---------------------------+
|        Node (Worker)      |
|---------------------------|
|   Kubelet                 |
|   Kube-Proxy              | ---> Maintains iptables/IPVS rules
|   Container Runtime        |
+---------------------------+

[Service ClusterIP] --> [Pod 1 IP]
                       --> [Pod 2 IP]
                       --> [Pod 3 IP]
```

---

## ⚙️ Modes of Operation

| Mode              | Description                                         | Notes                         |
| ----------------- | --------------------------------------------------- | ----------------------------- |
| **iptables**      | Uses Linux iptables for NAT-based forwarding        | Default mode in most clusters |
| **IPVS**          | Uses Linux IP Virtual Server for higher performance | Best for large-scale clusters |
| **eBPF (Cilium)** | Kernel-level packet filtering and forwarding        | Advanced and modern mode      |

---

## 🧩 Example (iptables Mode)

```bash
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers
```

Example routing:

```
Service 10.96.0.15:80
 ├─> Pod 10.244.1.2:80
 ├─> Pod 10.244.2.3:80
```

Requests to 10.96.0.15:80 are load balanced between backend Pods.

---

## ☁️ Kube-Proxy in Amazon EKS

In **EKS**, **Kube-Proxy** runs as a **DaemonSet** — one Pod per worker node.

| Area                | Managed By                 | Description                                   |
| ------------------- | -------------------------- | --------------------------------------------- |
| **Deployment**      | AWS (add-on or user)       | Included by default in EKS AMI                |
| **Mode**            | User configurable          | Default: `iptables`                           |
| **Upgrade & Patch** | AWS (if managed as add-on) | Updated automatically during version upgrades |
| **CNI Integration** | User                       | Works with AWS VPC CNI for Pod IP assignment  |

AWS now supports **Kube-Proxy as a managed add-on**, so you can upgrade or roll back easily via CLI or console.

---

## 🔍 Real-World Scenarios

| Scenario                     | Behavior                            | Fix                                        |
| ---------------------------- | ----------------------------------- | ------------------------------------------ |
| Pod cannot reach Service IP  | Missing or corrupted iptables rules | Restart kube-proxy DaemonSet               |
| Uneven load balancing        | Stale endpoints                     | Recreate Service or restart kube-proxy     |
| Node-to-node traffic failing | CNI or VPC route issue              | Check VPC CNI and node routing tables      |
| External access fails        | Incorrect Service type              | Check Service type (LoadBalancer/NodePort) |

---

## 💥 Common Issues

| Issue                       | Root Cause                       | Fix                         |
| --------------------------- | -------------------------------- | --------------------------- |
| `Connection refused`        | Kube-Proxy down or misconfigured | Restart kube-proxy Pod      |
| Pods unreachable            | Stale Service endpoints          | Delete and recreate Service |
| Slow networking             | iptables mode in large clusters  | Switch to IPVS or eBPF      |
| Uneven traffic distribution | Sticky sessions or DNS caching   | Review Service settings     |

---

## 🛠 Troubleshooting Commands

```bash
# Check Kube-Proxy Pods
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide

# View Kube-Proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Verify Service endpoints
kubectl get endpoints <service-name>

# Check Service IPs
kubectl get svc -o wide
```

---

## ✅ Best Practices

* Use **IPVS mode** for large or high-traffic clusters.
* Keep **Kube-Proxy** version consistent with the cluster version.
* Enable **CloudWatch logging** for kube-proxy (via EKS add-on).
* Avoid manual modification of iptables rules.
* Restart **Kube-Proxy** after CNI updates.
* Monitor network latency metrics (e.g., `kube-proxy sync_proxy_rules_duration_seconds`).

---

## 🧠 Summary

| Concept         | Description                                                 |
| --------------- | ----------------------------------------------------------- |
| **What**        | Manages Service networking and routing on each node.        |
| **Why**         | Ensures Pods and Services can communicate seamlessly.       |
| **How**         | Uses iptables/IPVS/eBPF rules to route and balance traffic. |
| **EKS Context** | Runs as DaemonSet, managed by AWS as an add-on.             |
| **Example**     | Routes traffic for Service 10.96.0.15:80 to multiple Pods.  |

> 🧩 **In short:** The **Kube-Proxy** acts as Kubernetes’ network router — dynamically managing rules and load balancing traffic so Pods and Services communicate reliably across the cluster.

---
---

# 🧩 Container Runtime in Kubernetes

## 🧾 Concept / What

A **Container Runtime** is the **software component responsible for running containers** on each worker node in a Kubernetes cluster. It executes the actual container processes based on instructions received from the **Kubelet** via the **Container Runtime Interface (CRI)**.

> 🧩 In short: The **Kubelet** tells the node *what* to run; the **Container Runtime** actually runs it.

---

## 🎯 Why / Purpose

| Purpose                      | Description                                                     |
| ---------------------------- | --------------------------------------------------------------- |
| **Container Execution**      | Runs and manages containers inside Pods.                        |
| **Image Management**         | Pulls container images from registries (ECR, Docker Hub, etc.). |
| **Resource Isolation**       | Uses Linux namespaces and cgroups to separate resources.        |
| **Pod Lifecycle Support**    | Creates, monitors, and terminates containers as per Pod specs.  |
| **Integration with Kubelet** | Communicates using the standardized CRI API.                    |

---

## ⚙️ How It Works (Step-by-Step)

1️⃣ **Pod Scheduled to Node**
Scheduler selects a node for the Pod.

2️⃣ **Kubelet Fetches PodSpec**
Kubelet receives instructions from the API Server.

3️⃣ **Kubelet Invokes Runtime via CRI**
The Kubelet uses the **Container Runtime Interface (CRI)** to tell the runtime:
“Create a Pod sandbox and start containers with these images.”

4️⃣ **Runtime Pulls Images**
Downloads required container images from the configured registry.

5️⃣ **Runtime Creates and Runs Containers**
Sets up:

* Namespaces (network, PID, mount)
* Cgroups (resource limits)
* Volumes and environment variables

6️⃣ **Kubelet Monitors Health**
Polls the runtime for container status and reports it to the API Server.

---

## 🧩 Container Runtime Interface (CRI)

A **gRPC-based API** that defines how the Kubelet communicates with container runtimes.

### 🔹 CRI Architecture

```
[Kubelet]
   │
   ▼
[Container Runtime Interface (CRI)]
   │
   ├── [Container Runtime] → containerd / CRI-O / Docker-shim
   └── [Image Service] → Pulls and manages images
```

---

## ⚙️ Popular Container Runtimes

| Runtime                 | Description                                          | Notes                     |
| ----------------------- | ---------------------------------------------------- | ------------------------- |
| **containerd**          | Default runtime; lightweight, CNCF project by Docker | Default in EKS            |
| **CRI-O**               | Red Hat/OpenShift optimized runtime                  | Enterprise-grade          |
| **Docker (dockershim)** | Original runtime, deprecated since v1.24             | Replaced by containerd    |
| **gVisor**              | Sandbox runtime for enhanced isolation               | Security-focused          |
| **Kata Containers**     | Runs containers in lightweight VMs                   | Used for secure workloads |

---

## 🧱 containerd: Default Runtime in EKS

**containerd** is now the **default Kubernetes runtime** and is preinstalled in EKS-Optimized AMIs.

### 🔹 Lifecycle Flow

1. Kubelet → CRI → containerd
2. containerd → runc → container creation
3. containerd monitors and reports container states

### 🔹 Verify Runtime

```bash
kubectl get node <node-name> -o jsonpath='{.status.nodeInfo.containerRuntimeVersion}'
```

Output:

```
containerd://1.7.5
```

---

## ☁️ Container Runtime in Amazon EKS

| Node Type              | Runtime              | Managed By             | Description                      |
| ---------------------- | -------------------- | ---------------------- | -------------------------------- |
| **Managed Node Group** | containerd           | AWS                    | Installed via EKS-Optimized AMI  |
| **Self-Managed Node**  | containerd           | You (bootstrap script) | Pre-included in AMI              |
| **Fargate**            | Internal AWS runtime | AWS                    | Invisible to user, fully managed |

### 🔹 EKS Bootstrap Example

```bash
/etc/eks/bootstrap.sh <cluster-name>
```

Automatically starts containerd and connects to the cluster.

---

## 🧩 containerd vs Docker

| Feature               | Docker                           | containerd                     |
| --------------------- | -------------------------------- | ------------------------------ |
| **Scope**             | Full platform (build, ship, run) | Core runtime only              |
| **Overhead**          | Higher                           | Lightweight                    |
| **Integration**       | Needed dockershim (deprecated)   | Native CRI support             |
| **Performance**       | Slightly slower startup          | Faster and leaner              |
| **K8s Compatibility** | Deprecated since v1.24           | Default in all modern clusters |

> In short: Docker included containerd internally — Kubernetes now talks directly to **containerd**, skipping Docker.

---

## 💥 Common Issues

| Issue                            | Root Cause                       | Fix                                   |
| -------------------------------- | -------------------------------- | ------------------------------------- |
| Pod stuck in `ContainerCreating` | Image pull failure               | Check registry and credentials        |
| Pod in CrashLoopBackOff          | App error inside container       | Review container logs                 |
| Image not found                  | Wrong image name or tag          | Verify image and registry permissions |
| Node `NotReady`                  | containerd crash or socket issue | Restart containerd service            |
| High disk usage                  | Unused images or logs            | Clean using `ctr images prune`        |

---

## 🛠 Troubleshooting Commands

```bash
# Check runtime information
kubectl get node -o wide
kubectl describe node <node-name>

# On EC2 node (EKS)
systemctl status containerd
journalctl -u containerd -f

# Check containers and images
ctr containers list
ctr images list

# Clean old images
ctr images prune
```

---

## ✅ Best Practices

* Use **EKS-Optimized AMIs** with containerd preinstalled.
* Regularly **prune unused images** to free up space.
* Use **ECR private repos** with IAM roles for secure pulls.
* Set **resource limits** to prevent node pressure.
* Enable **read-only root filesystems** for security.
* Monitor containerd metrics via **CloudWatch** or **Prometheus**.

---

## 🧠 Summary

| Concept         | Description                                                          |
| --------------- | -------------------------------------------------------------------- |
| **What**        | Software responsible for running containers on nodes.                |
| **Why**         | Executes and manages containers based on Kubelet instructions.       |
| **How**         | Pulls images, creates containers, applies isolation, reports status. |
| **EKS Context** | Uses containerd by default in all managed/self-managed nodes.        |
| **Example**     | Kubelet → CRI → containerd → runc → running container.               |

> 🧩 **In short:** The **Container Runtime** is the engine that actually runs your Pods. In EKS, **containerd** is the default runtime — fast, lightweight, and fully integrated with Kubelet via CRI.

---
---

# 🧩 kubectl & kubeconfig Basics

## 🧾 Concept / What

**kubectl** is the command-line tool used to interact with the Kubernetes API Server. It allows you to deploy applications, inspect and manage cluster resources, and view logs.

**kubeconfig** is the configuration file that stores information about clusters, users, and contexts. It tells kubectl how to connect and authenticate to a specific cluster.

> 🧩 kubectl = CLI tool
> kubeconfig = Authentication and cluster connection info

---

## 🎯 Why / Purpose

| Component      | Purpose                                         | Example            |
| -------------- | ----------------------------------------------- | ------------------ |
| **kubectl**    | Executes commands to manage Kubernetes clusters | `kubectl get pods` |
| **kubeconfig** | Stores credentials and cluster endpoints        | `~/.kube/config`   |

Together, they enable secure and seamless management of clusters.

---

## ⚙️ How kubectl & kubeconfig Work Together

1️⃣ User runs a command: `kubectl get pods`
2️⃣ kubectl reads the kubeconfig file to find:

* Cluster endpoint (API server URL)
* Authentication method (IAM user/role)
* Context (namespace and cluster)
  3️⃣ kubectl connects to the API Server via HTTPS.
  4️⃣ API Server validates the IAM token and permissions.
  5️⃣ etcd returns the data → kubectl displays the result.

✅ kubectl never stores tokens or credentials permanently. It just sends temporary, signed requests to the API Server.

---

## 🧱 kubeconfig File Example

```yaml
apiVersion: v1
kind: Config

clusters:
- name: eks-prod
  cluster:
    server: https://ABC123XYZ.yl4.ap-south-1.eks.amazonaws.com
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJ...

users:
- name: eks-prod-user
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws
      args:
        - eks
        - get-token
        - --cluster-name
        - eks-prod

contexts:
- name: eks-prod-context
  context:
    cluster: eks-prod
    user: eks-prod-user
    namespace: default

current-context: eks-prod-context
```

---

## 🧠 Sections Explained

| Section             | Description                                         |
| ------------------- | --------------------------------------------------- |
| **clusters**        | Contains API server URL and CA certificate.         |
| **users**           | Specifies how to authenticate (IAM role or user).   |
| **contexts**        | Combines cluster, user, and namespace info.         |
| **current-context** | Defines which cluster/context is active by default. |

---

## ☁️ kubeconfig in Amazon EKS

You don’t manually create kubeconfig files in EKS. You generate or update them using:

```bash
aws eks update-kubeconfig --region ap-south-1 --name eks-prod
```

This command:

* Fetches cluster endpoint and CA certificate.
* Adds entries for cluster, user, and context.
* Uses your current IAM identity for authentication.

You can confirm:

```bash
kubectl get nodes
```

✅ Lists worker nodes from your EKS cluster.

---

## 🧩 Adding Multiple EKS Clusters to Same kubeconfig

To manage multiple clusters (like dev, qa, prod), run:

```bash
aws eks update-kubeconfig --region ap-south-1 --name eks-dev
aws eks update-kubeconfig --region ap-south-1 --name eks-qa
aws eks update-kubeconfig --region ap-south-1 --name eks-prod
```

Each command automatically merges new cluster data into the same `~/.kube/config` file — no manual edits needed.

To view and switch contexts:

```bash
kubectl config get-contexts
kubectl config use-context eks-prod-context
```

---

## 🧩 Using IAM Roles for kubeconfig Access

If you want kubectl to use a specific IAM role:

```bash
aws eks update-kubeconfig --region ap-south-1 --name eks-prod --role-arn arn:aws:iam::123456789012:role/EKS_AdminRole
```

✅ This ensures kubectl always assumes that IAM role before connecting to EKS.

---

## 🧩 Automatic IAM Integration

You don’t specify IAM credentials manually in kubeconfig. The AWS CLI and IAM Authenticator handle it dynamically.

Each time you run a kubectl command, kubeconfig runs this:

```bash
aws eks get-token --cluster-name eks-prod
```

That command:

* Uses your **current IAM user/role** credentials.
* Generates a **temporary token (valid 15 minutes)**.
* Sends that token to EKS API Server for authentication.

If you’re logged in via IAM User, EC2 Role, STS-assumed role, or SSO — the same identity is automatically used.

---

## 🕒 EKS Authentication Token Lifetime and Auto-Renewal

* The authentication token generated by `aws eks get-token` is valid for **15 minutes** only.
* **kubectl automatically refreshes the token** for each command you run.
* You never need to manually re-fetch it.

You only need to re-authenticate if:

* Your AWS CLI session expires.
* Your IAM role or SSO token times out.

Then simply run `aws sso login`, `aws sts assume-role`, or `aws eks update-kubeconfig` again.

✅ kubectl fetches a new token automatically every time you execute a command.

---

## 🧰 Common Commands

```bash
kubectl get pods
kubectl describe node <node-name>
kubectl logs <pod>
kubectl config get-contexts
kubectl config use-context eks-prod-context
kubectl cluster-info
```

---

## ✅ Best Practices

* Always generate kubeconfig via `aws eks update-kubeconfig`.
* Use `--role-arn` for specific IAM roles.
* Keep kubeconfig files private and secure.
* Use separate contexts for dev, qa, and prod.
* Avoid manual edits to kubeconfig.
* Monitor AWS IAM session validity.

---

## 🧠 Summary

| Concept             | Description                                           |
| ------------------- | ----------------------------------------------------- |
| **kubectl**         | CLI tool for managing Kubernetes clusters.            |
| **kubeconfig**      | Stores cluster connection and authentication details. |
| **IAM Integration** | Automatically uses your current IAM credentials.      |
| **Token Lifetime**  | 15 minutes; auto-refreshed for each command.          |
| **Best Practice**   | Generate via AWS CLI; no manual editing needed.       |

> 🧩 **In short:**
>
> * kubeconfig dynamically uses your active IAM user or role for EKS authentication.
> * Tokens are short-lived (15 minutes) but automatically renewed per command.
> * You never have to manually fetch tokens — kubectl handles that seamlessly.

---
---

# 🧩 Declarative vs Imperative Management in Kubernetes

## 🧾 Concept / What

There are two primary ways to manage resources in Kubernetes:

| Approach        | Definition                                                                                       |
| --------------- | ------------------------------------------------------------------------------------------------ |
| **Imperative**  | You tell Kubernetes **exactly what action** to take right now.                                   |
| **Declarative** | You tell Kubernetes **what final state you want**, and Kubernetes figures out how to achieve it. |

---

## 🎯 Why / Purpose

| Situation                  | Why It Matters                                                                                     |
| -------------------------- | -------------------------------------------------------------------------------------------------- |
| **Small or testing setup** | Imperative commands are fast and convenient.                                                       |
| **Production or CI/CD**    | Declarative manifests (YAML files) are preferred for consistency, version control, and automation. |

Declarative management allows Kubernetes to automatically maintain the desired state and self-heal when changes or failures occur.

---

## ⚙️ How It Works

### 🧩 Imperative Example

Directly run commands to perform operations:

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=LoadBalancer
kubectl scale deployment nginx --replicas=5
kubectl delete pod nginx-abcde
```

* Executes changes immediately.
* No record of desired configuration.
* Good for testing or one-time changes.

✅ **Advantages:** Quick and simple for small-scale testing.

❌ **Disadvantages:** Not version-controlled, not repeatable, and lacks auditing.

---

### 🧩 Declarative Example

You define the **desired state** in a YAML manifest and apply it using kubectl:

#### Example: `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Apply the YAML manifest:

```bash
kubectl apply -f deployment.yaml
```

Now Kubernetes:

* Saves the desired configuration in etcd.
* Continuously ensures the actual cluster state matches it.
* Automatically recreates or fixes drifted resources.

✅ **Advantages:** Repeatable, version-controlled, self-healing.

❌ **Disadvantages:** Requires maintaining YAML manifests.

---

## 📂 What `-f` Flag Means in kubectl

`-f` or `--filename` flag tells kubectl to use a **YAML manifest file** as input.

### Examples

```bash
kubectl apply -f deployment.yaml    # Apply a single file
kubectl apply -f ./manifests/       # Apply all YAML files in a folder
kubectl delete -f deployment.yaml   # Delete a resource defined in YAML
kubectl apply -f file.yaml --dry-run=client   # Validate syntax only
```

✅ The `-f` flag connects kubectl with your **manifest file** — the declarative definition of your resource.

---

## 🧠 Real-World Use Case Comparison

| Task       | Imperative (Quick)                              | Declarative (Production)                 |
| ---------- | ----------------------------------------------- | ---------------------------------------- |
| Create Pod | `kubectl run mypod --image=nginx`               | Define Pod YAML and apply it             |
| Deploy App | `kubectl create deployment myapp --image=nginx` | Use GitOps or Helm with YAML definitions |
| Scale Pods | `kubectl scale deployment myapp --replicas=5`   | Update replicas in deployment.yaml       |
| Rollback   | Manual rollback                                 | Re-apply old manifest                    |
| Audit      | No record                                       | Git version control                      |

---

## ⚙️ Kubernetes Declarative Flow

When applying a manifest:
1️⃣ `kubectl apply -f` sends YAML to the API Server.
2️⃣ API Server stores it in **etcd** as the **desired state**.
3️⃣ Controller Manager compares desired vs actual state.
4️⃣ Scheduler and Kubelet reconcile differences automatically.

✅ This is the **Reconciliation Loop** — Kubernetes always aligns actual state with desired state.

---

## 💥 Common Issues

| Issue                    | Cause              | Fix                                                |
| ------------------------ | ------------------ | -------------------------------------------------- |
| Overwriting manual edits | Config drift       | Avoid mixing imperative and declarative management |
| Missing labels/selectors | YAML misconfig     | Validate with `kubectl diff` or `kubectl explain`  |
| YAML errors              | Syntax/indentation | Validate with `kubectl apply --dry-run=client`     |
| Drifted states           | Mixed workflows    | Stick to declarative YAMLs                         |

---

## 🛠 Troubleshooting & Validation

```bash
kubectl diff -f deployment.yaml              # Show differences
kubectl get deployment nginx -o yaml         # View full resource config
kubectl apply --dry-run=client -f file.yaml  # Validate YAML before applying
```

---

## ☁️ Declarative in EKS CI/CD Pipelines

In production EKS clusters, pipelines always use **Declarative YAML**.

Example Jenkins stage:

```groovy
stage('Deploy to EKS') {
  steps {
    sh '''
      aws eks update-kubeconfig --region ap-south-1 --name eks-prod
      kubectl apply -f k8s/deployment.yaml
      kubectl rollout status deployment/my-app
    '''
  }
}
```

✅ Ensures repeatability, consistency, and auto-recovery across environments.

---

## ✅ Best Practices

* Always use **Declarative YAML** for production.
* Store YAMLs in **Git** → enables **GitOps**.
* Use **Helm** or **Kustomize** for reusable YAMLs.
* Avoid mixing imperative and declarative workflows.
* Validate manifests using `kubectl diff` or dry-run before applying.

---

## 🧠 Summary

| Concept             | Declarative                  | Imperative                  |
| ------------------- | ---------------------------- | --------------------------- |
| **Definition**      | Define desired state         | Execute direct action       |
| **Persistence**     | Stored in etcd               | No persistence              |
| **Auditing**        | Version-controlled           | None                        |
| **Recovery**        | Auto self-healing            | Manual                      |
| **Usage**           | Production, CI/CD            | Local testing               |
| **Command Example** | `kubectl apply -f file.yaml` | `kubectl create deployment` |

> 🧩 **In short:**
>
> * **Imperative:** “Do this now.”
> * **Declarative:** “Keep it this way.”
>
> Declarative management is the Kubernetes standard — stable, traceable, and ideal for production and GitOps workflows.

---
---

# 🧩 Difference Between EKS, ECS, and Fargate

## 🧾 What Each Service Is

| Service     | Full Form                  | Description                                                                               |
| ----------- | -------------------------- | ----------------------------------------------------------------------------------------- |
| **EKS**     | Elastic Kubernetes Service | AWS-managed **Kubernetes** control plane — run containerized workloads using Kubernetes.  |
| **ECS**     | Elastic Container Service  | AWS’s **proprietary container orchestration** platform — simpler and fully AWS-native.    |
| **Fargate** | AWS Fargate                | **Serverless compute engine** for containers — runs workloads *without managing servers*. |

---

## 🎯 Why They Exist

| Goal                             | EKS                           | ECS                             | Fargate                     |
| -------------------------------- | ----------------------------- | ------------------------------- | --------------------------- |
| **Kubernetes compatibility**     | ✅ Full Kubernetes API support | ❌ Proprietary AWS orchestration | 🔸 Acts as compute for both |
| **AWS-native simplicity**        | ⚙️ Moderate (K8s complexity)  | ✅ Simple setup and integration  | ✅ Simplest (serverless)     |
| **No infrastructure management** | ❌ (if EC2) / ✅ (if Fargate)   | ❌ (if EC2) / ✅ (if Fargate)     | ✅ Fully managed             |

---

## ⚙️ How They Work (Control Plane vs Data Plane)

| Component         | **EKS**                         | **ECS**                       | **Fargate**                 |
| ----------------- | ------------------------------- | ----------------------------- | --------------------------- |
| **Control Plane** | Managed by AWS (K8s Masters)    | Managed by AWS                | Managed by AWS              |
| **Worker Nodes**  | EC2 or Fargate                  | EC2 or Fargate                | No nodes (fully serverless) |
| **Networking**    | Kubernetes CNI (VPC CNI Plugin) | ECS networking (ENI per task) | ECS/EKS ENI-based           |
| **Scheduling**    | Kubernetes Scheduler            | ECS Service Scheduler         | AWS internal scheduler      |

---

## ☁️ Real-World Scenarios

| Use Case                            | Best Option            | Why                                |
| ----------------------------------- | ---------------------- | ---------------------------------- |
| Existing Kubernetes ecosystem       | **EKS**                | Compatible with Helm, ArgoCD, etc. |
| AWS-only, simpler container setup   | **ECS**                | No Kubernetes overhead             |
| Run containers without managing EC2 | **Fargate**            | Fully serverless execution         |
| Cost optimization                   | **EKS/ECS on Fargate** | Pay only for runtime               |
| Multi-cloud or hybrid               | **EKS**                | Kubernetes portability             |

---

## 🧩 Fargate’s Role

Fargate isn’t an alternative *to* ECS or EKS — it’s a **compute option** *for both*.

| Mode               | Description                    |
| ------------------ | ------------------------------ |
| **EKS on EC2**     | You manage worker nodes        |
| **EKS on Fargate** | AWS runs pods directly         |
| **ECS on EC2**     | You manage container instances |
| **ECS on Fargate** | AWS runs tasks directly        |

✅ Think of **Fargate** as: *“Run my containers — I don’t care about servers.”*

---

## 💰 Pricing Overview

| Component         | EKS                      | ECS                    | Fargate               |
| ----------------- | ------------------------ | ---------------------- | --------------------- |
| **Control Plane** | ~$0.10/hr (~$73/mo)      | Free                   | Free                  |
| **Compute**       | Pay for EC2 or Fargate   | Pay for EC2 or Fargate | Pay per vCPU + memory |
| **Networking**    | Standard AWS VPC charges | Same                   | Same                  |

---

## 🧩 EKS Control Plane Management

### ⚙️ What AWS Manages

EKS’s **control plane** includes:

* `kube-apiserver`
* `etcd`
* `kube-controller-manager`
* `kube-scheduler`

All of these components are **fully managed by AWS** — you cannot access or modify them.

AWS handles:

* Multi-AZ high availability
* Auto-scaling of API servers
* Backups and encryption of `etcd`
* Security patching and control plane upgrades
* Monitoring and failover

You only manage **worker nodes** and **Kubernetes resources**.

---

### 🧰 What You Can Manage

| Area                    | Responsibility                              |
| ----------------------- | ------------------------------------------- |
| Worker Nodes            | EC2 or Fargate management                   |
| Add-ons                 | CNI, CoreDNS, kube-proxy                    |
| Application Deployments | Pods, Services, Ingress, etc.               |
| IAM Integration         | Users, Roles, IRSA setup                    |
| Monitoring              | CloudWatch, Prometheus, Grafana integration |
| Networking              | VPC, Subnets, and Security Groups           |

---

### 🧩 AWS vs Your Responsibilities

| Responsibility             | AWS                  | You                            |
| -------------------------- | -------------------- | ------------------------------ |
| **API Server**             | ✅ Managed            | ❌ Not accessible               |
| **etcd Database**          | ✅ Managed, backed up | ❌ Not visible                  |
| **Control Plane Upgrades** | ✅ AWS executes       | ⚙️ You trigger via CLI/Console |
| **Worker Node Lifecycle**  | ❌                    | ✅                              |
| **Kubernetes Workloads**   | ❌                    | ✅                              |
| **RBAC & IAM Integration** | Shared               | ✅ You configure                |

---

### 🔄 Control Plane Upgrade Example

```bash
aws eks update-cluster-version --name eks-prod --kubernetes-version 1.31
```

* You **initiate** the upgrade.
* AWS **handles** the actual control plane upgrade.

After that, you must upgrade your **worker nodes** manually.

---

### 💡 If You Want Full Control Over Control Plane

You would have to deploy Kubernetes manually using `kops` or `kubeadm` on EC2 — that’s a **self-managed Kubernetes cluster**, not EKS.

| Cluster Type                    | Control Plane Managed By |
| ------------------------------- | ------------------------ |
| **EKS**                         | AWS                      |
| **Self-Managed (kubeadm/kops)** | You                      |

---

## ✅ Summary

| Feature             | EKS                  | ECS                     | Fargate                      |
| ------------------- | -------------------- | ----------------------- | ---------------------------- |
| **Type**            | Managed Kubernetes   | AWS-native orchestrator | Serverless container compute |
| **Control Plane**   | Managed by AWS       | Managed by AWS          | Managed by AWS               |
| **Node Management** | You (EC2/Fargate)    | You (EC2/Fargate)       | Fully managed                |
| **Ease of Use**     | Moderate             | Easy                    | Very Easy                    |
| **Portability**     | Multi-cloud (K8s)    | AWS-only                | AWS-only                     |
| **Ideal Use**       | Kubernetes workloads | AWS-native apps         | Serverless containers        |

> 🧩 **In short:**
>
> * **EKS** → Kubernetes on AWS (AWS manages control plane).
> * **ECS** → AWS’s own orchestrator (simple, AWS-only).
> * **Fargate** → Serverless compute for running ECS/EKS tasks or pods.
>
> You can’t manage the **EKS control plane** — AWS owns it. You just interact with it and manage worker nodes and workloads.

---
---
---

