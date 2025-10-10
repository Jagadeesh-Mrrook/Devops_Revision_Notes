# Deployment and Environment Considerations

## Detailed Explanation Version

### Concept / What:

Deployment and environment considerations define how software is moved from development into production, and how different environments are structured to support development, testing, and stable releases. The goal is to create controlled spaces for code to evolve safely and predictably.

### Why / Purpose / Use Case:

* **Separation of concerns:** Developers work in Dev without impacting users.
* **Risk reduction:** Bugs in Dev or Staging don’t affect Production.
* **Faster feedback loops:** Issues are detected earlier.
* **Environment parity:** Ensures behavior in Production can be reliably tested in Staging/UAT.

### How it Works / Steps / Flow:

Three main environments are commonly used:

```
Dev Environment
  └─ Developers code & run initial unit tests
QA Environment
  └─ QA engineers perform functional, penetration, and integration tests
Staging / UAT Environment
  └─ Mirrors production; clients and QA perform User Acceptance Testing
Production Environment
  └─ Live environment for end-users; monitored for uptime and performance
```

* **Environment Isolation:** Separate resources or access per environment.
* **Configuration Management:** Environment-specific configs via `.env` files, ConfigMaps (K8s), Parameter Store (AWS).
* **Automation:** Deployments automated through CI/CD pipelines to reduce human error.

### Cloud / Kubernetes Examples:

* **AWS:**

  * Dev: Smaller EC2 instances, test S3 buckets, snapshot RDS.
  * QA: Larger resources than Dev, performing full internal testing.
  * Staging/UAT: Mirrors Production resources; CloudWatch monitoring.
  * Production: Full-scale resources with auto-scaling and monitoring.

* **Kubernetes:**

  * Namespaces to isolate Dev, QA, Staging, Prod.
  * Separate deployments, services, and config maps per namespace.

### Common Issues / Errors:

* Dev code accidentally deployed to Production.
* Environment-specific bugs not caught due to differences between environments.
* Hard-coded configuration values causing failures.

### Troubleshooting / Fixes:

* Use automated CI/CD gates to prevent wrong deployments.
* Maintain separate resource identifiers per environment.
* Test environment parity regularly using smoke tests.

### Best Practices / Tips:

* Use Infrastructure as Code (Terraform, CloudFormation) for reliable environment provisioning.
* Automate environment setup to prevent configuration drift.
* Keep secrets separate per environment.
* Monitor all environments, not just Production.
* Ensure Staging/UAT closely mimics Production, but scale resources cost-effectively.

---
---

# CI/CD Pipelines for Front-End and Back-End

## Concept / What:

CI/CD pipelines automate the build, test, and deployment process for applications. For microservice-based architectures, front-end and back-end applications often have separate pipelines to ensure independent, efficient deployments.

* **CI (Continuous Integration):** Automates code build, testing, and artifact creation.
* **CD (Continuous Deployment):** Automates deployment to the target environment (Dev, QA, UAT, Prod).

---

## Why / Purpose / Use Case:

* **Separation of front-end and back-end pipelines** avoids resource conflicts and reduces failures.
* Supports **multiple microservices** with independent deployments.
* Provides **control and configurability** for different environments.
* Real-world use cases:

  * Front-end may be static (S3 + CloudFront) or dynamic (containerized in EKS).
  * Back-end services are microservices deployed to Kubernetes clusters.
* Each microservice/repository can manage its own **CI/CD workflow**, reducing interdependencies.

---

## How it Works / Steps / Examples:

### 1. Repository and Branching Strategy

* Each microservice (back-end) or application (front-end) should have **separate GitHub repositories**.
* Each repo follows a **feature branching strategy**:

  * `main` → production-ready code
  * `development` → active development branch
  * `feature/*` → individual developer features
  * `qa` → branch for QA testing
  * `uat` → branch for UAT/Staging testing
* Pull requests (PRs) are raised from feature branches to development → QA → UAT → main.

### 2. Microservice Repository Structure

For each microservice, the GitHub repository contains:

```
microservice-name/
├── src/                   # Application source code
├── k8s/                   # Kubernetes manifests (Deployment, Service, ConfigMap, Secrets)
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── terraform/             # Microservice-specific infrastructure code
│   ├── dev/
│   ├── qa/
│   ├── uat/
│   └── prod/
├── Jenkinsfile.ci         # CI pipeline
├── Jenkinsfile.cd         # CD pipeline
└── Dockerfile
```

* **Explanation:**

  * `src/` → Microservice code.
  * `k8s/` → Kubernetes manifests applied via CI/CD pipeline.
  * `terraform/` → Microservice-specific infra like IAM roles, DB schema, secrets.
  * `Jenkinsfile.ci` → Build, test, and push Docker images.
  * `Jenkinsfile.cd` → Deploy to specific environment (Dev, QA, UAT, Prod).

### 3. CI/CD Pipelines for Back-End Microservices

**CI Pipeline Steps:**

1. Triggered by PR or merge to development/feature branch.
2. Build microservice code.
3. Run unit and integration tests.
4. Run code analysis (e.g., SonarQube).
5. Build Docker image.
6. Push image to container registry (e.g., AWS ECR).

**CD Pipeline Steps:**

1. Triggered after successful CI or manually for environment deployment.
2. Deploy to target Kubernetes environment.
3. Apply Kubernetes manifests using `kubectl` or Helm.
4. Perform smoke tests and health checks.
5. Optionally update service version for rollback capability.

### 4. CI/CD Pipelines for Front-End Applications

**Front-End Deployment Types:**

* **Static Front-End:** Served via S3 + CloudFront → no Kubernetes deployment required.
* **Dynamic Front-End:** Containerized and deployed on EKS → similar CI/CD flow as back-end.

**Front-End Repository Structure Example:**

```
frontend-app/
├── src/                   # Front-end source code
├── k8s/                   # Optional if containerized
├── Jenkinsfile.ci
├── Jenkinsfile.cd
└── build/                 # Built static assets
```

**Clarifications & Real-World Practices:**

* Static front-end does not require microservice-style deployment.
* Dynamic front-end can be containerized but usually relies on backend APIs.
* CI/CD separation ensures independent testing and deployment from back-end services.
* Shared infra (like EKS cluster, node groups) can be reused across multiple microservices.
* Each microservice repo has its own CI/CD pipelines and Terraform code if necessary.

---

## Common Issues / Errors:

* CI/CD pipeline failure due to dependency issues between front-end and back-end.
* Docker image build failures.
* Kubernetes manifest misconfiguration.
* Incorrect environment selection in pipeline parameters.

---

## Troubleshooting / Fixes:

* Ensure pipeline triggers and environment variables are correctly configured.
* Validate Docker builds locally before pushing.
* Use `kubectl apply --dry-run` for manifests testing.
* Separate front-end and back-end repositories for independent CI/CD runs.
* Use pipeline concurrency limits to manage multiple microservice builds.

---

## Best Practices / Tips:

* Maintain separate GitHub repositories per microservice.
* Each microservice repo should have two Jenkinsfiles: CI and CD.
* Terraform strictly for infrastructure; Kubernetes manifests handled by CI/CD pipelines.
* Use versioned Docker images for rollback.
* Feature branching strategy to manage development and testing.
* Reuse shared infrastructure resources (EKS cluster, node groups) across multiple microservices.

---
---

# Containerized Deployment (Docker/Kubernetes) - Notes

## Concept / What

Containerized deployment is the practice of packaging an application along with its dependencies, configuration, and runtime environment into a container.

* **Docker**: Tool for building, running, and managing containers.
* **Kubernetes (K8s)**: Orchestrator for managing containers at scale, handling deployment, scaling, networking, and resilience.

> Docker is like the box you pack your app in, Kubernetes is like the warehouse manager who organizes and runs all the boxes efficiently.

---

## Why / Purpose / Use Case

1. **Environment Consistency**

   * Works the same in dev, QA, staging, or production.
   * Eliminates "it works on my machine" problems.

2. **Scalability**

   * Kubernetes can auto-scale containers based on load.
   * Useful for microservices architecture.

3. **Isolation**

   * Each container has its own environment, dependencies, and runtime.

4. **Resource Efficiency**

   * Containers share OS kernel → lighter than VMs.
   * Faster to start and stop.

5. **CI/CD Integration**

   * Docker images built in CI, stored in registry, deployed in CD.

**Example**: Each microservice (product, order, payment) in an e-commerce app runs in its own container.

---

## How it Works / Steps / Flow

### 1. Docker Workflow

1. **Write Dockerfile**

```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "start"]
```

2. **Build Docker Image**

```bash
docker build -t product-service:1.0 .
```

3. **Push to Registry**

```bash
docker tag product-service:1.0 <account>.dkr.ecr.us-east-1.amazonaws.com/product-service:1.0
docker push <account>.dkr.ecr.us-east-1.amazonaws.com/product-service:1.0
```

4. **Run Container**

```bash
docker run -p 3000:3000 product-service:1.0
```

### 2. Kubernetes Workflow

**Components**: Pod, Deployment, Service, ConfigMap/Secret

1. **Write Deployment & Service YAML**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product
  template:
    metadata:
      labels:
        app: product
    spec:
      containers:
      - name: product
        image: <account>.dkr.ecr.us-east-1.amazonaws.com/product-service:1.0
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  type: ClusterIP
  selector:
    app: product
  ports:
  - port: 80
    targetPort: 3000
```

2. **Apply YAML**

```bash
kubectl apply -f product-deployment.yaml
kubectl apply -f product-service.yaml
```

3. **Scale Pods**

```bash
kubectl scale deployment product-service --replicas=5
```

4. **Rolling Update / Rollback**

```bash
kubectl set image deployment/product-service product=product-service:1.1
kubectl rollout undo deployment/product-service
```

---

## Real-World Example (AWS + K8s)

* **Frontend**: Static content in S3 + CloudFront; dynamic dev environment → Docker container in EKS
* **Backend Microservices**: Each microservice has Dockerfile, Deployment & Service YAML, CI/CD pipelines (build image → push to ECR → deploy via Helm/K8s)
* **Shared Infrastructure**: EKS cluster, VPC, IAM roles, RDS, S3 managed via Terraform; microservice-specific resources in service repo if needed

---

## Evolution: VM → Docker → Kubernetes

| Stage           | Description                       | Pros                                            | Cons                                                |
| --------------- | --------------------------------- | ----------------------------------------------- | --------------------------------------------------- |
| Traditional VMs | Apps deployed directly on EC2/VMs | Simple                                          | Dependency conflicts, scaling manual, hard rollback |
| Docker          | Apps in containers                | Lightweight, isolated, consistent               | No orchestration, manual scaling                    |
| Kubernetes      | Containers orchestrated           | Auto-scaling, rolling updates, production-ready | Complexity, learning curve                          |

**Key Insight:** Docker is great for development or small setups; Kubernetes is the standard for production-grade microservices.

---

## Trade-offs / Challenges

| Advantage               | Trade-off / Challenge                                           |
| ----------------------- | --------------------------------------------------------------- |
| Isolation & Consistency | Requires container orchestration expertise                      |
| Easy Scaling            | Infrastructure complexity (EKS, networking, storage)            |
| Rapid Deployment        | Multiple pipelines per microservice                             |
| Environment Agnostic    | Debugging in containers can be tricky                           |
| Rollbacks & Versioning  | Image tagging and deployment versions must be managed carefully |

---

## Best Practices / Tips

1. One Dockerfile per microservice
2. Tag Docker images properly; avoid `latest` in prod
3. Use Helm charts to manage deployments & configs
4. Separate Dev/Staging/Prod clusters
5. Keep Kubernetes YAMLs microservice-specific
6. Monitor & log (CloudWatch, Prometheus, Grafana)
7. Manage secrets using Kubernetes Secrets or AWS Secrets Manager

---
---

# Versioning and Rollback Strategies (Detailed Notes)

## Concept / What

* **Versioning:** Assigning unique identifiers to each build, release, or artifact of an application/microservice.
* **Rollback:** Reverting an application or deployment to a previous stable version when a deployment fails.
* Goal: Controlled, traceable releases and rapid recovery from failures.

## Why / Purpose / Use Case

* **Controlled Releases:** Know exactly which version is deployed in dev, QA, staging, or production.
* **Safe Deployments:** Minimize downtime if a new version fails.
* **Audit & Compliance:** Track changes per build.
* **Microservices:** Each service can deploy independently; avoids version conflicts.

## How it Works / Steps / Design

### Versioning Methods

* **Semantic Versioning (SemVer):** `MAJOR.MINOR.PATCH` (e.g., `v2.3.1`)

  * MAJOR: breaking changes
  * MINOR: backward-compatible features
  * PATCH: bug fixes
* **Git Tags / Commit SHA:** Tag each release in Git; Docker images can use Git SHA or version tag.

### CI/CD Pipeline Integration

1. **CI Pipeline (Jenkins):**

   * Build artifact → run tests → tag artifact → push to artifact registry (ECR, JFrog, Nexus)
   * Jenkins stores **VERSION_TAG** and metadata in build history.

2. **CD Pipeline (Jenkins):**

   * Deploy artifact to environment using **VERSION_TAG**.
   * For rollback, fetch the **last successful build’s version** from Jenkins.

### Rollback Strategies

* **Manual Rollback:**

  * Select previous successful Jenkins build → redeploy artifact.
* **Automated Rollback:**

  * Jenkins pipeline detects failure → dynamically fetch previous successful build’s artifact version → redeploy.
* **Blue-Green Deployment:**

  * Two environments (blue & green); switch traffic only after validation.
  * Rollback = switch traffic back to old environment.
* **Canary Deployment:**

  * Gradual rollout; monitor metrics.
  * Rollback = redirect traffic back to stable version.

### Example: Jenkins-based Rollback

```groovy
pipeline {
  parameters {
    string(name: 'VERSION_TAG', defaultValue: '', description: 'Artifact version to deploy')
  }
  stages {
    stage('Deploy') {
      steps {
        script {
          def version = params.VERSION_TAG
          if(version == '') {
            // Fetch latest successful build version
            def lastStableBuild = currentBuild.previousSuccessfulBuild
            version = lastStableBuild.getEnvironment()['VERSION_TAG']
          }
          sh "kubectl set image deployment/payment-service payment-service=<registry>/payment-service:${version}"
        }
      }
    }
  }
}
```

### Real-World Examples (AWS + K8s)

* **Microservice:** `payment-service`

  * Current version: `v2.3.1`
  * New version: `v2.4.0`
* **CI:** Build Docker image, push to ECR
* **CD:** Deploy via Jenkins to Kubernetes
* **Rollback:** Jenkins/CD uses **last stable build version** to redeploy if new version fails
* **Frontend Static Content:** Versioned S3 buckets or CloudFront objects; rollback points to previous bucket version

## Common Issues / Errors

* Deploying untested version → production failures
* Conflicting dependencies between microservices
* Losing track of artifact versions if not properly tagged
* Jenkins pipeline fails due to missing previous build references

## Troubleshooting / Fixes

* Always tag artifacts uniquely (Git SHA + semantic version)
* Maintain build metadata and environment variables in Jenkins
* Keep 2–3 previous stable versions in registry for rollback
* Automate rollback with pipeline scripts referencing **previous successful build**
* Monitor deployments using metrics and logs (CloudWatch, Prometheus)

## Best Practices / Tips

1. Use semantic versioning consistently across all microservices
2. Docker images should have both **version tag** and **Git SHA**
3. Maintain at least 2–3 previous stable builds for rollback
4. Implement **Blue-Green or Canary deployments** for production
5. Jenkins pipelines should store version info and handle rollback dynamically
6. Automate rollback triggers based on **post-build success/failure**
7. Maintain a changelog for each version/release

---
---
---

