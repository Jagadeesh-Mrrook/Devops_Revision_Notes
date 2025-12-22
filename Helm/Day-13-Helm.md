# Deploy Microservices to EKS using Helm

## Concept / What

Deploying microservices to an Amazon EKS cluster using Helm means packaging Kubernetes manifests (Deployment, Service, Ingress, etc.) into reusable Helm charts, where **each microservice has its own chart** and is deployed independently as a Helm release.

Helm acts as the **package manager for Kubernetes**, handling installation, upgrades, rollbacks, and configuration management for microservices running on EKS.

---

## Why / Purpose / Real Use Case

In real-world DevOps setups:

* Applications are split into **multiple microservices** (auth, user, payment, order, etc.).
* Each microservice evolves independently and must be deployed separately.
* Kubernetes YAML becomes hard to manage across **dev / qa / prod** environments.

Helm solves this by:

* Providing **one chart per microservice**
* Allowing **environment-specific configuration** using separate values files
* Enabling **safe upgrades, rollbacks, and version control**
* Integrating cleanly with **CI/CD pipelines (Jenkins, GitHub Actions)**

Without Helm, managing raw YAML for dozens of services across environments becomes error-prone and unscalable.

---

## How it Works / Steps / Syntax

### 1. One Helm Chart per Microservice

Each microservice has its own chart and lifecycle.

```
auth-service/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-qa.yaml
├── values-prod.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
```

This ensures:

* Independent deployments
* Independent scaling
* Independent rollback

---

### 2. values.yaml (Environment-Agnostic Defaults)

`values.yaml` contains **default configuration** for the microservice.

```yaml
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/auth-service
  tag: latest

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  host: auth.example.com
```

---

### 3. Environment-Specific Values Files

Each environment overrides only what changes.

**values-dev.yaml**

```yaml
replicaCount: 1
image:
  tag: dev
```

**values-prod.yaml**

```yaml
replicaCount: 4
image:
  tag: prod
```

This allows the **same chart** to be reused across environments.

---

### 4. Deployment Template (templates/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

Helm replaces values during rendering based on the selected values file.

---

### 5. Service Template (templates/service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
```

---

### 6. Ingress Template (templates/ingress.yaml)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}
                port:
                  number: {{ .Values.service.port }}
```

---

### 7. Helm Install / Upgrade per Environment

```bash
helm upgrade --install auth-dev ./auth-service \
  -f values-dev.yaml
```

```bash
helm upgrade --install auth-prod ./auth-service \
  -f values-prod.yaml
```

Helm tracks releases separately per environment.

---

## Common Issues / Errors

* Hardcoding environment values inside templates
* Using one chart for all microservices
* Missing environment-specific values files
* Incorrect image tags deployed to prod
* Ingress conflicts between services

---

## Troubleshooting / Fixes

* Use `helm template` to preview rendered YAML
* Use `helm diff` before upgrades
* Validate values using `required` function
* Keep environment overrides minimal
* Verify release-specific resources using `kubectl get all -l app=<release>`

---

## Best Practices

* One Helm chart per microservice
* One values file per environment
* Never hardcode environment-specific data in templates
* Use Helm for deployment logic, not business logic
* Store charts and values in Git
* Always test upgrades in lower environments before prod

---
---

# Configure Ingress with ALB Controller using Helm (EKS)

## Concept / What

Configuring Ingress with the **AWS ALB Ingress Controller (AWS Load Balancer Controller)** using Helm means:

* Installing the **Ingress Controller** in the EKS cluster via Helm
* Defining **Ingress resources** in Helm charts for microservices
* Using **Ingress annotations** so the controller dynamically creates and manages an **Application Load Balancer (ALB)** in AWS

Helm is used for **installing the controller** and **templating Ingress manifests**, not for directly creating the ALB.

---

## Why / Purpose / Real Use Case

In real-world EKS deployments:

* External traffic must reach multiple microservices
* Each microservice should not expose a separate LoadBalancer Service
* Routing should be **host-based or path-based**
* ALB should be managed automatically

Using ALB Ingress Controller with Helm:

* Avoids manual ALB creation
* Centralizes traffic routing
* Supports SSL, path routing, and scaling
* Keeps Kubernetes-native traffic management

---

## How it Works / Steps / Syntax

### 1. Install AWS Load Balancer Controller using Helm

The controller is deployed **once per cluster**.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

This controller:

* Watches Ingress resources
* Reads ALB-related annotations
* Creates / updates ALB automatically

---

### 2. Ingress Template in Helm Chart (Microservice)

Each microservice chart defines its own Ingress template.

**templates/ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
    alb.ingress.kubernetes.io/group.name: main-alb
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}
                port:
                  number: {{ .Values.service.port }}
```

---

### 3. values.yaml for Ingress Configuration

```yaml
ingress:
  enabled: true
  host: auth.example.com
  path: /

service:
  port: 8080
```

Ingress behavior is fully controlled via values, not hardcoded.

---

### 4. How ALB Gets Created (Flow)

1. Helm installs the Ingress resource
2. ALB Controller detects the Ingress
3. Controller reads annotations
4. ALB is created in AWS automatically
5. Target groups are mapped to pods
6. Listener rules are configured

---

### 5. Multiple Microservices → One ALB

Using the same annotation:

```yaml
alb.ingress.kubernetes.io/group.name: main-alb
```

Results in:

* One ALB
* Multiple routing rules
* Different hosts or paths per service

---

### 6. Rendered Kubernetes YAML (Simplified)

```yaml
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
  - host: auth.example.com
    http:
      paths:
      - backend:
          service:
            name: auth-prod
            port:
              number: 8080
```

---

## Common Issues / Errors

* Missing ingress.class annotation
* ALB not created due to IAM permission issues
* Incorrect service port mapping
* Multiple ALBs created unintentionally
* Ingress not picked up by controller

---

## Troubleshooting / Fixes

* Check controller logs in kube-system namespace
* Validate IAM role permissions
* Use `kubectl describe ingress` to inspect events
* Ensure correct ingress.class value
* Verify service selector and port

---

## Best Practices

* Install ALB Controller once per cluster
* Use Helm for controller installation
* Template Ingress resources per microservice
* Use group.name annotation for shared ALB
* Keep ingress annotations configurable via values.yaml
* Avoid creating Service type LoadBalancer when using ALB

---
---

# Blue-Green & Canary Deployments using Helm Values

## Concept / What

Blue-Green and Canary deployments using Helm mean **controlling traffic and rollout behavior by changing values.yaml**, without rewriting Kubernetes manifests.

Helm does **not implement deployment strategies itself**. Instead, it:

* Templates Kubernetes resources (Deployments, Services, Ingress)
* Uses **values.yaml** to decide which version is active, how many replicas run, and how traffic is routed

The strategy is implemented by Kubernetes + Ingress, while Helm acts as the **configuration switch**.

---

## Why / Purpose / Real Use Case

In real-world production systems:

* Deployments must avoid downtime
* New versions must be tested safely
* Rollbacks must be fast

Blue-Green and Canary deployments help:

* Reduce risk during releases
* Validate new versions with real traffic
* Perform instant rollback

Helm is ideal because:

* Version switching becomes a **values change**
* CI/CD pipelines can automate rollout decisions
* Same chart works for all strategies

---

## How it Works / Steps / Syntax

### 1. Blue-Green Deployment using Helm Values

Two versions run in parallel:

* Blue = current production
* Green = new version

Traffic switches via Service or Ingress.

#### values.yaml

```yaml
activeColor: blue

blue:
  imageTag: v1
  replicaCount: 3

green:
  imageTag: v2
  replicaCount: 0
```

---

#### Deployment Template (templates/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Values.activeColor }}
spec:
  replicas: {{ index .Values .Values.activeColor "replicaCount" }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
      color: {{ .Values.activeColor }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
        color: {{ .Values.activeColor }}
    spec:
      containers:
      - name: app
        image: myapp:{{ index .Values .Values.activeColor "imageTag" }}
```

---

#### Service Template (templates/service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  selector:
    app: {{ .Release.Name }}
    color: {{ .Values.activeColor }}
  ports:
  - port: 80
    targetPort: 8080
```

Switching traffic:

```yaml
activeColor: green
```

Helm upgrade → instant traffic switch.

---

### 2. Canary Deployment using Helm Values

Canary runs alongside stable with fewer replicas or limited traffic.

#### values.yaml

```yaml
stable:
  imageTag: v1
  replicaCount: 4

canary:
  imageTag: v2
  replicaCount: 1
```

---

#### Deployment Template (Simplified)

```yaml
metadata:
  name: {{ .Release.Name }}-canary
spec:
  replicas: {{ .Values.canary.replicaCount }}
```

Ingress or Service Mesh handles traffic split.

---

### 3. Helm Upgrade Flow

```bash
helm upgrade myapp ./chart -f values-prod.yaml
```

Only values change — templates remain the same.

---

## Common Issues / Errors

* Hardcoding version names in templates
* Forgetting labels in Service selectors
* Running both versions at full scale accidentally
* No quick rollback strategy

---

## Troubleshooting / Fixes

* Use `helm template` before upgrading
* Validate selectors match pod labels
* Start canary with minimal replicas
* Keep previous values file for rollback

---

## Best Practices

* Control deployment strategy via values.yaml only
* Keep templates generic and reusable
* Automate strategy switching in CI/CD
* Always test canary before full rollout
* Use Helm rollback for instant recovery

---
---

# Zero-Downtime Upgrade using Helm

## Concept / What

A zero-downtime upgrade using Helm means **upgrading an application running on Kubernetes without interrupting live traffic**, by relying on Kubernetes rolling update behavior while Helm manages versioned releases and upgrades.

Helm does not implement zero-downtime logic itself. It updates Kubernetes manifests, and Kubernetes ensures pods are replaced safely.

---

## Why / Purpose / Real Use Case

In real-world production environments:

* Applications must stay available during deployments
* Users should not see errors or downtime
* Releases happen frequently through CI/CD pipelines

Zero-downtime upgrades are critical for:

* Microservices exposed via Services or Ingress
* Production EKS clusters
* Continuous delivery pipelines

Helm enables this by:

* Managing upgrades as controlled releases
* Allowing safe rollback if something fails

---

## How it Works / Steps / Syntax

### 1. Kubernetes Rolling Update Strategy

Helm upgrades trigger Kubernetes rolling updates defined in the Deployment.

**templates/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

This ensures:

* A new pod is created before an old pod is terminated
* At least one healthy pod always serves traffic

---

### 2. Stable Service During Upgrade

**templates/service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: 80
      targetPort: 8080
```

The Service:

* Is created once
* Is never recreated during upgrades
* Automatically routes traffic to ready pods

---

### 3. Helm Upgrade Flow

```bash
helm upgrade myapp ./chart -f values-prod.yaml
```

Upgrade sequence:

1. Helm updates the Deployment manifest
2. Kubernetes starts new pods with the new image
3. Readiness probes pass
4. Traffic shifts to new pods
5. Old pods are terminated

---

### 4. Rendered YAML (Key Part)

```yaml
rollingUpdate:
  maxUnavailable: 0
  maxSurge: 1
```

This configuration is the core of zero-downtime behavior.

---

## Common Issues / Errors

* Missing or incorrect readiness probes
* Running only 1 replica in production
* Incorrect rollingUpdate values
* Image pull failures during upgrade

---

## Troubleshooting / Fixes

* Always configure readiness probes
* Use minimum 2 replicas in production
* Run `helm upgrade --dry-run` before deploying
* Monitor rollout using `kubectl rollout status deployment/<name>`

---

## Best Practices

* Use rolling updates for standard production upgrades
* Keep Service selectors stable
* Avoid deleting or recreating Services
* Use Helm rollback for fast recovery
* Test upgrades in lower environments first

---
---

# Manage Multiple Microservices using Helm

## Concept / What

Managing multiple microservices using Helm means **deploying, upgrading, and operating each microservice independently using its own Helm chart and its own Helm release**.

Each microservice:

* Has its **own chart structure**
* Is deployed with a **separate release name**
* Can be upgraded, rolled back, or scaled without affecting others

Helm acts as the standard deployment mechanism across all services.

---

## Why / Purpose / Real Use Case

In real-world systems:

* Applications are split into many microservices
* Each service has a different lifecycle, team, and release frequency
* A failure in one service must not impact others

Helm helps by:

* Enforcing **clear boundaries per microservice**
* Allowing **independent CI/CD pipelines** per service
* Reducing blast radius during deployments
* Keeping deployments consistent across environments

Without this approach, deployments become tightly coupled and risky.

---

## How it Works / Steps / Syntax

### 1. One Helm Chart per Microservice

Each microservice owns its chart.

```
repo/
├── auth-service/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── payment-service/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── order-service/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
```

This ensures isolation and clarity.

---

### 2. Independent Helm Releases

Each chart is installed with its own release name.

```bash
helm install auth-dev ./auth-service
helm install payment-dev ./payment-service
helm install order-dev ./order-service
```

Each release:

* Is tracked independently
* Has its own revision history
* Can be upgraded or rolled back separately

---

### 3. Environment Management via values.yaml

Each microservice uses environment-specific values files.

```
values-dev.yaml
values-qa.yaml
values-prod.yaml
```

Example:

```bash
helm upgrade auth-prod ./auth-service -f values-prod.yaml
```

The same chart works across all environments.

---

### 4. Shared vs Service-Specific Components

Typical pattern:

* **Deployment, Service, HPA** → per microservice
* **Ingress, ConfigMaps** → either per service or shared (based on design)

Helm allows both patterns without coupling services.

---

### 5. Operational Visibility

Helm provides clear visibility:

```bash
helm list
helm history auth-prod
helm rollback auth-prod 2
```

Operations teams can manage each service cleanly.

---

## Common Issues / Errors

* Using one chart for multiple unrelated services
* Sharing release names across services
* Mixing environment values inside templates
* Tight coupling between microservice deployments

---

## Troubleshooting / Fixes

* Enforce one-chart-per-service rule
* Standardize chart structure across teams
* Validate values files per environment
* Use consistent naming conventions

---

## Best Practices

* One Helm chart per microservice
* One Helm release per microservice per environment
* Independent CI/CD pipelines per service
* Consistent chart conventions across repositories
* Avoid cross-service dependencies in charts

---
---

# Helm in Jenkins Pipelines (CI/CD Integration)

## Concept / What

Using Helm in Jenkins pipelines means **separating build (CI) and deployment (CD) responsibilities**, where:

* **CI pipeline** builds Docker images, tags them, and pushes them to Amazon ECR
* **CD pipeline** deploys or upgrades applications on EKS using Helm, consuming the image tag produced by CI

Helm is used **only in the CD stage** to deploy Kubernetes manifests with the correct image version.

---

## Why / Purpose / Real Use Case

In real-world DevOps setups:

* CI and CD are often **separate Jenkins pipelines** or Jenkinsfiles
* Builds happen frequently; deployments are controlled
* Image versions must be traceable and reproducible

This separation:

* Avoids rebuilding images during deployment
* Enables promotion of the *same image* across environments (dev → qa → prod)
* Keeps deployments predictable and auditable

---

## How it Works / Steps / Syntax

### 1. CI Pipeline – Build, Tag, Push (Only Once)

**Responsibility:** Produce the Docker image and decide the tag.

Common tagging strategies:

* Jenkins build number (`BUILD_NUMBER`)
* Git commit SHA
* Semantic version (`1.2.3`)

Example CI steps:

```bash
docker build -t myapp:${BUILD_NUMBER} .
docker tag myapp:${BUILD_NUMBER} <account>.dkr.ecr.<region>.amazonaws.com/myapp:${BUILD_NUMBER}
docker push <account>.dkr.ecr.<region>.amazonaws.com/myapp:${BUILD_NUMBER}
```

At this point:

* Image is immutable
* Stored in ECR
* Ready to be deployed

---

### 2. Passing Image Tag from CI to CD (Key Problem)

Since CI and CD are separate, the **image tag must be transferred**.

There are three real-world patterns.

---

### Method 1: Export Image Tag as Jenkins Artifact / Parameter (Automated)

#### CI Pipeline

Save the tag into a file:

```bash
echo "IMAGE_TAG=${BUILD_NUMBER}" > image.env
```

Archive it:

```groovy
archiveArtifacts artifacts: 'image.env'
```

---

#### CD Pipeline

Retrieve artifact and load variable:

```groovy
copyArtifacts(projectName: 'ci-pipeline-job', filter: 'image.env')
load 'image.env'
```

Use it in Helm:

```bash
helm upgrade myapp ./chart \
  --set image.tag=${IMAGE_TAG}
```

✅ Fully automated
✅ No manual tag entry

---

### Method 2: GitOps Style – Update Helm Values in Git (Best Practice)

CI pipeline updates the image tag in Helm values stored in Git.

Example:

```yaml
image:
  repository: <ecr-repo>
  tag: "123"
```

CI commits the change:

```bash
sed -i "s/tag:.*/tag: ${BUILD_NUMBER}/" values-dev.yaml
git commit -am "Update image tag to ${BUILD_NUMBER}"
git push
```

CD pipeline triggers automatically from Git change.

✅ No manual input
✅ Full audit trail
✅ Preferred in GitOps setups

---

### Method 3: Manual Tag Input in CD (Least Preferred)

CD pipeline requires a parameter:

```groovy
string(name: 'IMAGE_TAG', description: 'ECR image tag to deploy')
```

Used in Helm:

```bash
helm upgrade myapp ./chart --set image.tag=${IMAGE_TAG}
```

❌ Manual
❌ Error-prone
❌ Not scalable

---

## Why CI Should Own Image Tagging

* CI has access to build metadata
* CI ensures image immutability
* CD should never rebuild or retag images

CD should only **consume a tested image**.

---

## Common Issues / Errors

* Rebuilding images in CD pipeline
* Using `latest` tag
* Manually copying tags between pipelines
* Deploying different images to different environments unintentionally

---

## Troubleshooting / Fixes

* Verify image exists in ECR before Helm deploy
* Use `helm template` to confirm rendered image tag
* Lock deployments to immutable tags
* Ensure CI → CD handoff is automated

---

## Best Practices

* CI builds and tags images once
* CD deploys the same image using Helm
* Prefer GitOps-style tag updates
* Never use `latest` in production
* Keep CI/CD responsibilities clearly separated

---
---
---
