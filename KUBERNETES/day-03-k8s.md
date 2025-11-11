 ## Kubernetes Deployment - Detailed Explanation

### **Concept / What**

A **Deployment** in Kubernetes is a higher-level controller used to manage stateless applications. It defines the desired state for **Pods** and automatically maintains that state by managing underlying **ReplicaSets**. Deployments ensure that the specified number of pods are always running, automatically handling failures, scaling, and updates.

---

### **Why / Purpose / Use Case in Real-World**

#### ✅ **Why We Use Deployments**

* Automates the management of Pods and ReplicaSets.
* Simplifies **scaling**, **updating**, and **rollback** operations.
* Ensures **high availability** with minimal manual intervention.
* Supports **declarative updates** and **self-healing**.

#### 💼 **Real-World Use Cases**

* Deploying microservices like `frontend`, `auth-service`, or `api-gateway`.
* Performing **zero-downtime rolling updates** for production applications.
* **Rolling back** a failed deployment version.
* Scaling out applications during traffic spikes.

---

### **How It Works / Steps / Syntax**

#### 1️⃣ Example Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3                  # Desired number of pods
  selector:
    matchLabels:
      app: nginx               # Must match pod template labels
  strategy:
    type: RollingUpdate        # Strategy used during updates
    rollingUpdate:
      maxUnavailable: 1        # Max unavailable pods during update
      maxSurge: 1              # Temporary extra pods during update
  template:
    metadata:
      labels:
        app: nginx             # Label for the pods
    spec:
      containers:
      - name: nginx
        image: nginx:1.25      # Container image and tag
        ports:
        - containerPort: 80    # Port exposed inside the pod
```

#### 2️⃣ Deploy and Monitor

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployments
kubectl rollout status deployment/nginx-deployment
```

#### 3️⃣ Update Image Version

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.26
```

#### 4️⃣ Rollback or View History

```bash
kubectl rollout history deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment
```

---

### **Common Issues / Errors**

| Issue               | Example                 | Cause                                         |
| ------------------- | ----------------------- | --------------------------------------------- |
| `CrashLoopBackOff`  | Pods restart repeatedly | App crash, bad config, or image issue         |
| `ImagePullBackOff`  | Image cannot be pulled  | Wrong image name, missing credentials         |
| `Pods not updating` | No change after update  | Rollout paused or tag unchanged               |
| `Selector mismatch` | ReplicaSet not created  | Labels mismatch between selector and template |

---

### **Troubleshooting / Fixes**

| Problem                    | Command / Fix                               |
| -------------------------- | ------------------------------------------- |
| Check rollout status       | `kubectl rollout status deployment/<name>`  |
| Describe deployment events | `kubectl describe deployment <name>`        |
| View logs of failing pod   | `kubectl logs <pod_name>`                   |
| Inspect ReplicaSets        | `kubectl get rs -l app=nginx`               |
| Rollback deployment        | `kubectl rollout undo deployment/<name>`    |
| Restart rollout manually   | `kubectl rollout restart deployment/<name>` |

---

### **Best Practices / Tips**

* Ensure **selector labels** match the **pod template labels**.
* Use **RollingUpdate** strategy for smooth upgrades.
* Always use **specific image tags**, not `latest`.
* Define **resource requests and limits** for stability.
* Add **liveness** and **readiness probes** for health checks.
* Keep manifests in **Git repositories** and use **GitOps** tools (ArgoCD, Flux) for deployments.
* Regularly check **rollout history** and **replica status** for visibility.

---

> **Summary:** Deployments are the backbone of application delivery in Kubernetes, providing a safe, declarative, and automated way to manage Pod lifecycles and perform updates or rollbacks efficiently.

---
---

## Deployment vs ReplicaSet - Detailed Explanation

### **Concept / What**

A **ReplicaSet** ensures that a specified number of identical **Pods** are always running. If a Pod fails or is deleted, the ReplicaSet automatically creates a new one to maintain the desired count.

A **Deployment** is a higher-level controller that **manages ReplicaSets**. It provides declarative updates for Pods and ReplicaSets, along with advanced features like **rollouts**, **rollbacks**, and **revision history**.

> In short: **ReplicaSet ensures count consistency**, while **Deployment adds version control, updates, and rollback features**.

---

### **Why / Purpose / Use Case in Real-World**

#### ✅ **Why ReplicaSet Alone Isn’t Enough**

* ReplicaSet only maintains a fixed number of Pods.
* It cannot perform **automatic rollouts or rollbacks**.
* There is **no version tracking** or history of changes.
* Updates must be performed **manually** (delete old Pods and create new ones).

#### ✅ **Why Deployment is Preferred**

* Automates creation and deletion of ReplicaSets.
* Provides **rolling updates** without downtime.
* Allows **rollback** to previous versions if deployment fails.
* Maintains **revision history** for traceability.
* Simplifies **scaling** and **version management**.

#### 💼 **Real-World Example**

When deploying a frontend app running on `v1`:

* With a **ReplicaSet**, you must manually delete pods and create new ones for `v2`.
* With a **Deployment**, simply update the image tag — Kubernetes automatically creates a new ReplicaSet for `v2` and performs a **rolling update** while gradually terminating old pods.

---

### **How It Works / Steps / Syntax**

#### 1️⃣ **ReplicaSet Manifest Example**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx
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
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Explanation:**

* Ensures 3 Pods are always running.
* If a Pod is deleted or fails, it creates a new one.
* No version tracking or rollout mechanism.

---

#### 2️⃣ **Deployment Manifest Example**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Explanation:**

* Manages ReplicaSets automatically.
* When the image tag is updated (e.g., `nginx:1.26`), Deployment creates a new ReplicaSet and gradually replaces old Pods.
* Keeps the previous ReplicaSet for rollback.

---

### **Common Issues / Errors**

| Issue            | Description                             | Example                                            |
| ---------------- | --------------------------------------- | -------------------------------------------------- |
| Label mismatch   | ReplicaSet/Deployment not managing Pods | Selector and template labels differ                |
| Stale ReplicaSet | Old ReplicaSet remains active           | Rollout incomplete or paused                       |
| Rollback fails   | Revision history cleared                | Used `kubectl rollout undo` after clearing history |

---

### **Troubleshooting / Fixes**

| Problem                                | Command / Fix                                  |
| -------------------------------------- | ---------------------------------------------- |
| View ReplicaSets managed by Deployment | `kubectl get rs -l app=<label>`                |
| Check rollout status                   | `kubectl rollout status deployment/<name>`     |
| Rollback to previous version           | `kubectl rollout undo deployment/<name>`       |
| Scale replicas                         | `kubectl scale deployment <name> --replicas=5` |
| Delete old ReplicaSet                  | `kubectl delete rs <name>`                     |

---

### **Best Practices / Tips**

* Always use **Deployments** for managing stateless applications.
* Use **ReplicaSets** directly only for low-level control or testing.
* Ensure **selector labels match** pod template labels.
* Avoid manually editing ReplicaSets managed by Deployments.
* Use **RollingUpdate** strategy for zero downtime.
* Track rollout history using `kubectl rollout history`.

---

### **Summary Table**

| Feature         | ReplicaSet                    | Deployment                               |
| --------------- | ----------------------------- | ---------------------------------------- |
| Purpose         | Maintain fixed number of Pods | Manage ReplicaSets and automate rollouts |
| Rollouts        | Manual                        | Automatic (RollingUpdate, Recreate)      |
| Rollback        | Not supported                 | Supported                                |
| Version History | None                          | Maintained                               |
| Scaling         | Manual                        | Declarative                              |
| Typical Use     | Internal component            | Real-world application management        |

---

> **Summary:** A ReplicaSet ensures a stable set of Pods, while a Deployment provides declarative updates, rollbacks, and version management. In real-world Kubernetes environments, Deployments are almost always used to manage Pods efficiently and safely.

---
---

## Rolling Updates & Rollbacks - Detailed Explanation

### **Concept / What**

A **Rolling Update** in Kubernetes is the default Deployment update strategy that updates Pods **gradually**, ensuring that some old version Pods remain running while new ones start. This ensures **zero downtime** and smooth version transitions.

A **Rollback** allows reverting a Deployment to a previous stable version if the new rollout fails (for example, due to a bad image, crash, or misconfiguration).

> **Rolling Update = gradual replacement**
> **Rollback = revert to previous working version**

---

### **Why / Purpose / Real-World Use Case**

#### ✅ **Why Rolling Updates**

* Avoids downtime during upgrades.
* Keeps services continuously available.
* Helps test new versions safely in production.

#### ✅ **Why Rollbacks**

* Quickly recover from deployment failures.
* Maintain service stability.
* Prevent extended outages due to bad versions.

#### 💼 **Real-World Example**

When updating `frontend-app:v1` to `frontend-app:v2`, Kubernetes gradually replaces old pods (v1) with new ones (v2). If issues arise (crashes or latency), a rollback command restores the old version instantly.

---

### **How It Works / Steps / Syntax**

#### 1️⃣ **Deployment Manifest Example (with RollingUpdate Strategy)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1     # Max unavailable pods during rollout
      maxSurge: 1           # Extra pods allowed temporarily
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
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Key Parameters Explained:**

* `type: RollingUpdate` → Default update strategy.
* `maxUnavailable` → Controls downtime (pods allowed to be unavailable).
* `maxSurge` → Controls speed (extra pods allowed during rollout).

---

### **2️⃣ Performing a Rolling Update**

#### 🧩 **Declarative Way (Preferred)**

```bash
# Update YAML file to new image version
kubectl apply -f deployment.yaml
```

Kubernetes detects the image change and performs a rolling update automatically.

#### 🧩 **Imperative Way (Quick Manual Update)**

```bash
kubectl set image deployment/nginx-deploy nginx=nginx:1.26
```

This updates the Deployment image directly from the CLI.

| Command             | Type        | Used In          | Description                             |
| ------------------- | ----------- | ---------------- | --------------------------------------- |
| `kubectl apply -f`  | Declarative | CI/CD, GitOps    | YAML-driven, version-controlled updates |
| `kubectl set image` | Imperative  | Manual / Testing | Quick one-liner updates                 |

> In **real-world pipelines**, teams prefer the **`kubectl apply -f`** method as it aligns with **GitOps** principles.

---

### **3️⃣ Monitor Rollout Progress**

```bash
kubectl rollout status deployment/nginx-deploy
```

Shows live rollout status (e.g., "updated 2 of 4 replicas").

---

### **4️⃣ View Rollout History**

```bash
kubectl rollout history deployment/nginx-deploy
```

Lists previous ReplicaSet revisions for rollback reference.

---

### **5️⃣ Perform Rollback**

```bash
kubectl rollout undo deployment/nginx-deploy
```

Reverts to the previous working version.

Or rollback to a specific version:

```bash
kubectl rollout undo deployment/nginx-deploy --to-revision=2
```

---

### **Common Issues / Errors**

| Issue             | Example                         | Cause                                             |
| ----------------- | ------------------------------- | ------------------------------------------------- |
| Rollout stuck     | "waiting for rollout to finish" | Readiness probe failure or insufficient resources |
| Image not updated | Pods still running old version  | Image cached or missing `imagePullPolicy: Always` |
| Rollback failed   | No previous revision            | Deployment recently created or history cleared    |
| Too many pods     | Temporary surge                 | `maxSurge` set too high                           |

---

### **Troubleshooting / Fixes**

| Problem                | Fix / Command                                              |
| ---------------------- | ---------------------------------------------------------- |
| Check rollout details  | `kubectl describe deployment <name>`                       |
| Inspect rollout events | `kubectl get events --sort-by=.metadata.creationTimestamp` |
| View logs for new pods | `kubectl logs <new_pod_name>`                              |
| Pause a rollout        | `kubectl rollout pause deployment/<name>`                  |
| Resume rollout         | `kubectl rollout resume deployment/<name>`                 |
| Restart rollout        | `kubectl rollout restart deployment/<name>`                |

---

### **Best Practices / Tips**

* Always configure **readiness probes** for smooth rollouts.
* Use **maxUnavailable=1** and **maxSurge=1** for production-safe updates.
* Avoid using the **latest** tag — use fixed, versioned tags.
* Monitor rollouts with **Prometheus**, **Grafana**, or `kubectl rollout status`.
* Maintain **rollout history** for at least 10 revisions.
* Use **`kubectl apply -f`** in CI/CD (declarative management).
* Test rollback procedure in non-prod environments regularly.

---

### **Summary Table**

| Feature          | Rolling Update                                   | Rollback                   |
| ---------------- | ------------------------------------------------ | -------------------------- |
| Purpose          | Gradual update to new version                    | Revert to previous version |
| Downtime         | Zero (if configured correctly)                   | Minimal                    |
| Default Strategy | ✅ Yes                                            | ✅ Supported                |
| Common Commands  | `set image`, `rollout status`, `rollout history` | `rollout undo`             |
| Controlled By    | Deployment                                       | Deployment                 |

---

> **Summary:** Rolling Updates ensure zero-downtime upgrades by gradually replacing Pods with newer versions. Rollbacks provide a safety net to revert to the last stable state quickly if the rollout fails. Together, they form the backbone of reliable Kubernetes application delivery.

---
---

## Deployment Strategies (Recreate vs RollingUpdate) - Detailed Explanation

### **Concept / What**

A **Deployment Strategy** defines how Kubernetes replaces existing Pods with new ones when an update occurs (such as a new image version). The two primary strategies are:

| Strategy          | Description                                                                        |
| ----------------- | ---------------------------------------------------------------------------------- |
| **Recreate**      | Deletes all existing Pods before creating new ones — causes downtime.              |
| **RollingUpdate** | Gradually replaces old Pods with new ones — minimizes downtime (default strategy). |

---

### **Why / Purpose / Use Case in Real-World**

#### ✅ **Why Recreate Strategy**

* Needed when **old and new versions cannot coexist** (e.g., DB schema changes, breaking API updates).
* Simpler, predictable, but causes **complete downtime** during the update.
* Often used in **non-production environments** where temporary downtime is acceptable.

#### ✅ **Why RollingUpdate Strategy**

* Ensures **continuous application availability** during updates.
* Suitable for **stateless applications** where multiple versions can safely run together.
* Used in **production** for almost all web and microservice-based workloads.

#### 💼 **Real-World Example**

* When updating a backend service with a breaking schema change → **Recreate**.
* When updating a frontend image from `v1` to `v2` → **RollingUpdate**.

---

### **How It Works / Steps / Syntax**

#### 1️⃣ **Recreate Strategy Example**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-recreate
spec:
  strategy:
    type: Recreate
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: myapp:v1
        ports:
        - containerPort: 8080
```

**Explanation:**

* `strategy.type: Recreate` → Deletes all old Pods first.
* No old Pods serve traffic until new ones are ready → downtime guaranteed.

---

#### 2️⃣ **RollingUpdate Strategy Example**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-rolling
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1     # Max unavailable pods during rollout
      maxSurge: 1           # Extra pods allowed temporarily
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: myapp:v2
        ports:
        - containerPort: 8080
```

**Explanation:**

* `type: RollingUpdate` → Default strategy for Deployments.
* `maxUnavailable` → Controls downtime (how many pods can be unavailable).
* `maxSurge` → Controls rollout speed (how many extra pods can exist temporarily).
* New Pods are created gradually, and once ready, old Pods are terminated.

---

### **Common Issues / Errors**

| Issue              | Strategy      | Cause                                    |
| ------------------ | ------------- | ---------------------------------------- |
| Downtime           | Recreate      | All pods deleted before new ones start   |
| Rollout hangs      | RollingUpdate | Readiness probe failure or startup delay |
| Incomplete update  | RollingUpdate | Image not updated or rollout paused      |
| App not responding | RollingUpdate | Misconfigured `maxUnavailable` or probes |

---

### **Troubleshooting / Fixes**

| Problem                       | Command / Fix                                                    |
| ----------------------------- | ---------------------------------------------------------------- |
| Rollout taking too long       | Check readiness probes with `kubectl describe deployment <name>` |
| App unavailable after rollout | Verify if Recreate strategy is used (expected downtime)          |
| Check rollout events          | `kubectl get events --sort-by=.metadata.creationTimestamp`       |
| Pause / resume rollout        | `kubectl rollout pause/resume deployment/<name>`                 |
| Revert to stable version      | `kubectl rollout undo deployment/<name>`                         |

---

### **Best Practices / Tips**

* ✅ Use **RollingUpdate** for 99% of production workloads.
* ⚙️ Use **Recreate** only when old and new versions conflict.
* 📈 Adjust `maxSurge` and `maxUnavailable` to balance speed vs. availability.
* 🔍 Always define **readiness probes** to prevent premature traffic routing.
* 🚀 Maintain 3+ replicas for safe rolling updates.
* 🧩 Verify rollout status using `kubectl rollout status`.

---

### **Summary Table**

| Feature       | Recreate            | RollingUpdate                          |
| ------------- | ------------------- | -------------------------------------- |
| Downtime      | ✅ Full downtime     | ⚠️ Minimal (can be near-zero)          |
| Default       | ❌ No                | ✅ Yes                                  |
| Rollout Speed | Fast but disruptive | Gradual and safe                       |
| Availability  | Interrupted         | Maintained                             |
| Coexistence   | Not allowed         | Allowed                                |
| Use Case      | Breaking changes    | Stateless updates                      |
| Configuration | Simple              | Tunable (`maxSurge`, `maxUnavailable`) |

---

> **Summary:** The Recreate strategy replaces all pods at once, causing downtime but ensuring a clean version swap. The RollingUpdate strategy gradually replaces old pods with new ones, minimizing downtime and maintaining availability, making it the default and preferred choice for production environments.

---
---

## Blue-Green & Canary Deployment Patterns (Backend-Focused) - Detailed Explanation

### **Concept / What**

Deployment patterns like **Blue-Green** and **Canary** are advanced Kubernetes deployment strategies mainly used for **backend applications** to ensure **safe, zero-downtime**, and **controlled rollouts**.

| Strategy                  | Definition                                                                                                                                                                                   |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Blue-Green Deployment** | Run two versions of the backend service simultaneously (Blue = current live version, Green = new version). After verifying the new version, traffic is switched from Blue → Green instantly. |
| **Canary Deployment**     | Gradually roll out the new backend version to a small subset of users or pods. Monitor performance and expand if stable.                                                                     |

---

### **Why / Purpose / Real-World Backend Use Case**

#### ✅ **Why Blue-Green**

* Ensures **zero downtime** for backend updates.
* Allows **instant rollback** (switch traffic back to Blue if issues arise).
* Enables **testing of new API versions** in real environment before full release.

**Example:**

* `orders-service:v1` (Blue) → current live version.
* `orders-service:v2` (Green) → deployed in parallel.
* Once validated → Service selector switches to Green.
* Rollback = switch back to Blue.

#### ✅ **Why Canary**

* Reduces risk by exposing new version to **limited traffic first**.
* Allows real-world validation on production traffic.
* Prevents mass failures during version upgrades.

**Example:**

* Deploy `orders-service:v2` to 1 out of 5 pods.
* Monitor metrics (latency, errors).
* Gradually scale up `v2` and scale down `v1` once stable.

---

### **How It Works / Steps / Syntax**

#### 🧠 **Blue-Green Deployment (Backend Example)**

```yaml
# Blue Deployment (v1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders-api
      version: blue
  template:
    metadata:
      labels:
        app: orders-api
        version: blue
    spec:
      containers:
      - name: orders-api
        image: mycompany/orders-service:v1
        ports:
        - containerPort: 8080
---
# Green Deployment (v2)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders-api
      version: green
  template:
    metadata:
      labels:
        app: orders-api
        version: green
    spec:
      containers:
      - name: orders-api
        image: mycompany/orders-service:v2
        ports:
        - containerPort: 8080
---
# Common Service (routes to Blue or Green)
apiVersion: v1
kind: Service
metadata:
  name: orders-api-service
spec:
  selector:
    app: orders-api
    version: blue   # 🔁 Switch to 'green' once validated
  ports:
  - port: 80
    targetPort: 8080
```

**Switch traffic instantly:**

```bash
kubectl patch service orders-api-service -p '{"spec":{"selector":{"app":"orders-api","version":"green"}}}'
```

✅ **Result:**

* Blue pods (v1) keep running but no longer serve traffic.
* Green pods (v2) start serving API requests.
* Rollback = instant switch of Service selector back to Blue.

---

#### 🧠 **Canary Deployment (Backend Example)**

```yaml
# Stable Deployment (v1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api-stable
spec:
  replicas: 4
  selector:
    matchLabels:
      app: orders-api
      version: stable
  template:
    metadata:
      labels:
        app: orders-api
        version: stable
    spec:
      containers:
      - name: orders-api
        image: mycompany/orders-service:v1
        ports:
        - containerPort: 8080
---
# Canary Deployment (v2)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api-canary
spec:
  replicas: 1        # small portion of traffic
  selector:
    matchLabels:
      app: orders-api
      version: canary
  template:
    metadata:
      labels:
        app: orders-api
        version: canary
    spec:
      containers:
      - name: orders-api
        image: mycompany/orders-service:v2
        ports:
        - containerPort: 8080
---
# Shared Service
apiVersion: v1
kind: Service
metadata:
  name: orders-api-service
spec:
  selector:
    app: orders-api
  ports:
  - port: 80
    targetPort: 8080
```

✅ **Behavior:**

* Both `stable` (v1) and `canary` (v2) pods receive traffic.
* With 4 stable + 1 canary replica, ~20% of requests go to v2.
* Monitor metrics (latency, error rate, CPU usage).
* If stable → scale canary up, remove stable.
* If issues → delete canary pods for instant rollback.

---

### **Common Issues / Errors**

| Issue                          | Type       | Cause                                        |
| ------------------------------ | ---------- | -------------------------------------------- |
| Old pods still serving traffic | Blue-Green | Service selector not switched properly       |
| Traffic imbalance              | Canary     | Incorrect replica ratio or label mismatch    |
| Health check failures          | Both       | Unready pods or misconfigured probes         |
| Rollback delay                 | Canary     | Manual scale-down delay                      |
| Version conflicts              | Both       | Shared DB schema or incompatible API changes |

---

### **Troubleshooting / Fixes**

| Problem                                   | Command / Fix                                                |
| ----------------------------------------- | ------------------------------------------------------------ |
| Check which pods are serving traffic      | `kubectl get endpoints orders-api-service -o wide`           |
| Switch traffic back (rollback Blue-Green) | Patch Service selector back to Blue                          |
| Increase canary traffic gradually         | `kubectl scale deployment orders-api-canary --replicas=3`    |
| Delete failed canary pods                 | `kubectl delete deployment orders-api-canary`                |
| Monitor rollout metrics                   | Use Prometheus, Grafana, or `kubectl logs -l app=orders-api` |

---

### **Best Practices / Tips**

* ✅ Always configure **readiness and liveness probes**.
* ⚙️ Use **Blue-Green** for production-safe major releases.
* 🧩 Use **Canary** for incremental version rollouts.
* 🔁 Keep Blue environment live for quick rollback.
* 📈 Automate rollout and rollback via **Argo Rollouts** or **Jenkins pipelines**.
* 🔍 Monitor **latency, error rate, and API response codes** during rollout.
* 🧰 Use labels (`version=blue/green/canary`) consistently for clarity.

---

### **Summary Table (Backend Context)**

| Strategy          | Traffic Switch | Downtime         | Rollback                          | Cost                  | Use Case                    |
| ----------------- | -------------- | ---------------- | --------------------------------- | --------------------- | --------------------------- |
| **Blue-Green**    | Instant        | Practically zero | Instant (switch Service selector) | High (duplicate pods) | Safe backend releases       |
| **Canary**        | Gradual        | Minimal          | Gradual (scale down canary)       | Medium                | Controlled API rollout      |
| **RollingUpdate** | Automatic      | Low              | Supported                         | Normal                | Default Deployment behavior |

---

> **Summary:** For backend microservices running in Kubernetes, Blue-Green deployments offer instant traffic switching and safe rollback, while Canary deployments enable gradual exposure of new versions to minimize risk. Both strategies help achieve near-zero downtime and controlled, production-safe releases.

---
---

## Deployment Revision History - Detailed Explanation

### **Concept / What**

In Kubernetes, **Deployment Revision History** refers to how Kubernetes keeps track of every change made to a Deployment over time. Each change creates a new **ReplicaSet**, which represents a new **revision**. These revisions allow you to view, manage, and roll back to previous versions of your application.

🧠 Think of it like version control for your Deployments — each update is a new commit (revision).

---

### **Why / Purpose / Real-World Use Case**

#### ✅ **Why It’s Useful**

* Enables quick **rollback** to stable versions if a new deployment fails.
* Helps in **debugging** or comparing previous configurations.
* Provides **deployment traceability** — who deployed what, and when.

#### 💼 **Real-World Example**

Imagine your `payment-api` deployment:

1. `v1` → deployed (Revision 1)
2. `v2` → new rollout (Revision 2)
3. `v3` → another rollout (Revision 3)

If version 3 causes errors, you can roll back to version 2 instantly:

```bash
kubectl rollout undo deployment/payment-api --to-revision=2
```

---

### **How It Works / Steps / Syntax**

#### 1️⃣ **View Revision History**

```bash
kubectl rollout history deployment/payment-api
```

Example output:

```
REVISION  CHANGE-CAUSE
1         Initial deploy (v1)
2         Image updated to v2
3         Image updated to v3
```

#### 2️⃣ **View Specific Revision Details**

```bash
kubectl rollout history deployment/payment-api --revision=2
```

Shows image version, environment variables, and configuration.

#### 3️⃣ **Rollback to a Specific Revision**

```bash
kubectl rollout undo deployment/payment-api --to-revision=2
```

Rolls back to revision 2 and replaces the current ReplicaSet with that configuration.

---

### **Revision Storage Control: `revisionHistoryLimit`**

Kubernetes controls how many old revisions are retained using the field:

```yaml
spec:
  revisionHistoryLimit: 10
```

If this is **not specified**, Kubernetes uses a **default of 10**.

When the number of revisions exceeds the limit, Kubernetes automatically **deletes the oldest ReplicaSets**.

#### ✅ **Behavior Summary**

| Parameter                 | Description                |
| ------------------------- | -------------------------- |
| Default                   | `revisionHistoryLimit: 10` |
| Minimum                   | 0 (no history stored)      |
| Recommended Range         | 5–20                       |
| Effect of Exceeding Limit | Oldest ReplicaSets deleted |

#### 💡 **Example YAML**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  replicas: 3
  revisionHistoryLimit: 5   # Keeps only last 5 revisions
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      containers:
      - name: payment-api
        image: mycompany/payment-service:v3
```

---

### **Common Issues / Errors**

| Issue                          | Cause                                 | Fix                                  |
| ------------------------------ | ------------------------------------- | ------------------------------------ |
| No previous revision found     | New deployment or old history deleted | Increase `revisionHistoryLimit`      |
| Rollback failed                | Revision number not available         | Check history first                  |
| Too many ReplicaSets retained  | Default limit too high                | Set a smaller `revisionHistoryLimit` |
| Frequent unnecessary revisions | Metadata-only updates                 | Avoid frequent meaningless applies   |

---

### **Troubleshooting / Fixes**

| Problem                        | Command / Fix                               |
| ------------------------------ | ------------------------------------------- |
| View all rollout revisions     | `kubectl rollout history deployment/<name>` |
| View specific revision details | `kubectl rollout history --revision=<n>`    |
| Check current revision         | `kubectl describe deployment <name>`        |
| Clean old ReplicaSets          | `kubectl delete rs <replicaset-name>`       |
| Rollback to previous version   | `kubectl rollout undo deployment/<name>`    |

---

### **Best Practices / Tips**

* ✅ Always set `revisionHistoryLimit` explicitly (5–10 for CI/CD environments).
* ✅ Use `kubectl annotate` with `kubernetes.io/change-cause` to document why a change was made:

  ```bash
  kubectl annotate deployment payment-api kubernetes.io/change-cause="Updated to v2 - fixed logging"
  ```
* ⚙️ Regularly clean up old ReplicaSets to reduce etcd load.
* 🧩 Avoid generating revisions for non-functional metadata-only changes.
* 📋 Keep CI/CD change logs for better traceability.

---

### 🔍 **Limitations / Trade-offs / Important Notes**

⚠️ **1️⃣ Default Limit of 10**

* If `revisionHistoryLimit` is not specified, Kubernetes keeps only the last 10 revisions.
* Any older ReplicaSets are automatically garbage-collected and cannot be rolled back.

⚠️ **2️⃣ Not a Full Configuration Diff Tool**

* Kubernetes does not store detailed diffs of what changed, only the final ReplicaSet states.
* You cannot compare two revisions directly using `kubectl`.

⚠️ **3️⃣ Rollback Recreates Pods**

* Rollbacks are not instant; they terminate current pods and spin up new ones — slight latency possible.

⚠️ **4️⃣ Storage Overhead**

* Keeping many historical ReplicaSets increases cluster metadata and etcd size.
  Tune `revisionHistoryLimit` wisely.

⚠️ **5️⃣ Per-Deployment Basis**

* Revision tracking is per Deployment — unrelated Deployments do not share history.

---

### 🧠 **Summary Table**

| Feature                | Purpose                               | Default              | Command                                  |
| ---------------------- | ------------------------------------- | -------------------- | ---------------------------------------- |
| View rollout history   | Shows all previous revisions          | 10 stored by default | `kubectl rollout history`                |
| View specific revision | Inspect details of one version        | N/A                  | `kubectl rollout history --revision=<n>` |
| Rollback               | Revert to previous version            | Uses last revision   | `kubectl rollout undo`                   |
| Limit history          | Restrict number of stored ReplicaSets | 10                   | `revisionHistoryLimit`                   |

---

> **Summary:** Kubernetes automatically keeps track of Deployment revisions using ReplicaSets. By default, it stores up to 10 previous revisions, allowing rollback to any of them. The number of stored revisions can be adjusted with `revisionHistoryLimit`. However, keeping too many increases metadata overhead, while too few limit rollback options — so balance is key.

---
---

## Rollout Pause, Resume, Undo - Detailed Explanation

### **Concept / What**

Kubernetes provides manual rollout control commands for Deployments: `pause`, `resume`, and `undo`. These commands allow you to manage ongoing rollouts safely. A rollout represents the process of updating a Deployment (for example, updating image versions or replicas).

| Command                  | Purpose                              |
| ------------------------ | ------------------------------------ |
| `kubectl rollout pause`  | Temporarily stop an ongoing rollout. |
| `kubectl rollout resume` | Continue a paused rollout.           |
| `kubectl rollout undo`   | Roll back to a previous revision.    |

These commands help ensure safer updates and enable manual validation during deployments.

---

### **Why / Purpose / Use Case**

* **Pause:** Used when you want to stop an active rollout temporarily to inspect or validate changes before continuing.
* **Resume:** Used after validation to allow Kubernetes to continue replacing pods with the new version.
* **Undo:** Used to revert to the previous or a specific stable version if the current rollout causes issues.

**Real-World Use Case:**
If a Jenkins pipeline deploys an application (`kubectl apply -f deployment.yaml`) and you notice pods failing health checks, you can manually pause the rollout, fix the issue, and then resume or undo it.

---

### **How It Works / Steps / Syntax**

1. **Check rollout status:**

   ```bash
   kubectl rollout status deployment/orders-api
   ```
2. **Pause rollout:**

   ```bash
   kubectl rollout pause deployment/orders-api
   ```
3. **Fix issue (if any):**
   Update image, config, or resource definitions.
4. **Resume rollout:**

   ```bash
   kubectl rollout resume deployment/orders-api
   ```
5. **Undo rollout:**

   ```bash
   kubectl rollout undo deployment/orders-api
   ```

   or roll back to a specific revision:

   ```bash
   kubectl rollout undo deployment/orders-api --to-revision=2
   ```

**Verification Commands:**

```bash
kubectl rollout history deployment/orders-api
kubectl get rs -l app=orders-api
```

---

### **Common Issues / Troubleshooting**

| Issue                        | Cause                                     | Fix                                 |
| ---------------------------- | ----------------------------------------- | ----------------------------------- |
| Rollout stuck                | Deployment paused or probe failures       | `kubectl rollout resume`            |
| Rollback failed              | Revision deleted due to low history limit | Increase `revisionHistoryLimit`     |
| Pods not updating            | Wrong image tag or context                | Check `kubectl describe deployment` |
| Rollout paused automatically | Misconfigured readiness/liveness probes   | Inspect deployment and fix probes   |

---

### **Best Practices / Tips**

* Use **pause → validate → resume** flow during sensitive updates.
* Always monitor rollout using `kubectl rollout status`.
* Combine with **readiness probes** to avoid serving traffic from unready pods.
* Set `revisionHistoryLimit` high enough for reliable rollback.
* Automate rollbacks for failed health checks in CI/CD.
* Document changes using `kubectl annotate deployment <name> kubernetes.io/change-cause="..."`.

---

### **Limitations / Trade-offs / Important Notes**

* Rollbacks re-create pods, so temporary latency is possible.
* Only revisions stored under `revisionHistoryLimit` can be rolled back.
* Rollout commands are **manual control mechanisms**; CI/CD systems like ArgoCD or Flux handle rollouts declaratively instead.
* In automated setups (like Jenkins), if the rollout is automated, these commands can still be run manually in parallel for troubleshooting.

---

### **Manual Rollout Operations in Real-World Setup**

In production setups, CI/CD pipelines (like Jenkins) run automatically on **Jenkins Agent EC2 instances**, which have all required tools — `kubectl`, `AWS CLI`, `Terraform`, `Ansible`, and `Maven`. DevOps engineers do not perform these operations from local laptops.

For manual interventions (pause, resume, undo):

1. **SSH into Bastion Host** (public subnet)

   ```bash
   ssh -i devops-key.pem ec2-user@<bastion-public-ip>
   ```
2. **SSH from Bastion → Jenkins Agent (private subnet)**

   ```bash
   ssh ec2-user@<jenkins-agent-private-ip>
   ```
3. **Run commands on Jenkins Agent**

   ```bash
   aws eks update-kubeconfig --name prod-cluster --region ap-south-1
   kubectl rollout pause deployment/orders-api
   kubectl rollout resume deployment/orders-api
   kubectl rollout undo deployment/orders-api
   ```

This multi-hop access model ensures security while allowing controlled manual operations.

**Why this model is used:**

* Jenkins Agent has all CLIs and IAM permissions.
* Bastion is the secure public entry point.
* Jenkins Agents are in private subnets (no direct SSH access).
* Ensures compliance and centralized auditing.

---

### **Summary Table**

| Command          | Purpose                        | Used In                        | Notes                            |
| ---------------- | ------------------------------ | ------------------------------ | -------------------------------- |
| `rollout pause`  | Temporarily freeze a rollout   | During manual inspection       | Safe and reversible              |
| `rollout resume` | Continue rollout               | After validation               | Ensures update continuation      |
| `rollout undo`   | Roll back to previous revision | During issue recovery          | Recreates old ReplicaSet         |
| Access Method    | Bastion → Jenkins Agent        | Secure production environments | No direct SSH from local laptops |

---

**Summary:** Rollout control commands (`pause`, `resume`, `undo`) are crucial for managing Kubernetes deployments safely. In real-world CI/CD setups, DevOps engineers connect securely via the Bastion host to the Jenkins agent EC2 (where kubectl and AWS CLI are installed) to manually execute these commands for debugging, rollback, or controlled release operations.

---
---
---


