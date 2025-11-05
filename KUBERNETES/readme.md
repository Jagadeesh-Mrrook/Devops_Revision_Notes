# K8s Concepts List

A comprehensive list of **Kubernetes (EKS-focused)** concepts, organized for theory learning and interview preparation — similar to the Terraform Master Checklist.
This checklist is purely conceptual; YAML manifests will be included later during daily deep dives.

---

## 1. **Kubernetes & EKS Fundamentals**

* [ ] What is Kubernetes & why it’s used
* [ ] Kubernetes architecture (Control Plane & Worker Nodes)
* [ ] EKS architecture and AWS-managed components
* [ ] Core components: API Server, etcd, Controller Manager, Scheduler, Kubelet, Kube Proxy
* [ ] kubectl & kubeconfig basics
* [ ] Declarative vs Imperative management
* [ ] Difference between EKS, ECS, and Fargate

---

## 2. **Pods & ReplicaSets**

* [ ] Pod definition & lifecycle
* [ ] Single vs multi-container Pods
* [ ] initContainers & sidecar containers
* [ ] ReplicaSet purpose and desired state management
* [ ] Labels, selectors, annotations
* [ ] Pod restart policies and termination
* [ ] Debugging Pods (`kubectl logs`, `exec`, `describe`)

---

## 3. **Deployments**

* [ ] Deployment vs ReplicaSet
* [ ] Rolling updates & rollbacks
* [ ] Deployment strategies (Recreate, RollingUpdate)
* [ ] Blue-Green & Canary deployment patterns
* [ ] Deployment revision history
* [ ] Rollout pause, resume, undo

---

## 4. **Services & Networking**

* [ ] Kubernetes networking model
* [ ] ClusterIP, NodePort, LoadBalancer
* [ ] AWS VPC CNI plugin
* [ ] CoreDNS service discovery
* [ ] Headless and ExternalName services
* [ ] Network troubleshooting basics

---

## 5. **Ingress & Ingress Controllers**

* [ ] What is Ingress and why it’s used
* [ ] AWS ALB Ingress Controller
* [ ] Path-based & host-based routing
* [ ] TLS/SSL termination in EKS
* [ ] Multi-service routing
* [ ] Common Ingress errors and troubleshooting

---

## 6. **ConfigMaps & Secrets**

* [ ] ConfigMap creation & usage
* [ ] Secrets (base64 encoding, KMS encryption)
* [ ] Injecting configs and secrets into Pods
* [ ] Managing application configuration securely
* [ ] AWS Secrets Manager & SSM Parameter Store integration

---

## 7. **Storage in EKS**

* [ ] Persistent storage overview
* [ ] Volume types (emptyDir, hostPath, EBS, EFS)
* [ ] Persistent Volume (PV) & Persistent Volume Claim (PVC)
* [ ] StorageClass and dynamic provisioning
* [ ] Volume expansion & reclaim policies
* [ ] AWS EBS CSI & EFS CSI drivers

---

## 8. **Namespaces & Resource Quotas**

* [ ] Namespace creation and purpose
* [ ] ResourceQuota & LimitRange
* [ ] CPU/Memory requests and limits
* [ ] Isolation and multi-team setup
* [ ] Namespace context switching in kubectl

---

## 9. **RBAC & IAM Integration**

* [ ] Kubernetes authentication & authorization
* [ ] Roles, ClusterRoles, RoleBindings, ClusterRoleBindings
* [ ] ServiceAccounts
* [ ] IAM Role for ServiceAccount (IRSA)
* [ ] Least privilege model
* [ ] Access testing (`kubectl auth can-i`)

---

## 10. **Scheduling & Node Management**

* [ ] Scheduler working (filters & scoring)
* [ ] NodeSelector & Affinity/Anti-Affinity
* [ ] Taints and Tolerations
* [ ] Node draining, cordoning, and maintenance
* [ ] Fargate Profiles in EKS

---

## 11. **Autoscaling**

* [ ] Horizontal Pod Autoscaler (HPA)
* [ ] Vertical Pod Autoscaler (VPA)
* [ ] Cluster Autoscaler
* [ ] Metrics Server setup
* [ ] Autoscaling thresholds and stabilization

---

## 12. **Monitoring & Logging**

* [ ] CloudWatch Container Insights
* [ ] Prometheus & Grafana overview
* [ ] Fluent Bit logging agent in EKS
* [ ] Log routing to CloudWatch / OpenSearch
* [ ] Monitoring nodes, pods, and control plane metrics

---

## 13. **Helm**

* [ ] What is Helm and why it’s used
* [ ] Helm chart structure (Chart.yaml, templates, values)
* [ ] Installing and upgrading Helm charts
* [ ] Overriding values.yaml
* [ ] Rollback and versioning
* [ ] Hosting charts in private repos (CodeArtifact/S3)

---

## 14. **CI/CD & GitOps Integration**

* [ ] Jenkins + kubectl deployment pipeline
* [ ] Helm integration with CI/CD
* [ ] ArgoCD and GitOps overview
* [ ] Environment promotion (dev → stage → prod)
* [ ] Blue-Green and Canary deployment automation

---

## 15. **Troubleshooting & Maintenance**

* [ ] Pod issues (CrashLoopBackOff, ImagePullBackOff)
* [ ] Pending Pods (resource/scheduling issues)
* [ ] DNS and Service communication issues
* [ ] Node NotReady and CNI issues
* [ ] Event logs and audit troubleshooting
* [ ] Common EKS maintenance operations

---

## 16. **Security & Compliance**

* [ ] Pod Security Standards (PSS)
* [ ] Network Policies
* [ ] Restricting privileged containers
* [ ] Image security and scanning (ECR)
* [ ] Secret encryption using KMS
* [ ] IAM + IRSA for secure pod access
* [ ] Security assessment tools (kubebench, kube-hunter)

---

## 17. **Backup, Restore & Disaster Recovery**

* [ ] etcd backups in managed EKS
* [ ] Velero for backups and restores
* [ ] EBS snapshot-based backups
* [ ] Cluster migration and recovery
* [ ] Disaster recovery strategies

---

## 18. **Advanced & Real-World Topics**

* [ ] Multi-cluster management (EKS Anywhere)
* [ ] Cost optimization & capacity planning
* [ ] Observability and alerting best practices
* [ ] Real-world production scenarios and RCA handling
* [ ] Deployment lifecycle management at scale

---

## ✅ Pro Tip

Use this list as your master tracker.
Mark topics `[x]` as you complete them, and during daily sessions we’ll include the YAML manifests inline for each relevant Kubernetes object.

