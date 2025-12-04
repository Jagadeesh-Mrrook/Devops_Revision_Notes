# Pod Issues: CrashLoopBackOff & ImagePullBackOff (Detailed Explanation)

## **Concept / What**

### **CrashLoopBackOff**

A Pod repeatedly crashes after starting. Kubernetes restarts it several times with increasing back-off delay.

### **ImagePullBackOff**

Kubernetes cannot pull the required container image from the registry, so it backs off from retrying.

---

## **Why / Purpose / Use Case in Real-World**

### CrashLoopBackOff – Why it Happens

* Application runtime failure
* Missing environment variables
* Wrong command/entrypoint
* DB connection issues
* File or directory not found
* Permission issues
* Liveness probe killing container
* OOM errors causing termination

### ImagePullBackOff – Why it Happens

* Wrong image name or tag
* Image not available in registry
* Missing registry authentication
* IAM role issues in EKS
* Private registry without pull-secret
* Worker node network problems

---

## **How it Works / Steps / Syntax**

### **Identify CrashLoopBackOff**

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

Check:

* Exit codes
* Termination reason
* Crash patterns

Check logs:

```bash
kubectl logs <pod-name> -c <container-name>
```

### **Identify ImagePullBackOff**

```bash
kubectl describe pod <pod-name>
```

Look for:

* `ErrImagePull`
* `Failed to pull image`
* `pull access denied`

### **Manifest Examples**

#### Example 1: CrashLoop due to missing ENV

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: myapp-container
      image: myrepo/myapp:1.0
      env:
        - name: DB_HOST
          value: ""  # Causes crash
```

#### Example 2: ImagePullBackOff due to wrong tag

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: wrongimage
spec:
  containers:
    - name: app
      image: myregistry/myapp:latest123  # Wrong tag
```

#### Example 3: Using imagePullSecrets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: myprivate/app:1.0
```

---

## **Common Issues / Errors**

### CrashLoopBackOff

* Application startup failure
* Missing ConfigMap/Secret values
* Incorrect file paths
* Port already in use
* Permission denied
* Liveness probe restarts
* Insufficient memory (OOMKilled)

### ImagePullBackOff

* Wrong image tag or repo
* Missing authentication
* Expired ECR token
* Private registry without secret
* DNS failures
* No route to registry (NAT/Firewall)

---

## **Troubleshooting / Fixes**

### Fixing CrashLoopBackOff

* Check logs and fix application errors
* Provide correct ENV variables
* Fix command/entrypoint
* Correct ConfigMap/Secret reference
* Increase resource limits
* Adjust liveness/readiness probes
* Fix permissions using chmod/chown

### Fixing ImagePullBackOff

* Correct image name and tag
* Add/create imagePullSecret
* Fix IAM role on EKS worker nodes
* Verify registry connectivity
* Ensure NAT/Gateway/DNS working

---

## **Best Practices / Tips**

* Avoid using `latest` tag
* Validate images locally before deployment
* Use proper liveness/readiness probes
* Use Secrets for sensitive ENV values
* Ensure worker nodes have correct IAM roles (EKS)
* Use `imagePullPolicy: IfNotPresent` to reduce unnecessary pulls

---
---

# Pending Pods (Resource & Scheduling Issues) – Detailed Explanation

## **Concept / What**

A Pod remains in **Pending** state when Kubernetes accepts it but **cannot schedule it on any node**. The Pod is created but not assigned to a node due to resource shortages, scheduling rules, storage problems, or node readiness issues.

---

## **Why / Purpose / Use Case in Real-World**

### **Why Pods Become Pending**

* Nodes lack enough **CPU/Memory** to meet Pod requests
* Pod has constraints (nodeSelector, affinity, tolerations) that no node satisfies
* Nodes are tainted and Pod cannot tolerate them
* PVC cannot bind to a PV (storage unavailability)
* Nodes are **NotReady**, cordoned, or CNI is broken
* Namespace **ResourceQuota** is exceeded
* Node groups scaled to zero (EKS autoscaling)

Pending Pods help identify scheduling or infrastructure issues before containers start running.

---

## **How It Works / Steps / Syntax**

### **Check Pod Status**

```bash
kubectl get pods
```

### **Describe Pod for Scheduling Errors**

```bash
kubectl describe pod <pod-name>
```

Look at **Events**:

* `insufficient cpu`
* `insufficient memory`
* `node(s) had taints`
* `persistentvolumeclaim not bound`
* `pod didn't match node selector`

### **Manifest Examples**

#### **Example 1 — Over-requesting resources**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: heavy-app
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "4"        # Node doesn't have 4 CPU available
          memory: "8Gi"  # Node doesn't have enough memory
```

#### **Example 2 — NodeSelector mismatch**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: region-specific
spec:
  nodeSelector:
    region: us-east-1a  # No node with this label exists
  containers:
    - name: app
      image: nginx
```

#### **Example 3 — Taints & tolerations**

Node taint:

```bash
kubectl taint nodes worker1 dedicated=app:NoSchedule
```

Pod with toleration:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "app"
    effect: "NoSchedule"
```

#### **Example 4 — PVC not bound**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  volumes:
    - name: myvol
      persistentVolumeClaim:
        claimName: myclaim
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: myvol
          mountPath: "/data"
```

If `myclaim` is not **Bound**, Pod stays Pending.

---

## **Common Issues / Errors**

### **Resource Issues**

* CPU/memory shortage
* Over-requesting resources
* Highly utilized nodes
* Node group scaled down

### **Scheduling Constraints Issues**

* nodeSelector mismatch
* Missing tolerations
* Anti-affinity blocking scheduling

### **Node Issues**

* Node `NotReady`
* CNI malfunctioning
* Node cordoned/drained

### **Storage Issues**

* PVC stuck in Pending
* No matching PV
* Wrong StorageClass
* EBS volume stuck (EKS)

### **Quota Issues**

* Namespace ResourceQuota exceeded

---

## **Troubleshooting / Fixes**

### **Fix Resource Problems**

* Lower CPU/Memory requests
* Increase node size or count
* Enable/verify Cluster Autoscaler

### **Fix Scheduling Constraint Problems**

* Correct nodeSelector labels
* Adjust affinity/anti-affinity
* Add required tolerations
* Remove problematic node taints

### **Fix Storage Problems**

* Ensure proper StorageClass
* Create PV or fix dynamic provisioning
* Ensure PVC/PV zones match (EKS)

### **Fix Node Problems**

* Restart node/CNI
* Mark node Ready (uncordon)
* Fix IAM roles for EKS worker nodes

### **Fix Quota Issues**

```bash
kubectl describe quota
```

Increase limits or delete unused Pods.

---

## **Best Practices / Tips**

* Avoid over-requesting resources
* Use Cluster Autoscaler for EKS
* Validate all Pod constraints (nodeSelector, affinity)
* Keep node taints minimal unless needed
* Regularly monitor PVC/PV health
* Plan stateful workloads considering zone restrictions
* Ensure proper CNI configuration for new nodes

---
---

# DNS & Service Communication Issues – Detailed Explanation

## **Concept / What**

DNS and service communication issues occur when Pods cannot resolve service names or cannot reach other services inside the Kubernetes cluster. Kubernetes uses **CoreDNS** for service discovery, and any failure in DNS, networking, or service configuration can cause communication breakdowns.

---

## **Why / Purpose / Use Case in Real-World**

DNS ensures that services inside Kubernetes can communicate using service names instead of IPs. When DNS or networking breaks, Pods experience:

* DNS lookup failures
* Service unreachable errors
* Timeouts
* Connection refused

### **Why These Issues Happen**

* CoreDNS is failing or misconfigured
* Service selectors do not match Pod labels
* No endpoints created for a service
* CNI plugin failure or incorrect networking setup
* Wrong DNS names used in applications
* PVC problems or unready nodes affecting DNS pods
* IP exhaustion in VPC (EKS)

---

## **How It Works / Steps / Syntax**

### **Service Discovery Flow**

1. Pod sends DNS query to CoreDNS
2. CoreDNS resolves `<service>.<namespace>.svc.cluster.local`
3. DNS returns the Service ClusterIP
4. kube-proxy routes traffic from Service → Endpoints → Target Pod

### **Useful Commands**

```bash
kubectl get svc
kubectl get endpoints <service>
kubectl describe svc <service>
```

### **Testing DNS**

```bash
kubectl exec -it <pod> -- nslookup myservice
kubectl exec -it <pod> -- curl myservice:80
```

---

## **Manifest Examples**

### **Example 1 – Service with Wrong Selector (Common Issue)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: fronted  # Typo: should be 'frontend'
  ports:
    - port: 80
      targetPort: 80
```

**Effect:** No endpoints created, DNS resolves but traffic has nowhere to go.

### **Example 2 – CoreDNS ConfigMap (Correct Form)**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
    }
```

CoreDNS failures usually appear as CrashLoopBackOff or DNS query timeouts.

---

## **Common Issues / Errors**

* `no such host`
* `lookup <service> on <dns-ip>: server misbehaving`
* `i/o timeout`
* `connection refused`
* DNS resolution extremely slow
* Service has **zero endpoints**

### **Common Causes**

* Selector mismatch between Service and Pods
* CoreDNS pod crash or incorrect ConfigMap
* CNI network failure (Calico, AWS VPC CNI)
* Nodes unable to reach DNS resolver
* Wrong service FQDN used by the application
* ENI or IP exhaustion in EKS nodes

---

## **Troubleshooting / Fixes**

### **Check Endpoints**

```bash
kubectl get endpoints <service>
```

If empty → selector mismatch.

### **Verify Pod Labels**

```bash
kubectl get pods --show-labels
```

### **Test DNS Resolution**

```bash
kubectl exec -it <pod> -- nslookup myservice.default.svc.cluster.local
```

### **Check CoreDNS Health**

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### **Fix CNI Issues**

* Restart CNI daemonset
* Recycle problematic nodes
* Ensure VPC subnets have free IPs

### **Validate Service & Ports**

```bash
kubectl describe svc <service>
```

Ensure:

* Correct targetPort
* Correct selector labels

---

## **Best Practices / Tips**

* Always verify Service selectors match Pod labels
* Use fully-qualified DNS names for cross-namespace communication
* Avoid unnecessary modifications to CoreDNS ConfigMap
* Regularly monitor CoreDNS and CNI DaemonSets
* Validate cluster networking after node scaling events
* Use tools like busybox/dnsutils for DNS debugging
* Ensure correct VPC IP allocation in EKS

---
---

# Node NotReady & CNI Issues – Detailed Explanation

## **Concept / What**

### **Node NotReady**

A node enters **NotReady** state when the Kubelet on that node stops reporting healthy status to the Kubernetes API server. This means the control plane cannot reliably schedule Pods or communicate with the node.

### **CNI Issues**

CNI (Container Network Interface) is responsible for assigning IPs to Pods and configuring networking. CNI issues cause Pods to lose networking, fail to get IPs, or fail to communicate across nodes.

---

## **Why / Purpose / Use Case in Real-World**

### **Why Node NotReady Occurs**

* Kubelet failure or crash
* DiskPressure or MemoryPressure
* Node network not reachable
* Time synchronization issues (NTP drift)
* Certificate issues (kubelet cert expired)
* CNI plugin failure causes node networking to break

### **Why CNI Issues Occur**

* No available Pod IPs (subnet exhaustion)
* CNI DaemonSet crashed (aws-node, calico-node, etc.)
* iptables/IPVS rules corrupted
* Missing kernel modules
* Misconfigured NetworkPolicies
* Incorrect IAM role for VPC CNI in EKS

---

## **How It Works / Steps / Syntax**

### **Identify Node NotReady**

```bash
kubectl get nodes
kubectl describe node <node-name>
```

Look for:

* `Kubelet stopped posting node status`
* `NetworkUnavailable`
* `DiskPressure`
* `MemoryPressure`
* `PIDPressure`

### **Check Kubelet Logs**

```bash
journalctl -u kubelet -f
```

### **Identify CNI Issues**

Check CNI pods:

```bash
kubectl get pods -n kube-system | grep cni
```

Check Pod IP allocation:

```bash
kubectl get pods -o wide
```

If Pod IP is `<none>` → CNI failure.

Check network interfaces:

```bash
ip addr
```

EKS: Check ENIs

```bash
aws ec2 describe-network-interfaces
```

---

## **Manifest Context / Real Scenarios**

### **Scenario 1 – Disk Full → Node NotReady**

Symptoms:

```
NodeHasDiskPressure
```

Fix: Clean `/var/log`, expand disk size.

### **Scenario 2 – Pod Fails to Get IP (CNI Broken)**

Pod shows:

```
IP: <none>
```

CNI logs:

```
failed to assign IP address: no available IPs
```

Fix: Increase subnet CIDR, add more nodes, reduce max-pods per node.

### **Scenario 3 – Missing IAM Role (EKS)**

AWS VPC CNI requires:

* AmazonEKSWorkerNodePolicy
* AmazonEKS_CNI_Policy
* AmazonEC2ContainerRegistryReadOnly

Missing → CNI fails → node becomes unstable.

### **Scenario 4 – CNI DaemonSet CrashLoopBackOff**

Check:

```bash
kubectl get pods -n kube-system | grep aws-node
```

Fix: Restart CNI pods or reapply CNI manifest.

---

## **Common Issues / Errors**

### **Node NotReady**

* Kubelet stopped
* Disk full
* Memory exhaustion
* Node unreachable
* Certificate expired
* CNI plugin crash

### **CNI Issues**

* Pod stuck with IP `<none>`
* Cross-node communication failing
* DNS failures
* CrashLoopBackOff CNI pods
* ENI allocation errors (EKS)
* NetworkPolicy blocking traffic

---

## **Troubleshooting / Fixes**

### **Fix Node NotReady**

1. Restart kubelet:

```bash
systemctl restart kubelet
```

2. Fix disk pressure:

```bash
du -sh /var/log/*
rm -rf <large-log-files>
```

3. Fix memory pressure: Increase node size, reduce pod limits.
4. Fix network routing: Restart CNI, check route tables.
5. Fix time sync:

```bash
timedatectl status
systemctl restart chronyd
```

6. Fix certificate issues (renew kubelet certs).

### **Fix CNI Issues**

* Restart CNI plugin pods:

```bash
kubectl -n kube-system delete pod -l k8s-app=aws-node
```

* Add more subnet IPs / additional ENIs
* Scale worker nodes
* Reapply CNI manifests
* Fix network policies blocking traffic
* Recycle problematic nodes

---

## **Best Practices / Tips**

* Monitor node pressure (disk/memory)
* Keep CNI plugin updated
* Allocate large enough VPC subnets (EKS)
* Avoid modifying iptables manually
* Use Managed Node Groups for node auto-repair
* Use Node Problem Detector for monitoring
* Regularly validate ENI allocation and IP usage

---
---

# Event Logs & Audit Troubleshooting – Detailed Explanation

## **Concept / What**

### **Event Logs**

Kubernetes Events are real‑time messages generated by the cluster components (Kubelet, Scheduler, Controller Manager). They explain **why** something happened in the cluster, such as pod scheduling, failures, image pull issues, PVC problems, and node pressure.

### **Audit Logs**

Audit logs record **who did what and when** in the Kubernetes API server. They track all API requests, user actions, resource changes, RBAC violations, authentication failures, and administrative operations.

Audit logs are **NOT enabled by default** and must be explicitly configured.

---

## **Why / Purpose / Use Case in Real‑World**

### **Why Events Are Important**

* Diagnose pod failures (CrashLoopBackOff, ImagePullBackOff)
* Understand scheduling issues
* Debug storage provisioning problems
* Detect node or CNI issues
* Provide instant visibility into cluster activities

### **Why Audit Logs Are Important**

* Security visibility (who deleted/modified what)
* Compliance (PCI, SOC2, HIPAA)
* Track unauthorized access attempts
* Troubleshoot IAM/RBAC issues
* Detect misbehaving scripts or controllers

### **Why Audit Logs Are NOT Enabled by Default**

* Generate extremely large log volume
* Increase storage cost (especially on EKS CloudWatch)
* Potential cluster performance impact
* Require custom audit policies and retention rules
* Security teams need to choose what to log

---

## **How It Works / Steps / Syntax**

### **Viewing Events**

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl describe pod <pod-name>
```

### **Viewing Container Logs**

```bash
kubectl logs <pod>
kubectl logs <pod> -c <container-name>
```

### **Audit Logs in EKS**

Enable via EKS Console → Logging → enable:

* audit
* api
* scheduler
* controllerManager
* authenticator

Logs appear in CloudWatch:

```
/aws/eks/<cluster-name>/cluster
```

### **Audit Logs in Self‑Managed Kubernetes**

API server requires explicit flags:

```bash
--audit-log-path=/var/log/audit.log
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
```

---

## **Common Issues / Errors**

### **Events Show Issues Like:**

* `FailedScheduling`
* `CrashLoopBackOff`
* `ImagePullBackOff`
* `FailedCreatePodSandBox`
* `NodeHasDiskPressure`
* `NetworkUnavailable`
* `ProvisioningFailed`
* `VolumeAttachmentFailed`

### **Audit Logs Reveal:**

* Who deleted deployments/services
* Unauthorized access attempts
* RBAC forbidden actions
* Automation tools modifying resources
* IAM‑based failures in EKS

---

## **Troubleshooting / Fixes**

### **Using Describe (Most Important Command)**

```bash
kubectl describe <resource> <name>
```

Shows warnings, state transitions, and detailed event history.

### **Using Logs for Applications**

* Check crash reasons
* Debug initContainers
* Investigate probe failures

### **Using Audit Logs for API Issues**

* Track resource deletion/modification
* Debug failed authentication
* Identify RBAC rule misconfigurations

### **In EKS Troubleshooting**

* Ensure control plane logging is enabled
* Search for `delete`, `forbidden`, or `unauthorized` entries
* Forward logs to SIEM (ELK, Splunk, Loki)

---

## **Best Practices / Tips**

* Always check `kubectl describe` first for any issue
* Keep audit logs enabled in production for security
* Use retention policies to manage cost
* Avoid enabling full audit logs in small dev clusters
* Send audit logs to ELK/Loki/Splunk for analysis
* Enable all control plane logs in EKS
* Review events regularly via monitoring systems

---
---

# Common EKS Maintenance Operations – Detailed Explanation

## **Concept / What**

EKS (Elastic Kubernetes Service) is a *managed* Kubernetes service where AWS manages the **control plane**, but DevOps engineers are responsible for maintaining:

* Kubernetes version upgrades (control plane & nodes)
* Worker nodes (node groups)
* Addons (CNI, CoreDNS, kube-proxy)
* Networking (subnets, ENIs, IPs)
* Logging, scaling, storage
* Security patches and cluster hygiene

Maintenance operations ensure the cluster stays secure, up-to-date, stable, and cost-efficient.

---

## **Why / Purpose / Use Case in Real-World**

Even though AWS manages the control plane infrastructure, companies must maintain:

* Kubernetes **versions** (AWS will NOT auto-upgrade them)
* Worker nodes (OS/kernel patches, AMI updates)
* Addons compatible with the Kubernetes version
* Network capacity (subnet IPs, ENIs)
* Logging and audit requirements
* Storage for workloads (EBS/EFS lifecycle management)

Without maintenance:

* Clusters become outdated
* Apps break due to API deprecations
* Security vulnerabilities accumulate
* Networking becomes unstable
* Control plane deprecation deadlines are missed

---

## **How It Works / Steps / Syntax**

# **1. Detailed Explanation: EKS Control Plane Upgrade (Kubernetes Version Upgrade)**

### **✔ AWS manages the control plane infrastructure**

* API Server
* etcd
* Control plane scaling
* Control plane security patches

But when it comes to **Kubernetes version**, AWS does NOT upgrade it automatically. You must decide when to upgrade.

---

### **✔ What happens in a control plane version upgrade?**

1. You choose the new version (ex: 1.27 → 1.28).
2. AWS upgrades the EKS control plane components:

   * kube-apiserver
   * api-aggregation layer
   * admission controllers
   * core controllers
   * etcd compatibility adjustments
3. AWS ensures zero downtime by using multi-AZ HA design.
4. AWS updates control plane API versions and removes deprecated APIs.
5. Cluster API behavior changes to match the new version.
6. After upgrade, AWS expects YOU to:

   * Upgrade node groups
   * Upgrade CNI / CoreDNS / kube-proxy
   * Fix deprecated API usage in your manifests

---

### **✔ Why control plane version upgrade is not automatic**

* Kubernetes releases frequently (every ~4 months).
* Many APIs get deprecated.
* Your workloads may use removed API versions.
* Helm charts may require updates.
* Addons need matching versions.
* Upgrading blindly may break production.

AWS leaves the decision to you.

---

### **✔ What does “EKS supports only 3 versions” mean?**

At any time, AWS supports only the **latest 3 Kubernetes versions**.

Example:
If latest is **1.29**, supported versions are:

* 1.29
* 1.28
* 1.27

Version 1.26 becomes deprecated.

If you stay too long on deprecated versions:

* You lose support
* Security patches stop
* You may be forced to upgrade
* Some features or addons will break

---

### **✔ How to Upgrade the Control Plane**

**Option 1: EKS Console**

1. Go to EKS → your cluster
2. Choose “Upgrade version”
3. Select the new version
4. Confirm upgrade

**Option 2: eksctl**

```bash
eksctl upgrade cluster --name=mycluster --region=ap-south-1
```

**Option 3: AWS CLI**

```bash
aws eks update-cluster-version \
  --name mycluster \
  --region ap-south-1 \
  --kubernetes-version 1.28
```

After control plane is upgraded, immediately upgrade:

* AWS VPC CNI
* CoreDNS
* kube-proxy

---

# **2. Node Group Upgrade (AMI, OS, Kernel)**

Worker nodes require upgrades for:

* Security patches
* Kernel updates
* Bug fixes
* CVE fixes
* New Kubernetes features

Update process:

1. Create a new node group or new version
2. Drain old nodes

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

3. Delete old nodes

---

# **3. Updating EKS Addons (CNI, CoreDNS, kube-proxy)**

These must match the Kubernetes version.

### AWS VPC CNI (aws-node)

Handles Pod IP assignment.
Upgrade:

```
EKS Console → Addons → Update
```

### CoreDNS

Handles cluster DNS.
Upgrade:

```
EKS Console → Addons → Update CoreDNS
```

### kube-proxy

Manages service networking rules.
Upgrade similarly through Addons.

---

# **4. Pod IP & Subnet Management (EKS-Specific)**

Pods consume VPC IPs.

Common issues:

* Subnet IP exhaustion
* ENI limitation

Fixes:

* Add larger subnets
* Add more subnets to node groups
* Enable prefix delegation
* Reduce `maxPods` per node
* Add more worker nodes

---

# **5. Autoscaling Maintenance (Cluster Autoscaler & HPA)**

### Cluster Autoscaler (CA)

Used for automatic node scaling.
Must be updated when Kubernetes version changes.

### Horizontal Pod Autoscaler (HPA)

Tune CPU/memory thresholds.

---

# **6. Logging & Monitoring Maintenance**

Tasks include:

* Enabling audit logs manually (not default)
* Adjusting CloudWatch log retention
* Monitoring control plane logs
* Managing Prometheus/Grafana health

---

# **7. Storage Maintenance**

Common tasks:

* Resize EBS volumes
* Clear orphaned EBS volumes
* Fix stuck PVCs
* Maintain EFS throughput modes

---

# **8. Cleaning Unused Resources (Cost Optimization)**

Remove:

* Unused ALBs
* Old EBS volumes
* Unused ENIs
* Idle node groups
* Old ECR images
* Old CloudWatch log groups

---

## **Common Issues / Errors**

* Node group upgrade failures
* Addon version mismatches
* Control plane upgrade blocked by deprecated APIs
* CNI failures after version bump
* IP exhaustion in subnets
* Autoscaler unable to scale

---

## **Troubleshooting / Fixes**

* Use `kubectl describe` to identify Pod/Node issues
* Check CNI DaemonSet logs in `kube-system`
* Validate ENI/IP allocation
* Verify addon compatibility
* Use PodDisruptionBudgets before draining nodes

---

## **Best Practices / Tips**

* Always upgrade in this order: **control plane → addons → node groups**
* Test upgrades in lower environments first
* Use Managed Node Groups (easier & safer)
* Implement PodDisruptionBudgets for safe draining
* Monitor VPC IP usage (enable prefix delegation)
* Use Blue/Green node group strategy for safe upgrades
* Maintain backup and rollback plans
* Keep all EKS addons updated monthly

---
---

