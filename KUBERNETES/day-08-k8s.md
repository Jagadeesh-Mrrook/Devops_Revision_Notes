# Kubernetes Notes — Namespace Creation & Purpose (Detailed Explanation Version)

## **Concept / What**

A **Namespace** is a logical partition inside a Kubernetes cluster used to organize, separate, and manage resources. It prevents name conflicts and provides boundaries for access, resource control, and environment separation.

---

## **Why / Purpose / Use Case in Real-World**

* **Multi-team isolation:** Each team controls resources in its own namespace.
* **Avoid name conflicts:** Same resource name (like `api`) can exist in multiple namespaces.
* **Environment separation:** Common namespaces — `dev`, `stage`, `prod`, `monitoring`, `logging`.
* **Resource control:** Apply ResourceQuotas & LimitRanges to restrict usage.
* **RBAC boundaries:** Grant team-specific permissions.
* **Organizational clarity:** Clear separation for apps and services.

---

## **How it Works / Steps / Syntax**

### **1. Create a Namespace (YAML)**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    team: frontend
```

### **2. Apply it**

```bash
kubectl apply -f namespace.yaml
```

### **3. Create Without YAML**

```bash
kubectl create namespace dev
```

### **Field Explanation**

* `apiVersion: v1` → Namespace is a core object.
* `kind: Namespace` → Defines this object.
* `metadata.name` → Namespace name (dev / qa / prod).
* `metadata.labels` → Useful for monitoring, cost tracking, RBAC targeting.

---

## **Common Issues / Errors**

* Accidental creation of resources in the `default` namespace.
* Applying YAML to the wrong namespace.
* RBAC permission issues (`Forbidden` error).
* Tools (Helm/ArgoCD/Jenkins) defaulting to `default` namespace.

---

## **Troubleshooting / Fixes**

* Check resources in a namespace:

  ```bash
  kubectl get pods -n dev
  ```
* Delete and re-apply in correct namespace.
* Fix RBAC with appropriate role/rolebinding.
* Always specify namespace in commands or YAML.

---

## **Best Practices / Tips**

* Avoid deploying workloads in the **default** namespace.
* Create separate namespaces per environment, team, or project.
* Always apply **ResourceQuotas + LimitRanges** to control usage.
* Use meaningful names (`payments-dev`, `app1-prod`).
* Add labels for cost/monitoring.
* **Remember: Namespaces DO NOT isolate network traffic. Only NetworkPolicies do.**

---
---

# Kubernetes Notes — ResourceQuota & LimitRange (Detailed Explanation Version)

## **Concept / What**

### **ResourceQuota**

A **ResourceQuota** restricts the total amount of CPU, memory, objects, and other resources that a single namespace can use.

### **LimitRange**

A **LimitRange** defines default, minimum, and maximum CPU/memory values **per pod or container** in a namespace.

---

## **Why / Purpose / Use Cases**

* **Prevent resource hogging:** One team cannot consume the entire cluster.
* **Avoid uncontrolled pod creation:** Limits number of pods, PVCs, service types, etc.
* **Ensure stability:** Containers cannot over-request or under-request resources.
* **Avoid cluster outage:** Prevents sudden CPU/memory exhaustion.
* **Force developers to follow resource standards:** Enforces min/max values.
* **Cost control:** Restrict creation of expensive resources (LoadBalancers, PVCs).

---

## **How it Works / Steps / Syntax**

### **1. ResourceQuota Manifest**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "8Gi"
    persistentvolumeclaims: "5"
    services.loadbalancer: "2"
```

**Field Explanation**

* `pods`: Max pod count allowed.
* `requests.cpu` / `requests.memory`: Max combined requests in namespace.
* `limits.cpu` / `limits.memory`: Max combined limits in namespace.
* `persistentvolumeclaims`: Restrict PVC creation.
* `services.loadbalancer`: Prevent unnecessary cloud load balancer costs.

**Apply & Check**

```bash
kubectl apply -f resourcequota.yaml
kubectl describe quota -n dev
```

---

### **2. LimitRange Manifest**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: 500m
        memory: 512Mi
      min:
        cpu: 50m
        memory: 64Mi
```

**Field Explanation**

* `default`: Auto-applied limit if user doesn't define one.
* `defaultRequest`: Auto-applied request if developer omits it.
* `max`: A container cannot exceed this CPU/memory.
* `min`: A container cannot request below this.

**Apply & Check**

```bash
kubectl apply -f limitrange.yaml
kubectl describe limitrange -n dev
```

---

## **How They Prevent Over-Allocation & Under-Allocation**

### **Over-Allocation Prevention**

* Developer requests too much CPU/memory → LimitRange `max` blocks it.
* Namespace tries exceeding total CPU/memory → ResourceQuota blocks it.
* Prevents one team from taking majority of cluster resources.

### **Under-Allocation Prevention**

* Developer requests too little (e.g., 0 CPU/mem) → LimitRange `min` blocks it.
* Ensures pods have enough resources to avoid crashes and OOM kills.

---

## **Real-World Scenarios**

* **Runaway CI jobs:** ResourceQuota stops unlimited pod creation.
* **App crashes due to zero resources:** LimitRange enforces minimum.
* **Teams exceeding budget:** Limits LoadBalancer & PVC count.
* **Developers requesting absurd resource values:** LimitRange caps max.

---

## **Common Issues / Errors**

* `pods is forbidden: exceeded quota`
* `exceeded quota for limits.cpu`
* `must specify memory` (LimitRange requires memory)
* Unexpected evictions due to quota exhaustion

---

## **Troubleshooting / Fixes**

* Check current quota usage:

  ```bash
  kubectl describe quota -n dev
  ```
* Check LimitRange rules:

  ```bash
  kubectl describe limitrange -n dev
  ```
* Reduce app resource requests.
* Delete unused pods to free quota.
* Adjust namespace quotas if scaling is required.

---

## **Best Practices / Tips**

* Always use ResourceQuota + LimitRange **together**.
* Set realistic limits based on actual application usage.
* Different quotas for dev/stage/prod.
* Avoid extremely low minimum values.
* Monitor namespace usage with Prometheus/Grafana.
* Update quotas regularly as workloads grow.

---
---

# Kubernetes Notes — CPU/Memory Requests & Limits (Detailed Explanation Version)

## **Concept / What**

**Requests** and **Limits** define how much CPU and memory a container is guaranteed and allowed to use.

* **Requests:** Minimum guaranteed resources. Scheduler uses this to place pods.
* **Limits:** Maximum resources a container can use. Kubelet enforces this.

---

## **Why / Purpose / Use Cases**

* Prevent one container from consuming entire node resources.
* Ensure fair usage among teams and applications.
* Enable correct scheduling decisions by Kubernetes.
* Avoid OOMKilled, throttling, node pressure, and instability.
* Maintain predictable app performance.

---

## **How It Works / Steps / Syntax**

### **1. Full Example with Requests and Limits**

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### **Scheduler Logic**

* Only **requests** matter for scheduling.
* Scheduler looks for a node with:

  * ≥ 200m CPU available
  * ≥ 256Mi memory available

### **CPU Behavior**

* Up to 500m allowed.
* Beyond 500m → **throttled**, not killed.

### **Memory Behavior**

* Up to 512Mi allowed.
* Beyond 512Mi → **OOMKilled**.

---

## **All Possible Combinations (Very Important)**

### **Case 1 — Only Requests (No Limits)**

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
```

**Behavior:**

* CPU can burst to full node capacity.
* Memory can grow until node OOM.

**Risk:** High — can starve other pods and cause node-level OOM.

---

### **Case 2 — Only Limits (No Requests)**

```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
```

**Behavior:**

* K8s auto-sets: `request = limit`.
* Hard to schedule (needs large nodes).
* CPU throttled at 500m.
* Memory OOMKilled at 512Mi.

**Risk:** Medium — scheduling issues.

---

### **Case 3 — Requests < Limits (Correct Pattern)**

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

**Behavior:**

* Best balance.
* Guaranteed baseline, allowed to burst.
* Memory safe until limit.

**Recommended for all workloads.**

---

### **Case 4 — Requests > Limits (Invalid)**

```yaml
resources:
  requests:
    cpu: 500m
  limits:
    cpu: 200m
```

**Error:** `request.cpu must be <= limit.cpu`

Pod will not start.

---

### **Case 5 — No Requests, No Limits (Very Risky)**

```yaml
resources: {}
```

**Behavior:**

* Scheduler treats requests as **0**.
* Pod can get scheduled on overloaded nodes.
* Pod can use unlimited CPU.
* Pod can consume all memory → Node OOM → other pods killed.

**Risk:** Extremely high — companies block this configuration.

---

## **Real-World Scenarios**

* **High spikes:** CPU throttling when app reaches limit.
* **Low resource apps:** Under-requesting → latency, crashes.
* **Over-requesting:** Scheduler can't place pod → `Insufficient cpu`.
* **Memory leaks:** Exceed limit → OOMKilled.

---

## **Common Issues / Errors**

* `0/5 nodes available: Insufficient cpu`
* `OOMKilled`
* `Back-off restarting failed container`
* `request.cpu must be <= limit.cpu`

---

## **Troubleshooting / Fixes**

* Check pod usage: `kubectl top pod -n <ns>`
* Check node capacity: `kubectl describe node`
* Check OOM reason: `kubectl describe pod | grep -i oom`
* Adjust requests/limits based on monitoring.

---

## **Best Practices / Tips**

* Always use **Requests < Limits**.
* Avoid missing both values.
* Use HPA/VPA to auto-adjust resources.
* Avoid setting very high limits without reason.
* Tune requests based on real usage metrics.
* Apply LimitRange & ResourceQuota for safe defaults.

---
---

# Kubernetes Notes — Isolation & Multi-Team Setup (Detailed Explanation Version)

## **Concept / What**

**Isolation** in Kubernetes means separating teams, applications, and environments so they do not interfere with each other. It is achieved through:

* **Namespaces (logical isolation)**
* **RBAC (access isolation)**
* **NetworkPolicies (network isolation)**
* **ResourceQuota & LimitRange (resource isolation)**

---

## **Why / Purpose / Use Cases**

* Prevent teams from accessing or modifying each other’s resources.
* Enforce security boundaries for sensitive applications.
* Restrict network communication between services.
* Control resource usage per team or project.
* Improve organizational clarity by grouping workloads.

---

## **How It Works / Steps / Syntax**

### **1. Namespace Isolation (Logical Separation)**

* Organizes workloads per team, project, or component.
* Does **not** provide network isolation by default.
* Used for grouping and applying quotas/policies.

---

### **2. Access Isolation with RBAC**

Restrict access so teams can only work within their own namespaces.

**Role Example:**

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: payments
  name: payments-developer
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
```

**RoleBinding Example:**

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: payments-binding
  namespace: payments
subjects:
  - kind: User
    name: team-a-dev
roleRef:
  kind: Role
  name: payments-developer
  apiGroup: rbac.authorization.k8s.io
```

---

### **3. Network Isolation with NetworkPolicies**

By default, **all pods can communicate with each other**, even across namespaces.
NetworkPolicies restrict ingress/egress.

**Deny All Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

**Allow Only Frontend → Backend:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
```

---

### **4. Resource Isolation (Quota + Limits)**

* Prevents one team from exhausting cluster resources.
* Ensures fair CPU/memory distribution.
* Limits pod count, PVC count, and LB usage.

---

## **Real-World Multi-Team Setup Examples**

* **payments** namespace → isolated with RBAC + NetworkPolicy
* **analytics** namespace → separate quotas and access
* **monitoring** namespace → open for Prometheus/Grafana
* **logging** namespace → used by Loki/ELK

Each namespace has:

* ResourceQuota
* LimitRange
* RBAC roles
* NetworkPolicies

---

## **Common Issues / Errors**

* Pods unable to communicate due to NetworkPolicy restrictions.
* `Forbidden` errors due to RBAC limitations.
* Pods failing to schedule because of ResourceQuota limits.
* YAML applied in the wrong namespace.

---

## **Troubleshooting / Fixes**

* Check access rights:

  ```bash
  kubectl auth can-i get pods -n <ns>
  ```
* View active NetworkPolicies:

  ```bash
  kubectl get networkpolicy -n <ns>
  ```
* Check quotas:

  ```bash
  kubectl describe quota -n <ns>
  ```
* Switch kubectl default namespace:

  ```bash
  kubectl config set-context --current --namespace=<ns>
  ```

---

## **Best Practices / Tips**

* Apply RBAC, NetworkPolicies, ResourceQuotas, and LimitRanges together.
* Use labels to categorize namespaces (team, environment, cost center).
* Start with default-deny NetworkPolicies.
* Avoid sharing namespaces between teams.
* Apply strict least-privilege RBAC rules.
* Use separate namespaces for critical components.

---
---

# Kubernetes Notes — Namespace & Cluster Context Switching in kubectl (Detailed Explanation Version)

## **Concept / What**

Context switching in Kubernetes means selecting which **cluster**, **user**, and **namespace** kubectl should operate on by default.

A **context** in kubeconfig is defined as:

```
cluster + user + namespace
```

You switch:

* **Between EKS clusters** using context switching.
* **Within a cluster** between namespaces using namespace switching.

---

## **Why / Purpose / Use Cases**

* Avoid accidental operations on the wrong cluster or namespace.
* Work on multiple EKS clusters (dev, stage, prod) across multiple AWS accounts.
* Reduce repetitive typing of `-n <namespace>`.
* Keep CI/CD, automation, and terminal sessions consistent.
* Prevent production outages due to context mistakes.

---

## **How It Works / Steps / Syntax**

# **1. Adding/Updating Context Using AWS CLI (Real-World Method)**

The primary and recommended method in AWS environments is:

```
aws eks update-kubeconfig --name <cluster-name> --region <region>
```

Example:

```
aws eks update-kubeconfig --name dev-eks --region ap-south-1
```

### **What this command does automatically:**

* Adds the EKS cluster entry to kubeconfig.
* Adds the IAM authentication provider.
* Adds/updates the user entry.
* Creates a context for the cluster.
* **Sets that context as the current context.**

### **Switching to another EKS cluster also works by re-running the same command:**

```
aws eks update-kubeconfig --name prod-eks --region ap-south-1
```

This updates kubeconfig and sets **prod-eks** as the active context.

---

# **2. Switching Between Existing Contexts (kubectl method)**

After kubeconfig holds multiple EKS clusters, switching between them is faster with:

```
kubectl config use-context <context-name>
```

Example:

```
kubectl config use-context dev-eks
kubectl config use-context prod-eks
```

### **Why this method exists:**

* No AWS API calls
* Instantly switches
* Keeps kubeconfig clean
* Used heavily in automation, scripts, CI/CD

Both "update-kubeconfig" and "use-context" achieve cluster switching. The difference is:

* **update-kubeconfig** edits kubeconfig and sets active context.
* **use-context** only switches without modifying cluster/user entries.

---

# **3. Switching Namespaces Within a Cluster**

Set default namespace for current context:

```
kubectl config set-context --current --namespace=<namespace>
```

Example:

```
kubectl config set-context --current --namespace=payments
```

Check current namespace:

```
kubectl config view --minify | grep namespace
```

---

# **4. Viewing Contexts and Clusters**

List all contexts:

```
kubectl config get-contexts
```

Current context:

```
kubectl config current-context
```

---

# **5. Real-World Workflow Example**

### **Step 1: Connect to dev cluster**

```
aws eks update-kubeconfig --name dev-eks --region ap-south-1
```

### **Step 2: Connect to prod cluster**

```
aws eks update-kubeconfig --name prod-eks --region ap-south-1
```

### **Step 3: Quickly switch between them**

```
kubectl config use-context dev-eks
kubectl config use-context prod-eks
```

### **Step 4: Set namespace inside each cluster**

```
kubectl config set-context --current --namespace=payments
```

---

## **Common Issues / Errors**

* Commands executed on wrong cluster due to wrong context.
* YAML accidentally applied in wrong namespace.
* Duplicate contexts created by repeated `update-kubeconfig` calls.
* IAM token authentication failures if AWS CLI is misconfigured.

---

## **Troubleshooting / Fixes**

* Check active context:

  ```bash
  kubectl config current-context
  ```
* View namespace:

  ```bash
  kubectl config view --minify | grep namespace
  ```
* Clean duplicated entries (manual cleanup in kubeconfig).
* Re-run update-kubeconfig if authentication expires:

  ```bash
  aws eks update-kubeconfig --name <cluster>
  ```

---

## **Best Practices / Tips**

* Use **aws eks update-kubeconfig** to register clusters.
* Use **kubectl config use-context** for fast switching.
* Use separate terminal tabs for dev/stage/prod.
* Always check context before running delete/apply commands.
* Use least-privilege IAM roles per cluster.
* Avoid constantly rewriting kubeconfig unless necessary.
* Always set the proper namespace in each context.

---
---
---


