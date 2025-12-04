# CI/CD & GitOps Integration — Jenkins + kubectl Deployment Pipeline (Detailed Explanation Version)

## **• Concept / What**

A Jenkins + kubectl deployment pipeline is a CI/CD workflow where Jenkins builds the application, creates and pushes Docker images, and deploys updates to Kubernetes using `kubectl` commands or Kubernetes manifest files. Jenkins directly controls the deployment process without Helm or GitOps tools.

---

## **• Why / Purpose / Use Case in Real-World**

* Used when teams want a simple, direct CI → CD deployment method.
* Useful when Helm/ArgoCD/GitOps tools are not implemented yet.
* Ideal for smaller teams or legacy pipelines already built on Jenkins.
* Gives full control of rollout, image updates, and validation inside Jenkins.

**Real scenarios:**

* Fintech/startups using Jenkins scripted/freestyle jobs.
* Deploying microservices to EKS using kubectl.
* Triggering deployments automatically after a successful image push.
* Using Jenkins to restart deployments or update image tags.

**Benefits:** Simple setup, faster implementation, direct control.

---

## **• How It Works / Steps / Syntax**

### **1. Jenkins checks out the code**

Pulls source code from GitHub/GitLab.

### **2. Jenkins builds the Docker image**

Example:

```
docker build -t myapp:v1 .
```

### **3. Jenkins pushes the image to ECR (AWS)**

```
aws ecr get-login-password | docker login ...
docker push <ecr-url>/myapp:v1
```

### **4. Jenkins deploys to Kubernetes using kubectl**

Methods:

* Apply the manifest:

  ```
  kubectl apply -f deployment.yaml
  ```
* Update image tag:

  ```
  kubectl set image deployment/myapp myapp=<image:tag> --record
  ```

### **5. Jenkins verifies the rollout**

```
kubectl rollout status deployment/myapp
```

---

## **• Example Deployment Manifest (Explained)**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: <ecr-url>/myapp:v1
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "300m"
              memory: "256Mi"
```

**Key fields:**

* `apiVersion`: Kubernetes API version.
* `kind`: Kubernetes object type.
* `metadata`: Deployment name + labels.
* `spec.replicas`: Pod count.
* `selector.matchLabels`: Must match pod labels.
* `template`: Pod definition.
* `containers.image`: Jenkins updates this tag.
* `resources`: Limit and request CPU/memory.

---

## **• Common Issues / Errors**

### **1. ImagePullBackOff**

* Cause: Cluster cannot pull image from ECR.
* Reason: Missing IAM role or imagePullSecret.

### **2. kubectl not found in Jenkins agent**

* Cause: Jenkins agent doesn’t have kubectl installed.

### **3. Wrong kubeconfig/context**

* Cause: Jenkins using wrong cluster config.

### **4. Rollout stuck**

* Cause: Failing readiness/liveness probes.

---

## **• Troubleshooting / Fixes**

* Configure IAM roles correctly for ECR pulls.
* Install kubectl on Jenkins agent (or use container agent).
* Use correct Kubernetes context or update kubeconfig:

  ```
  aws eks update-kubeconfig --name cluster --region ap-south-1
  ```
* Check logs and probe paths when pods fail to become Ready.
* Use `kubectl describe pod` for event-level debugging.

---

## **• Best Practices / Tips**

* Store kubeconfig, AWS credentials, and secrets in Jenkins credentials.
* Avoid hardcoding credentials or cluster configs in the Jenkinsfile.
* Prefer `kubectl set image` for automated deployments.
* Enable rollout history (`--record`).
* Add readiness/liveness probes.
* Use separate namespaces for dev/stage/prod.
* Tag images with build numbers for traceability.

---
---

# Helm Integration with CI/CD — Detailed Explanation Version

## **• Concept / What**

Helm is a Kubernetes package manager that bundles Kubernetes manifests into reusable packages called Charts. In CI/CD, Helm automates deployment, templating, configuration management, versioning, and rollbacks. Instead of applying raw YAML files with `kubectl apply`, CI/CD pipelines use `helm upgrade --install` to deploy applications.

---

## **• Why / Purpose / Use Case in Real-World**

* Helps reuse the same Kubernetes manifests across multiple environments using separate values files.
* Simplifies configuration management with templating.
* Provides versioning for deployments through Helm releases.
* Allows quick rollback in case of deployment failures.
* Prevents YAML duplication across dev, stage, and prod.
* Supports cleaner CI/CD pipelines for deployment automation.

**Real scenarios:**

* Microservices sharing a single chart but using values specific to each environment.
* Deploying to EKS using Jenkins/GitHub Actions workflows.
* Automating application promotion from dev → stage → prod.
* Organizations using Helm charts stored in internal chart repositories.

---

## **• How It Works / Steps / Syntax**

### **1. Helm chart directory structure**

```
myapp/
 ├── Chart.yaml
 ├── values.yaml
 ├── templates/
      ├── deployment.yaml
      ├── service.yaml
      ├── ingress.yaml
```

### **2. CI pulls code + chart**

Pipeline checks out source code and Helm chart.

### **3. CI builds and pushes Docker image**

Pushes application image to ECR or other registries.

### **4. CI updates values.yaml with new image tag**

```
image:
  repository: <ecr-url>/myapp
  tag: "build-123"
```

### **5. CI deploys using Helm**

```
helm upgrade --install myapp ./myapp -f values-prod.yaml --namespace prod
```

### **6. CI verifies rollout**

```
kubectl rollout status deployment/myapp
```

---

## **• Helm Chart Manifest Files (Explained)**

### **Chart.yaml**

```
apiVersion: v2
name: myapp
version: 1.0.0
description: A sample application
appVersion: "1.0"
```

* `version`: Helm chart's version.
* `appVersion`: Application's version.
* `name`: Release/chart name.

---

### **values.yaml**

```
replicaCount: 3

image:
  repository: <ecr-url>/myapp
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```

**Purpose:** Central config used by template files.

---

### **templates/deployment.yaml** (templated example)

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
```

**Key functions:**

* `{{ .Values.* }}` injects values from values.yaml.
* `{{ .Chart.Name }}` references chart name.
* `{{ .Release.Name }}` references release instance.

---

## **• Common Issues / Errors**

### **1. values.yaml file not found**

Incorrect or missing chart structure.

### **2. Template rendering errors**

Typos in template functions or missing required values.

### **3. Failing pre/post hooks**

Improperly configured Helm hooks causing deployments to hang.

### **4. Chart version issues**

Forgetting to bump chart version leads to confusion in environments.

### **5. Rollbacks not restoring configs**

ConfigMaps and Secrets may not roll back cleanly.

---

## **• Troubleshooting / Fixes**

* Use `helm lint` to identify structural or syntax issues.
* Validate rendered manifests:

  ```
  helm template myapp .
  ```
* Maintain separate environment-specific values files.
* Clean up stuck hooks using:

  ```
  kubectl delete job <job-name>
  ```
* Ensure chart version increments for each deployment.

---

## **• Best Practices / Tips**

* Maintain one chart per service.
* Prefer values files for environment-specific overrides.
* Use semantic versioning for chart versions.
* Always run `helm lint` in CI before deployments.
* Use `helm upgrade --install` for idempotent operations.
* Avoid embedding sensitive data in charts—use external secret managers.
* Use `--set image.tag` in CI instead of modifying values.yaml file.

---
---

# ArgoCD & GitOps — Detailed Explanation Version

## **• Concept / What**

GitOps is a deployment methodology where **Git acts as the single source of truth** for the Kubernetes desired state. All Kubernetes manifests, Helm charts, and configuration files are stored in Git.

ArgoCD is a **pull-based, declarative CD tool** installed inside a Kubernetes cluster (e.g., EKS). It continuously watches a Git repository and automatically deploys the state defined there to the cluster. Instead of CI/CD tools running `kubectl apply` or `helm upgrade`, ArgoCD handles the deployment lifecycle based on Git changes.

---

## **• Why / Purpose / Use Case in Real-World**

* Eliminates push-based deployments done via Jenkins/GitHub Actions.
* Ensures cluster state always matches what is stored in Git.
* Provides controlled, auditable, PR-based approval workflows.
* Simplifies multi-environment handling (dev, QA, stage, prod) using Git branches or folders.
* Offers a visual UI to manage sync status, health, and rollbacks.
* Provides **self-healing**: any drift (manual changes) is automatically corrected.
* Removes the need to manually run Helm or kubectl commands.
* Supports both Helm charts and raw Kubernetes manifests.

**Real scenarios:**

* PR merges drive environment promotion.
* CI builds Docker images but does not deploy.
* ArgoCD auto-syncs deployments to EKS clusters based on Git updates.
* Companies maintain separate repos for application code and deployment configuration.

---

## **• How It Works / Steps / Syntax**

### **1. Install ArgoCD into EKS**

ArgoCD runs as a set of Kubernetes deployments inside `argocd` namespace:

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This creates components such as:

* argocd-server (UI)
* argocd-repo-server
* argocd-application-controller
* dex-server
* redis

### **2. Access ArgoCD UI**

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Fetch initial admin password:

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### **3. ArgoCD watches Git repos for deployment definitions**

You define an ArgoCD Application which tells ArgoCD:

* Which repo to read
* Which branch
* Which path (Helm chart or raw YAML)
* Target cluster & namespace
* Whether to auto-sync

### **4. Deployment happens automatically when Git updates**

When a PR is merged into the watched branch:

* ArgoCD detects new commit
* Renders manifests (raw YAML or Helm templates)
* Applies them to the cluster
* Ensures cluster matches Git’s desired state

### **5. No kubectl/helm commands needed**

ArgoCD internally handles:

* Apply
* Replace
* Patch
* Prune
* Sync
* Diff
* Rollbacks

---

## **• ArgoCD Application Manifest (Explained)**

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/infra-config.git
    path: helm/myapp
    targetRevision: dev
    helm:
      valueFiles:
        - values-dev.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Explanation:**

* `source.repoURL`: Git repo containing manifests or Helm charts.
* `path`: Folder containing Kubernetes definitions.
* `targetRevision`: Branch ArgoCD watches.
* `helm.valueFiles`: Values file for that environment.
* `destination.namespace`: Where to deploy.
* `syncPolicy.automated`: Auto-sync on Git change.
* `prune`: Remove old resources not in Git.
* `selfHeal`: Fix drift if someone manually changes the cluster.

---

## **• CI vs GitOps (Workflow Difference)**

### **CI (Jenkins/GitHub Actions):**

* Runs on PR creation.
* Builds Docker image.
* Pushes image to registry.
* Updates image tag inside Git manifest/values file.

### **GitOps via ArgoCD:**

* Deploys ONLY after PR is approved & merged.
* Watches branches (dev/qa/stage/prod).
* Promotes using Git merges.
* Ensures deployments match Git exactly.

**Important:**
ArgoCD does NOT build images. CI still handles image creation.

---

## **• Approval Workflow in GitOps**

Approvals move from Jenkins → Git.
ArgoCD does not provide "approval" settings. Instead:

* PR approval acts as deployment approval.
* Protected branches ensure only authorized merges.
* After merge, ArgoCD auto-syncs and deploys.

**Stages example:**

* dev → watches `dev` branch
* qa → watches `qa` branch
* stage → watches `stage` branch
* prod → watches `prod` branch

Promotion is done by merging PRs between branches.

---

## **• Helm in GitOps**

* Helm charts are stored **directly in Git**, not packaged.
* No need to upload `.tgz` files to S3 or chartmuseum.
* ArgoCD renders Helm templates internally.
* Keep Kubernetes manifests inside `templates/` folder.
* Use multiple values files like:

  * `values-dev.yaml`
  * `values-qa.yaml`
  * `values-prod.yaml`

**Advantage:** One set of templates, multiple environments.
Raw manifest approach requires separate YAML files per environment.

---

## **• Common Issues / Errors**

### **1. OutOfSync state**

Cluster does not match Git.

### **2. Git authentication issues**

ArgoCD cannot clone private repos.

### **3. RBAC permission failures**

ArgoCD service account lacks rights to create/update resources.

### **4. Helm templating errors**

Wrong values or incorrect template logic.

### **5. Drift caused by manual kubectl edits**

ArgoCD detects and corrects drift if self-heal is enabled.

---

## **• Troubleshooting / Fixes**

* Use `argocd app diff` to compare Git vs cluster.
* Use `argocd app sync` for manual sync.
* Validate Helm charts with:

  ```
  helm lint
  helm template
  ```
* Fix RBAC permissions using Role/RoleBinding.
* Validate Git repo structure.

---

## **• Best Practices / Tips**

* Install ArgoCD **inside the EKS cluster**; no separate nodes required.
* Dedicated nodes are needed only for heavy tools (Prometheus/Grafana), not ArgoCD.
* Use separate repos for application code and infrastructure (GitOps) repos.
* Prefer Helm for multi-environment DRY configuration.
* Use PR-based approvals for deployment governance.
* Enable auto-sync + prune + self-heal for true GitOps operation.
* Avoid manual `kubectl` or `helm` deployments.
* Use Git branches to manage dev → qa → stage → prod promotions.

---
---

# Environment Promotion (dev → qa → stage → prod) — Detailed Explanation Version

## **• Concept / What**

Environment promotion is the controlled movement of an application version through multiple environments such as **dev**, **qa**, **stage**, and **prod**. Each environment acts as a validation checkpoint. Promotion ensures that only tested and approved builds progress toward production.

In traditional CI/CD systems (e.g., Jenkins), promotion is push-based. In GitOps with ArgoCD, promotion is pull-based and driven by Git merges.

---

## **• Why / Purpose / Use Case in Real-World**

* Prevents unstable code from reaching production.
* Allows QA, managers, and stakeholders to approve releases.
* Ensures proper validation at each level (functional, integration, user acceptance).
* Maintains clean DevOps workflows with clear separation of responsibilities.
* Provides traceability: every promotion is tied to a Git commit/PR.
* Supports safer, auditable deployments.

**Typical reasons for multiple environments:**

* **dev:** developer validation
* **qa:** testing and verification
* **stage:** production-like testing, final checks
* **prod:** live user traffic

---

## **• How It Works / Steps / Syntax**

### **1. CI Builds and Pushes Image**

* Developer raises PR.
* CI builds the Docker image.
* CI pushes the image to ECR.
* CI updates the manifest or Helm values in Git (image tag).

### **2. Environment Promotion via Jenkins (Traditional Push-Based)**

* User selects environment using parameterized build.
* Jenkins deploys using `kubectl apply` or `helm upgrade`.
* Approval gates pause deployment for QA/manager signoff.
* Pipeline pushes changes directly into the target environment.

### **3. Environment Promotion via GitOps (Pull-Based)**

Promotion happens through Git:

* dev → qa → stage → prod via PR merges.
* ArgoCD watches separate branches or folders for each environment.
* When PR is approved and merged, ArgoCD automatically deploys the change.
* No manual kubectl or helm commands needed.

### **4. Branch-Based Promotion (Most Common GitOps Pattern)**

```
feature → dev → qa → stage → prod
```

Each branch is tied to an ArgoCD Application.

### **5. Folder-Based Promotion**

```
environments/
  dev/
  qa/
  stage/
  prod/
```

ArgoCD apps watch these folders.

### **6. Image Tag Promotion (Simple Approach)**

Each environment has a values file:

* `values-dev.yaml`
* `values-qa.yaml`
* `values-stage.yaml`
* `values-prod.yaml`

Promotion updates the image tag in the appropriate file.

---

## **• Example ArgoCD Application (Per Environment)**

### **Dev Environment**

```
source:
  repoURL: https://github.com/org/infra.git
  path: helm/myapp
  targetRevision: dev

destination:
  namespace: dev
```

### **Stage Environment**

```
source:
  repoURL: https://github.com/org/infra.git
  path: helm/myapp
  targetRevision: stage

destination:
  namespace: stage
```

### **Prod Environment**

```
targetRevision: prod
destination:
  namespace: prod
```

---

## **• Common Issues / Errors**

### **1. Wrong branch watched by ArgoCD**

ArgoCD deploys unexpected configuration.

### **2. CI updates wrong values file**

Causes incorrect deployments.

### **3. Missing PR approvals**

Unreviewed changes promoted accidentally.

### **4. Environment folders not aligned**

Inconsistent structure creates confusion.

### **5. Image tag mismatch**

CI forgets to push updated tag to Git.

---

## **• Troubleshooting / Fixes**

* Verify ArgoCD targetRevision (branch).
* Validate that CI updated the correct manifest.
* Use `argocd app diff` to compare Git vs cluster.
* Use ArgoCD UI to check sync errors.
* Configure strict Git branch protections.
* Ensure CI writes to the correct environment folder or values file.

---

## **• Best Practices / Tips**

* Use **PR approvals** as the main promotion mechanism.
* Protect QA, stage, and prod branches.
* Do not allow direct commits to stage or prod.
* Prefer Helm charts for multi-environment DRY configuration.
* Keep ArgoCD auto-sync ON for dev/qa but OFF or manual-sync for prod (optional based on company policy).
* Maintain consistent folder structures across environments.
* Always tag images uniquely and update manifests accordingly.

---
---

# Blue-Green & Canary Deployment Automation — Detailed Explanation Version

## **• Concept / What**

**Blue-Green Deployment** uses two parallel environments — *Blue* (current live version) and *Green* (new version). Only one receives production traffic at a time. Switching traffic from Blue → Green is done by updating the Service selector or load balancer routing.

**Canary Deployment** gradually releases a new version to a small portion of users before exposing it to everyone. Traffic is usually split using an ingress controller, service mesh, or Argo Rollouts.

---

## **• Why / Purpose / Use Case in Real-World**

### **Blue-Green:**

* Zero-downtime releases.
* Quick rollback by switching traffic back to Blue.
* Ability to test Green in production environment without exposing users.
* Very simple to implement with native Kubernetes (no CRDs required).

### **Canary:**

* Safely test new versions with a small percentage of real traffic.
* Advanced control for high-risk applications.
* Enables metric-based validation and automatic rollback.
* Used in large-scale systems (payments, streaming, retail, telecom).

---

## **• How It Works / Steps / Syntax**

# **1. Blue-Green Deployment (Works with plain Kubernetes)**

### **How companies do this in real workflows:**

1. **Deploy Blue version (current)**
2. **Deploy Green version (new)** using `kubectl apply` or `helm upgrade`.
3. **Service initially points to Blue** using a selector:

```
selector:
  app: myapp
  version: blue
```

4. Test Green by hitting it internally (port-forward/cluster DNS).
5. **Switch traffic to Green** by changing the selector.
6. If issues occur → switch back instantly to Blue.

### **Real production command to switch traffic:**

```
kubectl patch svc myapp -n prod \
  -p '{"spec": {"selector": {"app": "myapp", "version": "green"}}}'
```

### **Rollback command:**

```
kubectl patch svc myapp -n prod \
  -p '{"spec": {"selector": {"app": "myapp", "version": "blue"}}}'
```

### **How Blue-Green works in GitOps (ArgoCD):**

* Update the Service selector in Git.
* ArgoCD auto-syncs and switches the traffic.
* Rollback is done by reverting the PR.

### **Helm/GitOps example (values file):**

```
activeVersion: green
```

Switching Blue → Green = updating one line in Git.

---

# **2. Canary Deployment (Requires CRD or traffic router)**

### Why it needs extra tools:

Kubernetes **cannot split traffic by percentage** natively. A Service load balances equally across pods.

So real-world Canary uses:

* **Argo Rollouts (CRD)**
* **Istio / Linkerd / AWS App Mesh**
* **NGINX Ingress Canary annotations**

### Typical Canary rollout sequence:

1. Deploy new version.
2. Send 5% traffic to v2.
3. Monitor metrics/logs/errors.
4. Increase to 20%, then 50%, then 100%.
5. Rollback automatically on failure.

### Real Argo Rollouts example:

```
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 30}
      - setWeight: 50
      - pause: {}
      - setWeight: 100
  selector:
    matchLabels:
      app: myapp
```

This performs progressive delivery with traffic splitting.

### With Istio VirtualService:

```
traffic:
- destination: myapp-v1
  weight: 90
- destination: myapp-v2
  weight: 10
```

---

## **• Common Issues / Errors**

### **Blue-Green**

* Incorrect Service selector causing wrong version to receive traffic.
* Forgetting to clean up old environment.
* Green version not tested before switching.
* Ingress/cache delays causing mixed responses.

### **Canary**

* Traffic weights not applied correctly.
* Misconfigured Ingress or service mesh.
* Rollout stuck due to missing pause/analysis steps.
* Automated rollback failing due to incorrect metric thresholds.
* NGINX canary annotations applied incorrectly.

---

## **• Troubleshooting / Fixes**

### **Blue-Green**

* Check Service selector using:

```
kubectl get svc myapp -o yaml
```

* Ensure Deployment labels match selector.
* Verify both Blue and Green pods are healthy before switching.
* Use ArgoCD UI to confirm sync.

### **Canary**

* Check rollout status:

```
kubectl argo rollouts get rollout myapp
```

* Validate traffic routing in Istio/Ingress.
* Check AnalysisTemplate logs for threshold failures.
* Ensure metrics provider (Prometheus/Datadog) is reachable.
* Confirm CRDs are installed correctly for Rollouts.

---

## **• Best Practices / Tips**

### **Blue-Green**

* Keep Blue pods running until Green is fully validated.
* Automate traffic switching through CI or GitOps.
* Use separate namespaces for Blue & Green in large setups.
* Run smoke tests on Green before activating.

### **Canary**

* Always start with small traffic percentages (1–5%).
* Integrate with Prometheus for automated rollback.
* Use Argo Rollouts dashboard for visibility.
* Pause rollout at key percentages (20%, 50%).
* Store rollout YAMLs in Git for reproducibility.
* Avoid canary for stateful workloads unless mesh is strong.

---
---
---


