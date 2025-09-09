# DevOps CI/CD and Infrastructure Notes

---

## 1. Tools & Deployment Locations (VD-SRC style)

| Tool/Service   | Location/Instance         | Notes                                                 |
| -------------- | ------------------------- | ----------------------------------------------------- |
| Jenkins Master | EC2                       | Manages CI/CD jobs and triggers pipelines             |
| Jenkins Agents | EC2                       | Runs CI/CD pipelines (build, test, deploy, Terraform) |
| Terraform      | Jenkins Agent             | Runs infra provisioning via pipelines                 |
| Helm           | Jenkins Agent / EKS nodes | Deploys Kubernetes resources                          |
| Docker         | Jenkins Agent             | Builds and manages container images                   |
| Maven          | Jenkins Agent             | Builds Java applications                              |
| Ansible        | EC2                       | Optional, used for configuration management           |
| SonarQube      | EC2                       | Static code analysis for application code             |
| Prometheus     | EC2                       | Metrics collection                                    |
| Grafana        | EC2                       | Dashboard for metrics visualization                   |
| Nexus/JFrog    | EC2                       | Artifact repository                                   |
| Git            | Local laptops             | Code version control; developers push code            |
| EKS Cluster    | EC2 nodes                 | Runs containerized applications                       |

---

### DevOps Tools per Environment

| Tool / Service         | Dev      | QA  | UAT | Prod |
| ---------------------- | -------- | --- | --- | ---- |
| **Jenkins**            | Yes      | No  | No  | Yes  |
| **CI Jenkinsfile**     | Yes      | Yes | Yes | Yes  |
| **CD Jenkinsfile**     | Yes      | Yes | Yes | Yes  |
| **Docker**             | Yes      | No  | No  | Yes  |
| **Maven**              | Yes      | No  | No  | Yes  |
| **Helm**               | Yes      | No  | No  | Yes  |
| **Ansible**            | Yes      | No  | No  | Yes  |
| **Terraform**          | Yes      | No  | No  | Yes  |
| **SonarQube**          | Yes      | No  | No  | No   |
| **Prometheus**         | Yes      | No  | No  | Yes  |
| **Grafana**            | Yes      | No  | No  | Yes  |
| **Loki**               | Yes      | No  | No  | Yes  |
| **Kubernetes/EKS**     | Yes      | Yes | Yes | Yes  |
| **Cluster Autoscaler** | Optional | No  | No  | Yes  |


## 2. Jenkins CI/CD File Management & Directory Structure

### **Application Repo Structure**

```
app-repo/
├── src/                   # Source code
├── tests/                 # Unit/Integration tests
├── Dockerfile             # Container build
├── Jenkinsfile-ci         # CI pipeline
└── Jenkinsfile-cd         # CD pipeline
```

### **CI Job**

* Jenkinsfile: `Jenkinsfile-ci`
* Trigger: PR or commit (webhook / SCM polling)
* Tasks: Build, test, static analysis, push artifact

### **CD Job**

* Jenkinsfile: `Jenkinsfile-cd`
* Trigger: Manual (parameterized)
* Parameters: ENV (dev/staging/prod), ARTIFACT\_VERSION
* Tasks: Deploy artifact, optionally run Terraform stages

### **Jenkins Job Setup Notes**

* Each job is **created manually** (or via Job DSL / JCasC for large orgs)
* Job configuration specifies:

  * GitHub repository URL
  * Branch (main/develop)
  * Jenkinsfile path (`Jenkinsfile-ci` or `Jenkinsfile-cd`)
* CI runs automatically on PRs/commits; CD is manual with parameters

---

## 3. Terraform Infrastructure Repo Directory Structure

```
infra-repo/
├── modules/                  # Reusable Terraform modules (VPC, EKS, RDS)
├── dev/                      # Dev environment Terraform configs
├── staging/                  # Staging environment Terraform configs
├── prod/                     # Prod environment Terraform configs
├── backend.tf                # Remote state configuration (S3 / Terraform Cloud)
├── main.tf                   # Terraform root configuration
├── Jenkinsfile-infra         # Jenkins pipeline for Terraform infra provisioning
└── Jenkinsfile-app-deploy    # Optional: App deployment if needed
```

### **Feature Branching Strategy**

* Main branch: production-ready code
* Feature branches: created per DevOps engineer
* PR workflow: Review → Approval → Merge into main
* Jenkins pipeline can trigger **on PR creation** for validation/plan/security scan
* Deployment to environments handled by **parameterized pipeline**; no separate QA/UAT branches

### **Terraform State & Security**

* State file stored in remote backend (S3/Terraform Cloud) to prevent conflicts
* Security scans: TFSEC, Checkov, TeraScan (optional, installed on Jenkins agent)

---

## 4. Summary / Key Points

| Topic            | Key Points                                                                                                        |
| ---------------- | ----------------------------------------------------------------------------------------------------------------- |
| Jenkins CI/CD    | Separate jobs for CI/CD; explicit Jenkinsfile path; CI auto-trigger, CD manual/parameterized                      |
| Application repo | Contains source code, tests, Dockerfile, Jenkinsfile-ci, Jenkinsfile-cd                                           |
| Infra repo       | Contains Terraform configs, modules, Jenkinsfile-infra; feature branch workflow; main branch production-ready     |
| Terraform        | Remote state backend; optional security scans; environment selection via parameters/folder structure              |
| Tool Deployment  | Most tools run on EC2; Jenkins agents run pipelines; Git on developer laptops                                     |
| Branching        | App: feature → develop → main; Infra: feature → main, no QA/UAT branches; pipelines handle environment deployment |

---

**Interview-ready phrasing:**

> “In most organizations, Jenkins jobs are manually created for CI and CD pipelines. CI jobs are triggered automatically by PRs or commits, while CD jobs are parameterized and triggered manually. Application and infrastructure code are maintained in separate repositories. Terraform state files are stored in a remote backend to prevent conflicts, and optional security scans are integrated. Feature branching strategy is followed for both repos to ensure safe collaboration, with pipelines controlling environment deployments.”


---

# DevOps Project Setup for Interview (Nykaa Project)

## Dev Environment (8 EC2 + 1 RDS)

| EC2 / RDS | Purpose / Tools                                                         | Microservices                                  |
| --------- | ----------------------------------------------------------------------- | ---------------------------------------------- |
| 3 EC2     | Jenkins (1 Master + 2 Agents) → Terraform, Ansible, Docker, Helm, Maven |                                                |
| 2 EC2     | Kubernetes Worker Nodes → run pods                                      | Notification Service, Order Management Service |
| 1 EC2     | SonarQube → code quality & static analysis                              |                                                |
| 1 EC2     | Observability → Prometheus + Grafana + Loki                             |                                                |
| 1 EC2     | Bastion Host → SSH access, admin tasks                                  |                                                |
| 1 RDS     | MySQL/Postgres → Dev database for microservices                         |                                                |

## QA Environment (2 EC2 + 1 RDS)

| EC2 / RDS | Purpose / Tools                    | Microservices                                  |
| --------- | ---------------------------------- | ---------------------------------------------- |
| 2 EC2     | Kubernetes Worker Nodes → run pods | Notification Service, Order Management Service |
| 1 RDS     | QA database (MySQL/Postgres)       |                                                |

> Uses Dev Jenkins, SonarQube, and Observability for CI/CD and monitoring

## UAT Environment (1 EC2 + 1 RDS)

| EC2 / RDS | Purpose / Tools                   | Microservices                                  |
| --------- | --------------------------------- | ---------------------------------------------- |
| 1 EC2     | Kubernetes Worker Node → run pods | Notification Service, Order Management Service |
| 1 RDS     | UAT database                      |                                                |

> Uses Dev Jenkins, SonarQube, and Observability

## Prod Environment (12 EC2 + 2 RDS)

| EC2 / RDS | Purpose / Tools                                                                               | Microservices                                  |
| --------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| 4 EC2     | Kubernetes Worker Nodes → run pods (behind ALB, in ASG)                                       | Notification Service, Order Management Service |
| 2 EC2     | Jenkins (1 Master + 1 Agent) → deploy Prod workloads, Terraform, Ansible, Docker, Helm, Maven |                                                |
| 2 EC2     | Observability → Prometheus + Grafana + Loki                                                   |                                                |
| 2 EC2     | Bastion Hosts → admin access, troubleshooting                                                 |                                                |
| 2 EC2     | Reserved/Spare (optional)                                                                     |                                                |
| 2 RDS     | Prod DB (primary + standby for HA)                                                            |                                                |

## Notes

* Each environment has its own **RDS instance** for isolation.
* **Microservices:** Notification Service + Order Management Service only.
* Non-prod environments use **Dev Jenkins, SonarQube, and Observability**.
* Prod is fully dedicated with HA-ready architecture.
* Worker nodes scale via **Kubernetes + Auto Scaling Groups** in Prod.
* Architecture is interview-ready for explaining responsibilities and infrastructure.

---

| Tool/Service             | Non-Prod (Dev/QA/UAT) Instance | Prod Instance              | Reasoning                      |
| ------------------------ | ------------------------------ | -------------------------- | ------------------------------ |
| **Jenkins Master**       | m5.large (2 vCPU, 8 GB)        | m5.xlarge (4 vCPU, 16 GB)  | Orchestration only.            |
| **Jenkins Agents**       | c5.xlarge (4 vCPU, 8 GB)       | r5.2xlarge (8 vCPU, 64 GB) | Build/test heavy.              |
| **SonarQube**            | r5.large (2 vCPU, 16 GB)       | r5.xlarge (4 vCPU, 32 GB)  | Memory intensive.              |
| **Prometheus + Grafana** | m5.large (2 vCPU, 8 GB)        | m5.xlarge (4 vCPU, 16 GB)  | Combined in one box.           |
| **Loki**                 | i3.large (2 vCPU, 15 GB, NVMe) | i3.xlarge (4 vCPU, 30 GB)  | Storage-heavy logs.            |
| **Nexus/Artifactory**    | r5.large (2 vCPU, 16 GB)       | r5.xlarge (4 vCPU, 32 GB)  | Artifact storage.              |
| **Bastion Host**         | t3.micro (1 vCPU, 1 GB)        | t3.micro (1 vCPU, 1 GB)    | Lightweight jump box.          |
| **EKS Worker Nodes**     | m5.large / c5.large mix        | m5.xlarge (4 vCPU, 16 GB)  | Non-Prod smaller, Prod bigger. |

