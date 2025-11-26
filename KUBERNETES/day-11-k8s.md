# Horizontal Pod Autoscaler (HPA) — Detailed Explanation Notes

## **Concept / What**

HPA automatically adjusts the number of pod replicas in a Deployment/ReplicaSet/StatefulSet based on resource usage metrics like CPU, memory, or custom metrics.

---

## **Why / Purpose / Real‑World Use Cases**

* Handles fluctuating traffic without manual scaling.
* Improves performance during peak load by adding pods.
* Reduces cost by scaling down when load is low.
* Used for APIs, background workers, queue processors, microservices.

---

## **How It Works / Steps / Syntax**

### **Prerequisites**

* Metrics Server installed.
* Target workload (Deployment/RS/StatefulSet).
* Optional: custom metrics.

### **Flow**

1. Metrics Server provides CPU/memory usage.
2. HPA reads usage values.
3. If usage > target → scale up.
4. If usage < target → scale down.
5. Uses formula to calculate desired replicas:

```
desired = ceil(currentReplicas * (currentMetric / targetMetric))
```

* `ceil` = round up.
* Result is **final total replicas** (not extra replicas).

### **Important Behavior Rules**

* `scaleUp` policies decide how fast pods can increase.
* `scaleDown` policies decide how fast pods can decrease.
* Example: value 50% in scaleDown = at most 50% reduction in one step.

### **Manifest Example (HPA)**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: java-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: java-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

---

## **Common Issues / Errors**

* HPA not scaling → Metrics Server missing.
* Frequent scaling → no stabilization settings.
* Scale down delays → default 5‑min stabilization.
* Pods pending after scale‑up → not enough cluster capacity.

---

## **Troubleshooting / Fixes**

* Check metrics: `kubectl top pods`, `kubectl top nodes`.
* Check HPA status: `kubectl describe hpa`.
* Fix Metrics Server TLS/API errors.
* If pods pending → check cluster events and consider Cluster Autoscaler.

---

## **Best Practices / Tips**

* Always set resource requests on containers.
* Use autoscaling/v2 for advanced policies.
* Add stabilization windows to avoid oscillation.
* Do load testing before setting CPU thresholds.
* Use Cluster Autoscaler to support HPA during scale‑up.

---
---

# Vertical Pod Autoscaler (VPA) — Detailed Explanation Notes

## **Concept / What**

Vertical Pod Autoscaler (VPA) automatically adjusts the CPU and memory requests/limits of pods based on actual usage. It does **not** change replica count; it only tunes resource values.

---

## **Why / Purpose / Use Cases**

* For applications that do not scale well horizontally.
* Prevents OOMKilled issues when memory usage grows.
* Reduces cost by lowering over-provisioned resources.
* Useful for Java services, monoliths, ML workloads, batch jobs.
* Ensures predictable performance by providing correct CPU/memory sizes.

---

## **How It Works / Steps / Syntax**

### **VPA Components**

* **Recommender**: observes usage and suggests optimal values.
* **Updater**: evicts pods to apply new values (if Auto mode).
* **Admission Controller**: injects updated CPU/memory at pod creation.

### **VPA Modes**

* `Off`: only provides recommendations.
* `Auto`: evicts pods and applies updated values automatically.
* `Initial`: applies updated values only during pod startup.

### **VPA Manifest Example**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: java-api-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: java-api

  updatePolicy:
    updateMode: "Auto"

  resourcePolicy:
    containerPolicies:
    - containerName: api
      minAllowed:
        cpu: "200m"
        memory: "300Mi"
      maxAllowed:
        cpu: "2"
        memory: "2Gi"
```

### **Field Meaning**

* `targetRef`: workload to monitor.
* `updateMode`: how VPA applies changes.
* `minAllowed` / `maxAllowed`: boundaries for CPU/memory recommendations.

### **Works With HPA?**

* Yes, but **HPA must use custom metrics** (not CPU/memory).
* Safe custom metrics examples:

  * RPS
  * Latency
  * Error rate
  * Active sessions
  * Queue length

---

## **Common Issues / Errors**

* Frequent pod restarts in Auto mode.
* Conflicts with HPA if CPU/memory metrics are used.
* Recommendations too high due to JVM spikes.
* Apps crashing if minAllowed is too low.

---

## **Troubleshooting / Fixes**

* Check VPA status: `kubectl describe vpa <name>`.
* View recommendations: `kubectl get vpa -o yaml`.
* Set `updateMode: Off` to avoid pod eviction.
* Configure proper minAllowed/maxAllowed values.
* Use Initial mode for StatefulSets.

---

## **Best Practices / Tips**

* Start with Off mode to analyze recommendations first.
* Always set minAllowed and maxAllowed.
* Avoid HPA+VPA together unless HPA uses custom metrics.
* Use Initial mode for stateful workloads.
* Monitor VPA eviction events.

---
---

# Cluster Autoscaler — Detailed Explanation Notes

## **Concept / What**

Cluster Autoscaler (CA) automatically adjusts the number of worker nodes in a Kubernetes cluster based on pending pods and node utilization. It scales **up** when pods cannot be scheduled, and scales **down** when nodes are underutilized.

---

## **Why / Purpose / Use Cases**

* Ensures pods created by HPA/VPA can be scheduled.
* Saves cost by removing unused nodes during low traffic.
* Handles traffic spikes by adding new nodes automatically.
* Suitable for e-commerce, microservices, batch workloads, and event-driven architectures.

---

## **How It Works / Steps / Syntax**

### **Flow**

1. Pods become **Pending** due to lack of resources.
2. CA checks node groups for available capacity.
3. CA increases node count in the ASG/nodegroup.
4. New nodes join and schedule pending pods.
5. During low usage, CA drains and removes underutilized nodes.

### **Supported Backends**

* AWS EC2 Auto Scaling Groups
* EKS Managed Node Groups
* Azure VMSS
* GCP MIG

### **Key Flags**

* `--nodes=min:max:nodegroup` → scaling limits
* `--scale-down-unneeded-time` → delay before scale-down
* `--scale-down-delay-after-add` → scale-down cooldown timer

### **Simplified Manifest Example**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - name: cluster-autoscaler
        image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.27.3
        command:
          - ./cluster-autoscaler
          - --cloud-provider=aws
          - --nodes=1:10:my-nodegroup
        env:
          - name: AWS_REGION
            value: ap-south-1
```

### **AWS Node Group Types**

* **Self-Managed Node Groups** → You manage EC2, ASG, bootstrap scripts.
* **Managed Node Groups** → AWS manages OS updates, patches, draining, lifecycle.
* Both use underlying **EC2 Auto Scaling Groups**.

---

## **Common Issues / Errors**

* Pending pods due to node affinity/taints.
* Scale-down blocked by small pods on otherwise idle nodes.
* CA unable to add nodes due to ASG max limit.
* IAM permission issues.
* Spot capacity unavailable.

---

## **Troubleshooting / Fixes**

* Check pending pods: `kubectl describe pod <name>`.
* Check CA logs: `kubectl -n kube-system logs deploy/cluster-autoscaler`.
* Verify ASG/nodegroup capacity and limits.
* Ensure node IAM role allows autoscaling actions.
* Fix taints, affinity, or instance type mismatches.

---

## **Best Practices / Tips**

* Prefer **Managed Node Groups** in modern EKS setups.
* Keep separate nodegroups for On-Demand and Spot.
* Set realistic min/max node limits.
* Use proper resource requests for accurate scaling.
* Tag nodegroups correctly for CA integration.
* Avoid unnecessary anti-affinity rules.

---
---

# Metrics Server — Detailed Explanation Notes

## **Concept / What**

Metrics Server is a lightweight component that collects real-time CPU and memory usage metrics from kubelets and exposes them through the Kubernetes Metrics API. It is required for HPA (CPU/Memory-based), VPA recommendations, and commands like `kubectl top`.

---

## **Why / Purpose / Use Cases**

* Needed for HPA to scale based on CPU/Memory.
* Provides resource metrics to the Kubernetes API.
* Allows resource monitoring via `kubectl top`.
* Used by dashboards such as Lens/K9s.
* Complements Prometheus (Prometheus is for long-term metrics, Metrics Server is for real-time cluster metrics).

---

## **How It Works / Steps / Syntax**

### **Workflow**

1. Kubelet exposes resource metrics on each node.
2. Metrics Server scrapes these metrics.
3. Metrics Server exposes them to Kubernetes Metrics API.
4. HPA/VPA/kubectl top query the Metrics API.

### **Installation (EKS Example)**

**Step 1 — Install Metrics Server**:

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Step 2 — Patch Deployment for EKS**:
Add these args:

```yaml
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP
```

**Step 3 — Validate**:

```
kubectl top nodes
kubectl top pods -A
```

### **Manifest Snippet (Important Fields)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: metrics-server
        args:
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP
```

---

## **Common Issues / Errors**

* `HPA shows unknown metrics` → Metrics Server not working.
* `x509: certificate signed by unknown authority` → kubelet TLS cert issue.
* `failed to scrape node` → network or IP address mismatch.
* `kubectl top` not showing data → Metrics Server down.

---

## **Troubleshooting / Fixes**

* Check logs: `kubectl logs -n kube-system deploy/metrics-server`.
* Add `--kubelet-insecure-tls` for EKS.
* Force InternalIP with `--kubelet-preferred-address-types=InternalIP`.
* Ensure network policies allow access to kubelet port 10250.
* Verify node is reachable.

---

## **Best Practices / Tips**

* Always install Metrics Server in EKS (it is not pre-installed).
* Use the latest version.
* Keep the deployment lightweight.
* Use Prometheus for long-term historical data, not as a replacement.
* Required for HPA if scaling on CPU/Memory.

---
---

# Autoscaling Thresholds & Stabilization — Detailed Explanation Notes

## **Concept / What**

Autoscaling thresholds and stabilization define when scaling should happen and how fast scaling actions are allowed to occur. These settings control:

* When HPA should scale up or down (trigger conditions).
* How many pods can be added/removed per scaling decision (step size).
* How long HPA waits before scaling down (stabilization window).
* Hard limits using `minReplicas` and `maxReplicas`.

---

## **Why / Purpose / Use Cases**

* Prevents rapid oscillation of pods during fluctuating loads.
* Ensures fast reaction to traffic spikes and slow/safe reaction to traffic drop.
* Controls burst scaling during peak load.
* Provides predictable and stable autoscaling behavior.
* Saves cost by avoiding unnecessary scale-ups and premature scale-downs.

---

## **How It Works / Steps / Syntax**

### **1. Thresholds (Triggers)**

Defined under `averageUtilization` (ex: CPU > 70% triggers scale-up).

### **2. Rate-Based Scaling Policies (Step Size)**

Defined under `behavior.scaleUp.policies` and `behavior.scaleDown.policies`.
Example:

```yaml
policies:
- type: Percent
  value: 100
  periodSeconds: 60
```

Meaning:

* HPA can increase replicas by **up to 100% of the current count** per scaling step.
* If current = 10, allowed increase = 10 → total = 20.
* In the next step: 20 → allowed +20 → total = 40, and so on.
* Scaling never exceeds `maxReplicas`.

### **3. Stabilization Windows**

```yaml
stabilizationWindowSeconds: 300
```

Meaning:

* Scale-down is delayed (ex: 5 minutes) to avoid fast shrinking of pods.
* Scale-up window is often `0` to react immediately.

### **4. Absolute Limits (minReplicas / maxReplicas)**

Example:

```yaml
minReplicas: 2
maxReplicas: 100
```

These are **hard boundaries**.

* Even if step policy allows doubling to 160, HPA stops at **100**.

### **Full Behavior Example**

Given:

* Current = 10
* Policy = 100%
* CPU keeps crossing threshold
* `maxReplicas = 100`

Scaling steps:

```
10 → 20 → 40 → 80 → 100 (stops at max)
```

---

## **Common Issues / Errors**

* HPA oscillates rapidly due to no stabilization window.
* HPA scales slowly when thresholds are too high.
* Pods drop too fast if scale-down stabilization is too small.
* Node churn due to aggressive HPA scaling triggering Cluster Autoscaler.
* Misconfigured maxReplicas causing HPA to stop scaling early.

---

## **Troubleshooting / Fixes**

* Check HPA status: `kubectl describe hpa`.
* Verify Metrics Server is working: `kubectl top pods`.
* Adjust CPU target based on load tests.
* Add/Increase scale-down stabilization window.
* Tune rate-based policies to match workload pattern.
* Ensure maxReplicas is appropriate for traffic volume.

---

## **Best Practices / Tips**

* Use **fast scale-up** (0 sec stabilization).
* Use **slow scale-down** (300+ sec stabilization).
* Keep CPU target around 50–70% for most microservices.
* Set realistic min/max replicas.
* Test scaling behavior with load tests.
* Use custom metrics for unpredictable workloads.
* Avoid aggressive scale-down to prevent user impact.

---
---


