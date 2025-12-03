# Docker in CI/CD — Building Docker Images in Jenkins (Detailed Explanation Notes)

## **1. Concept / What**

Building Docker images in CI (Jenkins) means automating the entire image creation process:

* Pull source code from SCM (GitHub/GitLab/Bitbucket)
* Build Docker image using a Dockerfile
* Tag the image (commit/build/tag)
* Push the image to a container registry (ECR/Hub)
* Make the image available for Kubernetes/EKS deployments

Jenkins becomes the **centralized Docker image builder**, ensuring consistent, reproducible builds.

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why DevOps teams do this:**

* Ensures **consistent and reproducible builds**
* Automatically builds images on every commit or PR
* Required for **EKS deployments** (K8s uses image tags to pull versions)
* Enables **automated CD pipelines** using Jenkins + kubectl/Helm
* Allows **versioning, rollback, and audit tracking**
* Ensures images are **stored in ECR**, not in local developer machines
* Facilitates **image scanning** and security checks during CI

### **Used in real-world scenarios like:**

* Microservices deployments to EKS
* Blue/Green, rolling deployments
* Multi-environment pipelines (dev → qa → prod)
* Automated releases based on Git tags or branches

---

## **3. How It Works / Steps / Syntax**

### **Step 1 — Jenkins pulls code from SCM**

```groovy
git branch: 'main', url: 'https://github.com/org/myapp.git'
```

---

### **Step 2 — Jenkins builds the Docker image**

Jenkins agent must have Docker installed.

#### **Build command:**

```sh
docker build -t myapp:BUILD_NUMBER .
```

#### **Pipeline snippet:**

```groovy
stage('Build Docker Image') {
    steps {
        script {
            IMAGE="myapp"
            TAG="${env.BUILD_NUMBER}"

            sh """
                docker build -t ${IMAGE}:${TAG} .
            """
        }
    }
}
```

---

### **Step 3 — Tagging the image properly**

Use unique tags to avoid Kubernetes using stale images.

Examples:

* `${BUILD_NUMBER}`
* `${GIT_COMMIT}`
* `feature-branch-name`
* `v1.4.0`

Tag example:

```sh
docker tag myapp:42 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:42
```

---

### **Step 4 — Login to ECR (AWS recommended method)**

```sh
aws ecr get-login-password --region ap-south-1 \
   | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

---

### **Step 5 — Push image to ECR**

```sh
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:42
```

---

### **Full Jenkins Example Pipeline (Real-World)**

```groovy
pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        REPO = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp"
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/jagga/myapp.git', branch: 'main'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    TAG = env.BUILD_NUMBER
                    sh "docker build -t myapp:${TAG} ."
                }
            }
        }

        stage('Login to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${AWS_REGION} \
                    | docker login --username AWS --password-stdin ${REPO}
                """
            }
        }

        stage('Push Image') {
            steps {
                script {
                    sh """
                    docker tag myapp:${TAG} ${REPO}:${TAG}
                    docker push ${REPO}:${TAG}
                    """
                }
            }
        }
    }
}
```

---

## **4. Common Issues / Errors**

### ❌ Jenkins agent does not have Docker installed

Error:

```
docker: command not found
```

### ❌ Jenkins inside Kubernetes cannot use Docker

Docker daemon isn’t available → must use Kaniko/Buildah.

### ❌ Incorrect Dockerfile COPY paths

```
COPY failed: file not found
```

Usually due to wrong context.

### ❌ Wrong image tags cause K8s to use old images

Kubernetes Deployment keeps pulling previous versions.

### ❌ ECR login failures

```
no basic auth credentials
```

Happens when Jenkins doesn’t authenticate properly.

### ❌ Build context too large

Sending massive context → slow CI builds.

---

## **5. Troubleshooting / Fixes**

### ✔ Install Docker on Jenkins agent

Amazon Linux:

```sh
sudo yum install docker -y
sudo systemctl start docker
sudo usermod -aG docker jenkins
```

### ✔ Use proper Dockerfile paths

```sh
docker build -f docker/prod.Dockerfile .
```

### ✔ For Jenkins running inside Kubernetes

Use Kaniko:

```sh
kaniko --context . --dockerfile Dockerfile --destination "$REPO:$TAG"
```

### ✔ Use unique tags

Fix image caching issues in Kubernetes.

### ✔ Clean unused local Docker artifacts

```sh
docker system prune -af
```

---

## **6. Best Practices / Tips (EKS-Focused)**

### ✔ Use unique, immutable tags

Avoid using **latest** in K8s.

### ✔ Build small, optimized images

Multi-stage builds → faster CI.

### ✔ Scan images immediately after building

For vulnerabilities.

### ✔ Keep credentials in Jenkins Credentials Store

Never hardcode AWS keys.

### ✔ Keep Dockerfile consistent across environments

Dev → QA → Prod using same Dockerfile.

### ✔ Use Git-based triggers

Push → Automatically build new Docker image.

### ✔ Store images in ECR for all deployments

Standard in AWS environments.

---
---

# Docker Layer Caching in CI/CD (Detailed Explanation Notes)

## **1. Concept / What**

Docker layer caching is a mechanism where Docker reuses previously built image layers instead of rebuilding them each time. Each Dockerfile instruction creates a layer, and if the instruction and its inputs haven’t changed, Docker will reuse that layer.

However, **traditional Docker layer caching works only on persistent Jenkins agents (VMs)**. It does **not work on Kubernetes-based Jenkins agents**, because pods are ephemeral and Docker cache is lost on every build.

To use caching in Kubernetes-based CI, alternative builders like **BuildKit, Kaniko, or Buildah** with remote cache must be used.

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why caching matters in CI/CD:**

* **Speeds up Docker builds** by avoiding re-running dependency installs (`npm install`, `mvn package`, etc.)
* **Reduces compute cost** on Jenkins agents
* **Faster feedback loop** for developers
* **Makes builds more predictable** when Dockerfile is structured correctly

### **Where DevOps teams use caching:**

* Microservices builds in Jenkins
* Multi-stage builds for production images
* Large dependency installations
* Build-heavy frameworks (Java Spring Boot, Angular, React)

---

## **3. How It Works / Steps / Syntax**

### **Basic layer caching logic**

A layer is reused when:

1. The Dockerfile instruction hasn't changed
2. The content used by that instruction hasn't changed
3. All previous layers were cache hits

Example:

```
FROM node:18-alpine        → cache hit
WORKDIR /app               → cache hit
COPY package*.json .       → cache hit
RUN npm install            → cache hit
COPY . .                   → cache miss (source code changed)
```

Only the last layer rebuilds.

---

## **Dockerfile structure for better caching**

### ❌ **Slow (bad caching)**

```
COPY . .
RUN npm install
```

Any source code change invalidates caching of dependencies.

### ✔️ **Fast (optimized caching)**

```
COPY package*.json .
RUN npm install
COPY . .
```

Dependencies rarely change → cache reused → faster builds.

---

## **4. Caching in Jenkins CI**

### **Case 1: Jenkins on VM (EC2, bare-metal)**

✔ Full Docker layer caching works
✔ Cache persists under `/var/lib/docker`
✔ Builds become progressively faster

### **Case 2: Jenkins agents on Kubernetes**

❌ **Traditional Docker caching does NOT work** because:

* Pods are ephemeral
* No persistent Docker daemon
* `/var/lib/docker` does not persist across builds
* Many Jenkins K8s agents don’t run Docker at all

### ✔ Supported caching alternatives in Kubernetes:

1. **BuildKit with registry cache**

```sh
docker buildx build \
  --cache-from=type=registry,ref=$ECR/myapp:cache \
  --cache-to=type=registry,ref=$ECR/myapp:cache,mode=max \
  -t $ECR/myapp:latest .
```

2. **Kaniko caching**

```sh
--cache=true --cache-repo=$ECR/myapp-cache
```

3. **Buildah with PVC**
   PersistentVolume is mounted to store container layers.

---

## **5. Common Issues / Errors**

### ❌ Cache not used

* Dockerfile layer order inefficient
* Large build context invalidates layers
* Jenkins agent cleans workspace
* Running inside Kubernetes pod without persistent storage

### ❌ Extremely slow builds

* Re-running dependency installs on every build
* Not using `.dockerignore`
* No registry-based caching (K8s agents)

### ❌ Full rebuild triggered unintentionally

* Modification in `package.json`, `requirements.txt`, or `pom.xml`
* Changing base image tag
* COPY command bringing unnecessary files

---

## **6. Troubleshooting / Fixes**

### ✔ Reorder Dockerfile for optimal caching

Put stable layers on top, code on bottom.

### ✔ Use `.dockerignore`

```
node_modules
.git
logs
dist
target
tmp
```

### ✔ Use BuildKit or Kaniko in Kubernetes CI

Avoid Docker-in-Docker (DinD) because caching is not persistent.

### ✔ Persistent caching with BuildKit registry cache

Stores cache in ECR or Docker Hub.

### ✔ Reduce build context

Avoid copying entire directory unnecessarily.

---

## **7. Best Practices / Tips (EKS-Focused)**

### ✔ Use multi-stage builds

Smaller images, better cache utilization.

### ✔ Enable BuildKit everywhere

```
export DOCKER_BUILDKIT=1
```

### ✔ Do not rely on Docker caching in Kubernetes

Always use registry-based caching.

### ✔ Keep dependency files independent

`package.json`, `requirements.txt`, `pom.xml` should not change frequently.

### ✔ Keep Dockerfile simple and predictable

Consistent structure improves caching effectiveness.

---
---

# Docker Login Using Credentials (AWS ECR Only) — Detailed Explanation Notes

## **1. Concept / What**

Docker login using credentials refers to authenticating Jenkins (or any CI/CD tool) with a container registry so it can **push** and **pull** Docker images. For AWS-based DevOps workflows, this means logging in to **Amazon Elastic Container Registry (ECR)**.

In CI/CD, this authentication happens automatically through **AWS CLI**, IAM roles, or Jenkins-stored credentials.

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why DevOps teams must authenticate to ECR:**

* Required to **push built Docker images** from Jenkins into ECR.
* Kubernetes/EKS must pull images from ECR during deployments.
* Prevents unauthorized access to private registries.
* Ensures automated CI/CD pipelines can run without manual login.
* Supports environment-specific repos: dev, qa, prod.
* Enables image scanning and version tracking in ECR.

### **Where it is used:**

* Jenkins → Build image → Login to ECR → Push image.
* EKS → Pull image from ECR → Deploy pods.
* Blue/green and rolling deployments.

---

## **3. How It Works / Steps / Syntax (ECR Only)**

### **Step 1 — Login to ECR using AWS CLI**

AWS provides a short-lived token for Docker authentication.

```sh
aws ecr get-login-password --region ap-south-1 \
 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

### Explanation

* `get-login-password` generates a temporary auth token.
* Jenkins pipes the token into Docker login.
* No password stored → secure.
* Valid for 12 hours.

---

### **Step 2 — Build Docker Image in Jenkins**

```sh
docker build -t myapp:${BUILD_NUMBER} .
```

---

### **Step 3 — Tag Image for ECR**

```sh
docker tag myapp:${BUILD_NUMBER} 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:${BUILD_NUMBER}
```

---

### **Step 4 — Push Image to ECR**

```sh
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:${BUILD_NUMBER}
```

---

### **Full Jenkins Pipeline Example (ECR Only)**

```groovy
pipeline {
    agent any

    environment {
        REGION = "ap-south-1"
        REPO = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp"
    }

    stages {
        stage('Login to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${REGION} \
                    | docker login --username AWS --password-stdin ${REPO}
                """
            }
        }

        stage('Build Image') {
            steps {
                sh "docker build -t myapp:${BUILD_NUMBER} ."
            }
        }

        stage('Push Image') {
            steps {
                sh """
                docker tag myapp:${BUILD_NUMBER} ${REPO}:${BUILD_NUMBER}
                docker push ${REPO}:${BUILD_NUMBER}
                """
            }
        }
    }
}
```

---

## **4. Common Issues / Errors (ECR Focused)**

### ❌ **"no basic auth credentials"**

Meaning:

* Jenkins could not authenticate to ECR.
* Login stage skipped or failed.

### ❌ AWS CLI not installed on Jenkins agent

```
aws: command not found
```

### ❌ IAM user/role missing ECR permissions

Required permissions:

* `ecr:GetAuthorizationToken`
* `ecr:BatchCheckLayerAvailability`
* `ecr:InitiateLayerUpload`
* `ecr:PutImage`

### ❌ Wrong region used during login

ECR repo region must match login region.

### ❌ EKS cannot pull image (ImagePullBackOff)

Caused by:

* Image not pushed
* Wrong tag in Deployment YAML
* ECR permissions missing on node IAM role

---

## **5. Troubleshooting / Fixes**

### ✔ Fix AWS region mismatch

```sh
aws ecr get-login-password --region ap-south-1
```

Must match ECR repository region.

### ✔ Attach correct IAM permissions

Use AWS-managed policy:

```
AmazonEC2ContainerRegistryPowerUser
```

### ✔ Install AWS CLI on Jenkins agents

Required for ECR login.

### ✔ Validate ECR login

```sh
docker info
```

Should show registry credentials.

### ✔ Verify image exists in ECR

```sh
aws ecr describe-images --repository-name myapp
```

---

## **6. Best Practices / Tips (ECR + EKS)**

### ✔ Always use AWS CLI login instead of storing passwords

More secure and token-based.

### ✔ Use IAM role-based authentication for Jenkins EC2 agents

Avoid storing AWS Access Keys.

### ✔ Store unique image tags for each build

Makes rollback and deployments reliable.

### ✔ Separate ECR repos for each environment (dev, qa, prod)

Cleaner isolation and access control.

### ✔ Validate ECR login before building image

Fail early when credentials are wrong.

---
---
# Docker Tagging Strategy (Commit / Branch / Semantic) — Detailed Explanation Notes

## **1. Concept / What**

A Docker tagging strategy defines how images are named and versioned when they are built in CI/CD pipelines. Tags act as identifiers that help Kubernetes/EKS pull the correct image version during deployments.

Common tag types:

* **Build number tags** (e.g., `myapp:42`)
* **Commit SHA tags** (e.g., `myapp:9f2c4a1`)
* **Branch tags** (e.g., `myapp:dev`, `myapp:qa`)
* **Semantic version tags** (e.g., `myapp:v1.4.2`)

A good tagging strategy ensures every image is unique, traceable, and suitable for automated deployments.

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why DevOps teams rely on tagging strategies:**

* Ensures Kubernetes/EKS always deploys the correct image
* Allows fast and accurate rollbacks
* Helps trace images back to specific commits or builds
* Prevents overwriting production images
* Enables environment isolation (dev/qa/prod)
* Supports GitOps workflows that rely on tag changes

### **Used in scenarios like:**

* CI/CD-driven deployments to EKS
* Release pipelines using Git tags
* Version tracking for microservices
* Debugging and audit trails

---

## **3. How It Works / Steps / Syntax**

### **1️⃣ Build Number Tagging (Simple & Common)**

```
myapp:${BUILD_NUMBER}
```

Example:

```
myapp:42
```

**Advantages:**

* Always unique
* Easy rollback
* Good for CI tracking

---

### **2️⃣ Commit SHA Tagging (Most Reliable for Production)**

Use a shortened Git commit hash:

```
myapp:${GIT_COMMIT}
```

Example:

```
myapp:9f2c4a1
```

**Advantages:**

* Fully unique
* Traceable to exact code commit
* Popular in enterprise CI/CD

---

### **3️⃣ Branch-Based Tagging (For Dev/QA Environments)**

```
myapp:dev
myapp:qa
myapp:staging
```

**Use case:**

* Auto-deploy dev branch → dev namespace
* Useful for preview environments

**Not recommended for production.**

---

### **4️⃣ Semantic Versioning (For Production Releases)**

```
vMAJOR.MINOR.PATCH
```

Example:

```
myapp:v1.4.2
```

**Advantages:**

* Clean release versions
* Works perfectly with Git tag–based triggers
* Easy for product teams to understand

---

## **Tagging Multiple Versions at Once (Best Practice)**

A single build can push multiple tags for the same image:

```
myapp:v1.4.2
myapp:202
myapp:9f2c4a1
```

This provides:

* Release version
* CI reference (build number)
* Commit traceability

---

## **4. Jenkins Pipeline Example (Tagging Logic)**

```groovy
stage('Tag Image') {
    steps {
        script {
            COMMIT = env.GIT_COMMIT.take(7)
            BUILD = env.BUILD_NUMBER

            sh """
            docker tag myapp:latest $REPO:${BUILD}
            docker tag myapp:latest $REPO:${COMMIT}
            docker tag myapp:latest $REPO:v1.3.${BUILD}
            """
        }
    }
}
```

---

## **5. Common Issues / Errors**

### ❌ Using `latest` tag

* Kubernetes may reuse cached image
* Hard to track what version is running
* Rollbacks become difficult

### ❌ Overwriting existing tags in ECR

* Causes deployment inconsistencies
* EKS may deploy an older cached image

### ❌ Tag mismatch between CI pipeline and Helm values

* Leads to stale deployments

### ❌ Using branch tags for production

* Not traceable or immutable

### ❌ Long commit SHA tags

* Makes dashboards messy; use short SHAs

---

## **6. Troubleshooting / Fixes**

### ✔ Always generate at least one unique, immutable tag

Commit SHA is the most reliable.

### ✔ Ensure CI updates the Kubernetes manifest/Helm values

Otherwise, rollout may never happen.

### ✔ Verify image exists in ECR

```sh
aws ecr describe-images --repository-name myapp
```

### ✔ Use imagePullPolicy properly

For dev environments:

```
imagePullPolicy: Always
```

### ✔ Avoid using the same tag for multiple builds

Prevents accidental rollbacks or mismatches.

---

## **7. Best Practices (EKS + CI/CD)**

### ✔ Use commit SHA for production tags

Most traceable and safest.

### ✔ Use BUILD_NUMBER for internal CI tracking

Easy correlation with logs.

### ✔ Use semantic versioning for formal releases

Clear business-friendly versioning.

### ✔ Push multiple tags for the same image

Maximizes traceability and rollback capability.

### ✔ Standardize tagging strategy across microservices

Ensures consistent deployment workflows.

---
---

# Pushing Docker Images to Amazon ECR — Detailed Explanation Notes

## **1. Concept / What**

Pushing Docker images to Amazon ECR means uploading the container image built in CI/CD (such as Jenkins) into an AWS-managed private registry. ECR stores all image versions, and Kubernetes/EKS pulls images directly from ECR during deployment.

The process in CI/CD is:

1. Login to ECR
2. Tag the built image
3. Push the tagged image to ECR

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why DevOps teams push images to ECR:**

* EKS requires images to be stored in a registry before deployment.
* Enables easy rollbacks using versioned image tags.
* Acts as a single source of truth for all microservice images.
* Ensures secure access using IAM roles instead of static passwords.
* Supports CI/CD pipelines for dev, qa, and production deployments.
* Allows scanning, lifecycle policies, and cross-region replication.

### **Where it is used:**

* Jenkins → Build → Push → Deploy to EKS
* Automated deployments triggered by Git events
* Multi-environment image promotion workflows

---

## **3. How It Works / Steps / Syntax (ECR Only)**

### **Step 1 — Login to ECR**

Authenticate Jenkins to ECR using AWS CLI:

```sh
aws ecr get-login-password --region ap-south-1 \
 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

* Generates a temporary token (valid ~12 hours)
* Prevents storing long-lived passwords
* Works automatically with IAM roles or Access Keys

---

### **Step 2 — Build Docker Image in Jenkins**

```sh
docker build -t myapp:${BUILD_NUMBER} .
```

---

### **Step 3 — Tag Image for ECR**

Tag must follow this structure:

```
<aws_account_id>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>
```

Example:

```sh
docker tag myapp:${BUILD_NUMBER} \
 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:${BUILD_NUMBER}
```

---

### **Step 4 — Push Image to ECR**

Upload the image:

```sh
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:${BUILD_NUMBER}
```

Layers are uploaded only when changed.

---

### **Complete Jenkins Pipeline Example (ECR Push Flow)**

```groovy
pipeline {
    agent any

    environment {
        REGION = "ap-south-1"
        REPO = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp"
    }

    stages {

        stage('Login to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${REGION} \
                    | docker login --username AWS --password-stdin ${REPO}
                """
            }
        }

        stage('Build Image') {
            steps {
                sh "docker build -t myapp:${BUILD_NUMBER} ."
            }
        }

        stage('Tag Image') {
            steps {
                sh "docker tag myapp:${BUILD_NUMBER} ${REPO}:${BUILD_NUMBER}"
            }
        }

        stage('Push Image') {
            steps {
                sh "docker push ${REPO}:${BUILD_NUMBER}"
            }
        }
    }
}
```

---

## **4. Common Issues / Errors (ECR Focused)**

### ❌ **1. "no basic auth credentials"**

* ECR login not performed
* Wrong region used in login command

### ❌ **2. ECR repository does not exist**

Error:

```
repository not found
```

Fix:

```sh
aws ecr create-repository --repository-name myapp
```

### ❌ **3. IAM role missing ECR permissions**

Required permissions:

* `ecr:GetAuthorizationToken`
* `ecr:InitiateLayerUpload`
* `ecr:PutImage`
* `ecr:BatchCheckLayerAvailability`

### ❌ **4. Jenkins agent missing AWS CLI or Docker**

Push will fail without required tools.

### ❌ **5. EKS pods fail with ImagePullBackOff**

Reasons:

* Wrong image tag in Deployment YAML
* Jenkins failed to push image
* Node IAM role missing:

```
AmazonEC2ContainerRegistryReadOnly
```

### ❌ **6. Incorrect tagging strategy**

Using `latest` leads to inconsistent deployments.

---

## **5. Troubleshooting / Fixes**

### ✔ Validate that image exists in ECR

```sh
aws ecr describe-images --repository-name myapp
```

### ✔ Check Jenkins ECR login

```sh
docker login ...
```

### ✔ Ensure correct AWS region in login command

Use same region as repo.

### ✔ Install AWS CLI and Docker properly on agents

Required for build + push.

### ✔ Verify Deployment YAML tag matches CI build tag

Otherwise rollout won't occur.

### ✔ Check IAM permissions on Jenkins and EKS worker nodes

Attach:

```
AmazonEC2ContainerRegistryPowerUser (for Jenkins)
AmazonEC2ContainerRegistryReadOnly (for EKS nodes)
```

---

## **6. Best Practices / Tips (ECR + EKS)**

### ✔ Push **unique immutable tags** (commit SHA, build number)

Avoid overwriting existing tags.

### ✔ Use IAM roles for Jenkins EC2 agents

Avoid storing long-lived AWS keys.

### ✔ Keep separate ECR repos for each microservice

Better organization and lifecycle management.

### ✔ Push multiple tags per build if needed

E.g., commit SHA + build number + semantic version.

### ✔ Fail CI pipeline immediately if push fails

Prevents bad deployments.

### ✔ Keep Docker images optimized and small

Faster push → faster deployments.

---
---

# Scanning Docker Images During CI/CD (AWS ECR + Jenkins) — Detailed Explanation Notes

## **1. Concept / What**

Image scanning is the process of analyzing a Docker image to detect security vulnerabilities, misconfigurations, outdated packages, malware, or accidental inclusion of secrets. In DevSecOps workflows, image scanning happens **during CI/CD**, before images are pushed to ECR or deployed to EKS.

Two common scanning methods used in AWS environments:

* **Local CI scanning (e.g., Trivy in Jenkins)**
* **ECR native image scanning (Basic or Enhanced/Inspector)**

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why DevOps teams scan images:**

* Prevent deploying vulnerable images into EKS clusters
* Detect HIGH/CRITICAL CVEs early
* Meet compliance requirements (SOC2, ISO, PCI)
* Ensure secure base images are used
* Block deployment pipelines when vulnerabilities are found
* Detect secrets accidentally committed into code or image layers

### **Real scenarios:**

* Jenkins scans images automatically during each build
* ECR scans newly pushed images on every deployment cycle
* Dev teams fix vulnerabilities before merge, reducing risk

---

## **3. How It Works / Steps / Syntax**

### **Method 1: Local Image Scanning in Jenkins (using Trivy)**

Trivy is a fast, accurate, open-source scanner widely used in CI/CD.

#### **Install Trivy on Jenkins agent**

Example (Debian/Ubuntu agent):

```sh
sudo apt-get install wget -y
wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_0.48.3_Linux-64bit.deb
sudo dpkg -i trivy_0.48.3_Linux-64bit.deb
```

#### **Run Trivy Scan inside Jenkins pipeline**

```groovy
stage('Scan Image') {
    steps {
        sh """
        trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:${BUILD_NUMBER}
        """
    }
}
```

* `--exit-code 1` → fails pipeline on HIGH/CRITICAL vulnerabilities
* Prevents insecure images from being pushed to ECR

---

### **Method 2: AWS ECR Native Scanning**

ECR supports two scanning modes:

* **Basic scanning (on push)**
* **Enhanced scanning (continuous, powered by Inspector)**

#### **Enable ECR Scan on Push**

```sh
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true
```

#### **View Scan Results**

```sh
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=42
```

Output includes:

* CVE list
* Severity: LOW, MEDIUM, HIGH, CRITICAL
* Affected packages
* Remediation steps

---

## **4. Example Jenkins Pipeline (Build → Scan → Push)**

```groovy
pipeline {
    agent any

    environment {
        REGION = "ap-south-1"
        REPO = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp"
    }

    stages {

        stage('Build') {
            steps {
                sh "docker build -t myapp:${BUILD_NUMBER} ."
            }
        }

        stage('Local Security Scan') {
            steps {
                sh """
                trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:${BUILD_NUMBER}
                """
            }
        }

        stage('Login to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${REGION} \
                  | docker login --username AWS --password-stdin ${REPO}
                """
            }
        }

        stage('Push Image') {
            steps {
                sh """
                docker tag myapp:${BUILD_NUMBER} ${REPO}:${BUILD_NUMBER}
                docker push ${REPO}:${BUILD_NUMBER}
                """
            }
        }
    }
}
```

---

## **5. Common Issues / Errors**

### ❌ **1. Trivy scan fails with critical vulnerabilities**

Pipeline stops, preventing insecure deployments.

### ❌ **2. Trivy cannot download vulnerability database**

* Jenkins agent has no internet
* Fix: use offline Trivy DB

### ❌ **3. ECR scan not triggered**

* scanOnPush is disabled

### ❌ **4. Outdated base images contain hundreds of CVEs**

Example:

```
node:14
python:3.6
ubuntu:18.04
```

These are deprecated, insecure.

### ❌ **5. Jenkins agent does not have Trivy installed**

Scan stage fails immediately.

### ❌ **6. EKS pulls images before ECR scanning completes**

Scan results might be incomplete if deployment triggers too fast.

---

## **6. Troubleshooting / Fixes**

### ✔ Update base images regularly

Use:

* `node:20-alpine`
* `python:3.11`
* `ubuntu:22.04`

### ✔ Fail pipeline on HIGH/CRITICAL issues

Improves security posture.

### ✔ Ensure Jenkins agents have internet or offline DB configured

Trivy needs vulnerability database.

### ✔ Enable ECR scanOnPush for all repos

Provides continuous security checks.

### ✔ Use multi-stage builds

Reduces unnecessary OS packages and lowers vulnerabilities.

### ✔ Use ECR Enhanced Scanning for production workloads

More accurate and deeper analysis.

---

## **7. Best Practices / Tips (EKS + CI/CD)**

### ✔ Scan images twice:

1. During CI (Trivy)
2. After push (ECR)

### ✔ Block deployment of vulnerable images

Critical for production clusters.

### ✔ Maintain an approved list of base images

Prevents developers from using risky or deprecated images.

### ✔ Automate Jira/Slack alerts when vulnerabilities found

Enhances DevSecOps workflow.

### ✔ Keep images minimal

Smaller images = fewer vulnerabilities.

### ✔ Re-scan images periodically

ECR Enhanced Scanning supports automatic continuous scanning.

---
---

# Scanning Docker Images During CI/CD (AWS ECR + Jenkins) — Detailed Explanation Notes

## **1. Concept / What**

Image scanning is the process of analyzing a Docker image to detect security vulnerabilities, misconfigurations, outdated packages, malware, or accidental inclusion of secrets. In DevSecOps workflows, image scanning happens **during CI/CD**, before images are pushed to ECR or deployed to EKS.

Two common scanning methods used in AWS environments:

* **Local CI scanning (e.g., Trivy in Jenkins)**
* **ECR native image scanning (Basic or Enhanced/Inspector)**

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why DevOps teams scan images:**

* Prevent deploying vulnerable images into EKS clusters
* Detect HIGH/CRITICAL CVEs early
* Meet compliance requirements (SOC2, ISO, PCI)
* Ensure secure base images are used
* Block deployment pipelines when vulnerabilities are found
* Detect secrets accidentally committed into code or image layers

### **Real scenarios:**

* Jenkins scans images automatically during each build
* ECR scans newly pushed images on every deployment cycle
* Dev teams fix vulnerabilities before merge, reducing risk

---

## **3. How It Works / Steps / Syntax**

### **Method 1: Local Image Scanning in Jenkins (using Trivy)**

Trivy is a fast, accurate, open-source scanner widely used in CI/CD.

#### **Install Trivy on Jenkins agent**

Example (Debian/Ubuntu agent):

```sh
sudo apt-get install wget -y
wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_0.48.3_Linux-64bit.deb
sudo dpkg -i trivy_0.48.3_Linux-64bit.deb
```

#### **Run Trivy Scan inside Jenkins pipeline**

```groovy
stage('Scan Image') {
    steps {
        sh """
        trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:${BUILD_NUMBER}
        """
    }
}
```

* `--exit-code 1` → fails pipeline on HIGH/CRITICAL vulnerabilities
* Prevents insecure images from being pushed to ECR

---

### **Method 2: AWS ECR Native Scanning**

ECR supports two scanning modes:

* **Basic scanning (on push)**
* **Enhanced scanning (continuous, powered by Inspector)**

#### **Enable ECR Scan on Push**

```sh
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true
```

#### **View Scan Results**

```sh
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=42
```

Output includes:

* CVE list
* Severity: LOW, MEDIUM, HIGH, CRITICAL
* Affected packages
* Remediation steps

---

## **4. Example Jenkins Pipeline (Build → Scan → Push)**

```groovy
pipeline {
    agent any

    environment {
        REGION = "ap-south-1"
        REPO = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp"
    }

    stages {

        stage('Build') {
            steps {
                sh "docker build -t myapp:${BUILD_NUMBER} ."
            }
        }

        stage('Local Security Scan') {
            steps {
                sh """
                trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:${BUILD_NUMBER}
                """
            }
        }

        stage('Login to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${REGION} \
                  | docker login --username AWS --password-stdin ${REPO}
                """
            }
        }

        stage('Push Image') {
            steps {
                sh """
                docker tag myapp:${BUILD_NUMBER} ${REPO}:${BUILD_NUMBER}
                docker push ${REPO}:${BUILD_NUMBER}
                """
            }
        }
    }
}
```

---

## **5. Common Issues / Errors**

### ❌ **1. Trivy scan fails with critical vulnerabilities**

Pipeline stops, preventing insecure deployments.

### ❌ **2. Trivy cannot download vulnerability database**

* Jenkins agent has no internet
* Fix: use offline Trivy DB

### ❌ **3. ECR scan not triggered**

* scanOnPush is disabled

### ❌ **4. Outdated base images contain hundreds of CVEs**

Example:

```
node:14
python:3.6
ubuntu:18.04
```

These are deprecated, insecure.

### ❌ **5. Jenkins agent does not have Trivy installed**

Scan stage fails immediately.

### ❌ **6. EKS pulls images before ECR scanning completes**

Scan results might be incomplete if deployment triggers too fast.

---

## **6. Troubleshooting / Fixes**

### ✔ Update base images regularly

Use:

* `node:20-alpine`
* `python:3.11`
* `ubuntu:22.04`

### ✔ Fail pipeline on HIGH/CRITICAL issues

Improves security posture.

### ✔ Ensure Jenkins agents have internet or offline DB configured

Trivy needs vulnerability database.

### ✔ Enable ECR scanOnPush for all repos

Provides continuous security checks.

### ✔ Use multi-stage builds

Reduces unnecessary OS packages and lowers vulnerabilities.

### ✔ Use ECR Enhanced Scanning for production workloads

More accurate and deeper analysis.

---

## **7. Best Practices / Tips (EKS + CI/CD)**

### ✔ Scan images twice:

1. During CI (Trivy)
2. After push (ECR)

### ✔ Block deployment of vulnerable images

Critical for production clusters.

### ✔ Maintain an approved list of base images

Prevents developers from using risky or deprecated images.

### ✔ Automate Jira/Slack alerts when vulnerabilities found

Enhances DevSecOps workflow.

### ✔ Keep images minimal

Smaller images = fewer vulnerabilities.

### ✔ Re-scan images periodically

ECR Enhanced Scanning supports automatic continuous scanning.

---
---

# Continuous Delivery vs Continuous Deployment — Detailed Explanation Notes

## **1. Concept / What**

Continuous Delivery (CD) and Continuous Deployment (CDP) are stages in a CI/CD pipeline that determine how software is released after it is built, tested, scanned, and packaged. Both automate the release process, but they differ in whether a **manual approval** is required before deploying to production.

### **Continuous Delivery**

The pipeline automates everything **up to staging or pre-production**, then waits for a **manual approval** before deploying to production.

### **Continuous Deployment**

The pipeline automates **everything, including production deployment**, with **no human approval** required.

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why Continuous Delivery is used:**

* Production deployments require human validation
* Compliance / audit / regulatory requirements
* Manager or team lead must approve production releases
* High-risk environments (banking, finance, healthcare)
* Ensures business control over production changes

### **Why Continuous Deployment is used:**

* Fast, automated production releases
* Safer for microservices and low-risk applications
* Reduces manual release bottlenecks
* Allows rapid iteration and delivery

---

## **3. How It Works / Steps / Syntax**

### **Continuous Deployment Flow:**

```
CI → Build → Test → Scan → Push Image → Deploy to DEV → Deploy to QA → Deploy to STAGE → Auto Deploy to PROD
```

### **Continuous Delivery Flow:**

```
CI → Build → Test → Scan → Push Image → Deploy to DEV → Deploy to QA → Deploy to STAGE → WAIT FOR APPROVAL → Deploy to PROD
```

The key difference is the approval gate before production.

---

## **Example: Jenkins Approval Step (Continuous Delivery)**

A manual approval block stops the pipeline before production deployment:

```groovy
stage('Approval for Production') {
    steps {
        timeout(time: 1, unit: 'HOURS') {
            input message: "Approve deployment to Production?"
        }
    }
}
```

Only after the authorized person clicks **Proceed**, the pipeline continues.

---

## **4. Common Issues / Errors**

### ❌ Deployment happens without approval

* Pipeline is configured as Continuous Deployment instead of Delivery

### ❌ Approval stage times out

* Manager did not approve within the defined time window

### ❌ Wrong person approves the deployment

* Lack of RBAC or pipeline permissions

### ❌ Pipeline stuck in approval indefinitely

* No timeout configured in `input` step

### ❌ Compliance audits fail

* Production deployments not recorded or approved properly

---

## **5. Troubleshooting / Fixes**

### ✔ Add timeout to approval stage

Prevents pipelines from hanging forever.

### ✔ Restrict approval to specific users

Use Jenkins RBAC or role-based access control.

### ✔ Add notifications for approval

Slack / Teams / Email alerts for waiting approvals.

### ✔ Log all approval actions

Useful for audits and compliance.

### ✔ Maintain separate pipelines for DEV/QA and PROD

Cleaner release flow.

---

## **6. Best Practices / Tips (EKS + CI/CD)**

### ✔ Use **Continuous Delivery** for production clusters

Keeps human oversight.

### ✔ Use **Continuous Deployment** for dev/testing environments

Faster iteration.

### ✔ Treat the approval stage as a quality gate

Ensures production readiness.

### ✔ Store deployment configs (Helm/YAML) in Git

Enables traceability and GitOps workflows.

### ✔ Include automated checks before approval

* Security scans
* Integration tests
* Performance tests

### ✔ Document release procedures clearly

Helps teams follow standard deployment processes.

---
---

