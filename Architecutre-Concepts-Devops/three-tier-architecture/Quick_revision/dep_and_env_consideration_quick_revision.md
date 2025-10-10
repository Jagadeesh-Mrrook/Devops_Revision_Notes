# Deployment and Environment Considerations - Quick Revision

### Concept / What:

Controlled movement of code through Dev, QA, Staging/UAT, and Production environments.

### Why / Purpose:

* Separate Dev, QA, and Prod to reduce risk.
* Detect issues early.
* Ensure production-like testing.

### How / Flow:

* **Dev:** Development & unit testing.
* **QA:** Functional, penetration, integration tests.
* **Staging/UAT:** Mirrors Prod; client testing and UAT.
* **Production:** Live environment.

### Cloud/K8s Examples:

* **AWS:** Dev (small resources), QA (larger), Staging/UAT (Prod-like), Production (full scale).
* **K8s:** Namespaces for each environment, separate deployments & config maps.

### Common Issues:

* Accidental Prod deployment.
* Environment-specific bugs.
* Hard-coded configs.

### Fixes:

* CI/CD gates.
* Separate resource identifiers.
* Smoke tests for parity.

### Best Practices:

* IaC for environment provisioning.
* Automate setup.
* Separate secrets.
* Monitor all environments.
* Staging/UAT should mimic Prod but scale cost-effectively.

---
---

# Quick Revision: CI/CD Pipelines for Front-End and Back-End

## Concept:

* CI/CD automates build, test, and deployment.
* Separate pipelines for front-end and back-end microservices.

## Purpose:

* Independent deployments for microservices.
* Environment isolation (Dev, QA, UAT, Prod).
* Automation reduces errors and enables rollback.

## Repository & Branching:

* **Separate GitHub repo per microservice**.
* Feature branching: `main` → `development` → `feature/*` → `qa` → `uat` → `main`

## Microservice Repo Structure:

```
microservice-name/
├── src/              # Code
├── k8s/              # Deployment, Service, ConfigMap
├── terraform/        # Microservice-specific infra
├── Jenkinsfile.ci    # CI
├── Jenkinsfile.cd    # CD
└── Dockerfile
```

## CI Pipeline:

1. Trigger on PR/merge
2. Build code
3. Run unit/integration tests
4. Code analysis (SonarQube)
5. Build Docker image
6. Push to registry (ECR/DockerHub)

## CD Pipeline:

1. Trigger after CI success or manually
2. Deploy to environment (Dev/QA/UAT/Prod)
3. Apply Kubernetes manifests
4. Smoke tests & health checks
5. Version updates for rollback

## Front-End Specific:

* **Static** → S3 + CloudFront, no Kubernetes.
* **Dynamic** → Containerized, deploy to EKS like back-end.
* Separate CI/CD pipelines per front-end repository.

## Best Practices:

* Separate repo per microservice.
* Two Jenkinsfiles per repo (CI + CD).
* Terraform strictly for infrastructure.
* Versioned Docker images for rollback.
* Reuse shared infra (EKS clusters, VPCs) across microservices.

---
---

# Quick Revision - Containerized Deployment (Docker/Kubernetes)

## What

* Containers package app + dependencies + runtime.
* Docker: builds and runs containers.
* Kubernetes: orchestrates containers at scale.

## Why

* Environment consistency (dev, QA, prod)
* Scalability & auto-scaling
* Isolation between microservices
* Resource-efficient
* CI/CD integration

## Docker Workflow

1. Write Dockerfile
2. Build image: `docker build -t service:1.0 .`
3. Push to registry: `docker push <registry>`
4. Run container: `docker run -p 3000:3000 service:1.0`

## Kubernetes Workflow

* Components: Pod, Deployment, Service, ConfigMap/Secret
* Apply Deployment & Service YAML
* Scale pods: `kubectl scale deployment/service --replicas=N`
* Rolling updates & rollback: `kubectl set image`, `kubectl rollout undo`

## Evolution

* VMs → Docker → Kubernetes
* Kubernetes preferred for production microservices

## Trade-offs

* * Isolation & consistency, easy scaling, rapid deployment
* * Complexity, multiple pipelines, debugging, version management

## Best Practices

1. One Dockerfile per microservice
2. Proper image tagging
3. Use Helm charts
4. Separate clusters for dev/staging/prod
5. Microservice-specific Kubernetes YAMLs
6. Monitor & log metrics
7. Manage secrets securely

---
---

# Versioning & Rollback Strategies (Quick Revision)

## Concept

* Versioning = Unique identifier for each build/artifact
* Rollback = Revert to previous stable version if deployment fails

## Purpose

* Controlled releases
* Safe deployments & minimal downtime
* Track microservice versions

## How it Works

* **CI Pipeline:** Build artifact, tag version, push to registry
* **CD Pipeline:** Deploy using `VERSION_TAG`
* **Rollback:** Fetch last successful build version from Jenkins

## Jenkins Rollback Example

```groovy
def lastStableBuild = currentBuild.previousSuccessfulBuild
def rollbackVersion = lastStableBuild.getEnvironment()['VERSION_TAG']
sh "kubectl set image deployment/payment-service payment-service=<registry>/payment-service:${rollbackVersion}"
```

## Best Practices

* Semantic versioning + Git SHA for Docker images
* Keep previous 2–3 stable builds
* Automate rollback using pipeline
* Optional Blue-Green / Canary deployments

---
---
---


