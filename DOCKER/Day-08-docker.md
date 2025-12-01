# Docker Security: Running Non-Root Containers (Detailed Explanation Notes)

---

## **1. Concept / What**

A **non-root container** is a Docker container that does **not run processes as the root user** (UID 0). Instead, it runs under a dedicated, restricted Linux user (e.g., `appuser`, UID 1000). Even though containers provide isolation, running as root inside the container still gives elevated permissions and increases security risk.

---

## **2. Why / Purpose / Real-World Use Cases**

### **2.1 Security Hardening**

* Reduces impact if the container is compromised.
* Prevents attackers from modifying system-level files.

### **2.2 Kubernetes/EKS Requirements**

* Many clusters enforce `runAsNonRoot: true`.
* Images without non-root support fail to deploy.

### **2.3 Compliance & Best Practices**

* CIS Benchmarks and enterprise security guidelines recommend avoiding root.

### **2.4 Safe CI/CD Execution**

* Jenkins or GitLab pipelines often validate that images run safely as non-root.
* Prevents accidental root-based builds.

### **2.5 Application Safety**

* Helps avoid accidental deletion or modification of sensitive directories.

---

## **3. How It Works / Steps / Syntax**

### **3.1 Create a Non-Root User in Dockerfile (Recommended)**

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Create group and user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy application files
COPY . .

# Give ownership to non-root user
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

CMD ["node", "server.js"]
```

### **3.2 Using Numeric UID Directly**

(Only works if UID exists in base image)

```dockerfile
USER 1000
```

### **3.3 Enforcing Non-Root in Kubernetes (if needed)**

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  runAsNonRoot: true
```

### **3.4 CI/CD Example (Jenkins)**

```groovy
stage('Build Image') {
  steps {
    sh "docker build -t myapp:${BUILD_NUMBER} ."
    sh "docker run --user 1000 myapp:${BUILD_NUMBER} node --version"
  }
}
```

---

## **4. Common Issues / Errors**

### **4.1 Permission Denied When App Starts**

Cause: Files owned by root.

```
/app/server.js: Permission denied
```

Fix:

```dockerfile
RUN chown -R appuser:appgroup /app
```

### **4.2 Kubernetes Error: Image Has No Non-Root User**

```
container has runAsNonRoot and image has no non-root user
```

Fix:

* Add a dedicated user in Dockerfile.
* Avoid relying only on Kubernetes overrides.

### **4.3 Log/File Writes Fail**

Cause: Non-root user lacks write permission.
Fix: Use application-owned directory, e.g., `/app/logs`.

### **4.4 UID Exists but No Home Directory**

Some users created without home directories cause runtime issues.
Fix: Add home or use `--no-create-home` consciously.

---

## **5. Troubleshooting / Fixes**

### **5.1 Check Running User**

```
docker exec -it <container_id> id
```

Expected output:

```
uid=1000(appuser) gid=1000(appgroup)
```

### **5.2 Check Ownership of Application Files**

```
docker exec -it <container_id> ls -l /app
```

### **5.3 Validate User Exists Inside Image**

```
docker run <image> id appuser
```

### **5.4 Fix Missing Permissions**

Add necessary `chown` or `chmod` in Dockerfile.

---

## **6. Best Practices / Tips**

### ✔ Create a dedicated non-root user in the Dockerfile.

### ✔ Use numeric UID/GID (e.g., 1000 or 1001).

### ✔ Ensure correct ownership of app directories.

### ✔ Avoid running as root unless absolutely necessary.

### ✔ Validate non-root behavior in CI/CD with `docker run --user`.

### ✔ Use minimal base images (Alpine, Distroless) to enforce good practices.

### ✔ Prefer fixing at the Dockerfile level instead of Kubernetes overrides.

---
---

# Docker Security: Image Scanning (Trivy, ECR) — Detailed Explanation Notes

---

## **1. Concept / What**

**Image scanning** is the security process of analyzing Docker images to find:

* OS vulnerabilities (CVEs)
* Application/library vulnerabilities
* Misconfigurations
* Hardcoded secrets
* Malware or supply chain risks

Two main scanners used in DevOps:

* **Trivy** (open-source, CLI, used in CI/CD)
* **Amazon ECR Scan-on-Push** (AWS Inspector engine)

Trivy is the **enforcement scanner**.
ECR scanning is an **optional visibility layer**.

---

## **2. Why / Purpose / Real-World Use Cases**

### **2.1 Vulnerability Prevention**

Prevents shipping images with known Critical/High CVEs.

### **2.2 DevSecOps and Compliance**

Required for PCI-DSS, SOC2, CIS benchmarks, and enterprise standards.

### **2.3 CI/CD Safety**

Pipelines fail early if vulnerabilities exist.

### **2.4 Secrets Detection**

Scans for hardcoded credentials inside Docker layers and source code.

### **2.5 Supply Chain Security**

Detects risks introduced by compromised base images or dependencies.

### **2.6 Continuous Monitoring (ECR)**

ECR gives post-push visibility of vulnerabilities over time.

---

## **3. How It Works / Steps / Syntax**

# **A. Trivy (Primary Scanner for CI/CD)**

### **Install Trivy**

```
sudo apt install trivy
```

OR

```
brew install trivy
```

### **Scan a Docker Image Locally**

```
trivy image myapp:latest
```

### **Scan a File System / Source Code**

```
trivy fs .
```

### **Scan a Remote Image in ECR**

```
trivy image <aws_account>.dkr.ecr.<region>.amazonaws.com/myapp:latest
```

### **CI/CD Example (Jenkins)**

```groovy
stage('Security Scan') {
  steps {
    sh 'trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:${BUILD_NUMBER}'
  }
}
```

`--exit-code 1` fails pipeline if vulnerabilities are found.

---

# **B. ECR Scan-on-Push (Secondary Optional Scanner)**

ECR uses **AWS Inspector** — not Trivy.

### **Enable Scan on Push**

```
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true
```

### **Start Manual Scan**

```
aws ecr start-image-scan \
  --repository-name myapp \
  --image-id imageTag=latest
```

### **View Scan Findings**

```
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=latest
```

### **Important:**

ECR scanning **does not block the push** and **cannot fail CI/CD**.
It only provides reports.

---

## **4. Common Issues / Errors**

### **4.1 Trivy DB Errors / Offline Mode**

```
Rate limit exceeded
Unable to download vulnerability DB
```

Fix:

```
trivy image --download-db-only
```

### **4.2 Cannot Scan Private ECR Image**

Fix login first:

```
aws ecr get-login-password --region ap-south-1 | \
docker login --username AWS --password-stdin <aws_account>.dkr.ecr.ap-south-1.amazonaws.com
```

### **4.3 CI/CD Fails Due to Too Many Vulnerabilities**

Fix:

* Update base image
* Patch OS layers
* Upgrade libraries
* Switch to Alpine/Distroless

### **4.4 False Positives in Debian/Ubuntu Images**

Fix: Filter severities:

```
--severity CRITICAL,HIGH
```

---

## **5. Troubleshooting / Fixes**

### **5.1 Trace Vulnerability Origin**

```
trivy image --trace myapp:latest
```

### **5.2 Scan Dockerfile for Misconfigurations**

```
trivy config Dockerfile
```

### **5.3 Scan Repo for Secret Leaks**

```
trivy fs --security-checks secret .
```

### **5.4 Validate ECR Scans Triggered**

```
aws ecr describe-image-scan-findings ...
```

---

## **6. Best Practices / Tips**

### ✔ Always run Trivy in CI **before pushing** to ECR (mandatory)

### ✔ Use ECR Scan-on-Push **as an additional layer** (optional)

### ✔ Fail pipeline on CRITICAL/HIGH vulnerabilities

### ✔ Keep images updated and rebuild regularly

### ✔ Use minimal base images (Alpine, Distroless)

### ✔ Avoid installing unnecessary packages (reduces CVEs)

### ✔ Add SBOM scanning (Syft, CycloneDX) for supply chain safety

### ✔ Regularly review ECR scan results for long-term image safety

---
---


# Trivy Learning — Detailed Explanation Notes

---

## **1. Concept / What**

**Trivy** is an open-source vulnerability and security scanner for:

* Docker images
* File systems (source code)
* Kubernetes manifests (optional)
* Dockerfiles
* IaC (Terraform, Helm charts)
* Git repositories
* SBOMs

Trivy identifies:

* OS vulnerabilities (CVEs)
* Application/library vulnerabilities
* Secrets inside code or images
* Misconfigurations
* Supply chain risks

It is the **primary scanner used in DevSecOps pipelines** to secure container images before pushing to registries like ECR.

---

## **2. Why / Purpose / Real-World Use Cases**

### **2.1 Block Vulnerable Images Before Deployment**

Trivy prevents insecure images from reaching ECR, EKS, or production by failing the CI pipeline.

### **2.2 Detect Secrets in Code and Images**

Finds leaked:

* AWS keys
* Tokens
* Passwords
* SSH keys
* DB credentials

### **2.3 Scan Base Images and Dependencies**

Ensures OS packages and language libraries are patched.

### **2.4 Compliance & Audit Requirements**

Used in large orgs to meet:

* SOC2
* PCI-DSS
* ISO27001
* CIS benchmarks

### **2.5 Supply Chain Security**

Helps validate the safety of base images and 3rd-party libraries.

---

## **3. How It Works / Steps / Syntax**

Trivy uses a **local vulnerability database**, downloaded from:

* NVD (National Vulnerability Database)
* GitHub Advisories
* OS vendor security feeds
* Aqua Security mirrors

DB is stored at:

```
~/.cache/trivy
```

### **3.1 Install Trivy**

Linux:

```
sudo apt install trivy
```

Mac:

```
brew install trivy
```

---

## **3.2 Scan Docker Images**

```
trivy image myapp:latest
```

Identifies CVEs based on OS + dependencies.

### **Scan Remote ECR Image**

```
trivy image <aws_account>.dkr.ecr.<region>.amazonaws.com/myapp:latest
```

---

## **3.3 Scan Source Code (File System)**

```
trivy fs .
```

Detects secrets + misconfigurations.

### **Secret-Only Scan**

```
trivy fs --security-checks secret .
```

---

## **3.4 Scan Dockerfile**

```
trivy config Dockerfile
```

Finds misconfigurations like:

* running as root
* exposing unnecessary ports
* installing insecure packages

---

## **3.5 Scan Kubernetes YAML (Optional for Later)**

```
trivy config k8s/
```

Detects insecure:

* securityContext
* privileged containers
* hostPath
* missing limits

(We'll cover later in Kubernetes security.)

---

## **3.6 CI/CD Integration (Most Important Use Case)**

```groovy
stage('Security Scan') {
  steps {
    sh 'trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:${BUILD_NUMBER}'
  }
}
```

Meaning:

* If CRITICAL/HIGH CVEs exist → pipeline FAILS
* Prevents push to ECR

---

## **4. Common Issues / Errors**

### **4.1 Trivy DB Not Updated / Offline**

```
Rate limit exceeded
Unable to download vulnerability DB
```

Fix:

```
trivy image --download-db-only
```

### **4.2 Cannot Scan ECR Image**

Fix AWS login:

```
aws ecr get-login-password --region ap-south-1 | \
docker login --username AWS --password-stdin <repo_url>
```

### **4.3 Too Many Vulnerabilities in Base Image**

Fix:

* Switch to Alpine / Slim
* Update base image
* Upgrade dependencies

### **4.4 Not Fixable CVEs**

Fix:

* Change OS family (Debian -> Alpine)
* Use Distroless images

---

## **5. Troubleshooting / Fixes**

### **5.1 See Which Layer Adds Vulnerability**

```
trivy image --trace myapp:latest
```

### **5.2 Check DB Cache**

```
ls ~/.cache/trivy
```

### **5.3 Scan SBOM**

If using Syft/Grype:

```
trivy sbom sbom.json
```

---

## **6. Best Practices / Tips**

### ✔ Use Trivy in CI/CD BEFORE pushing to ECR (mandatory)

### ✔ Fail pipeline on CRITICAL/HIGH

### ✔ Use minimal base images (Alpine, Slim, Distroless)

### ✔ Avoid unnecessary OS package installations

### ✔ Rebuild images regularly to pick newer patches

### ✔ Scan source code and Dockerfiles for secrets

### ✔ Use Trivy + ECR scanning combo for layered security

---
---

# Docker Security: Avoiding Secrets in Dockerfile — Detailed Explanation Notes

---

## **1. Concept / What**

**Avoiding secrets in Dockerfile** means ensuring that **no sensitive data** is ever embedded in:

* Dockerfile instructions
* Docker image layers
* Build arguments
* Environment variables inside the image
* Files copied into the image

Secrets include:

* AWS access keys
* DB passwords
* OAuth tokens
* SSH private keys
* Certificates
* API keys
* Cloud credentials

Docker images are immutable and shared across teams/environments. Any secret added during build becomes permanently stored in an image layer.

---

## **2. Why / Purpose / Real-World Use Cases**

### **2.1 Prevent Permanent Secret Exposure**

Docker image layers are forever. Even deleted lines remain in history.

### **2.2 Prevent Unauthorized Access**

Anyone who can pull the image can extract secrets using:

```
docker history
```

Or by extracting layers.

### **2.3 Avoid CI/CD Secret Leaks**

If secrets are embedded in Dockerfile commands, Jenkins/GitHub logs expose them.

### **2.4 Comply With Security Standards**

Embedding secrets violates:

* CIS Benchmarks
* SOC2
* ISO 27001
* Company internal security policies

### **2.5 Enable Secure Supply Chain**

Images must be free from embedded credentials when moved across environments.

---

## **3. How It Works / Steps / Syntax**

### ❌ **3.1 What NOT to do (Bad Practices)**

#### **Hardcoding secrets**

```
ENV AWS_SECRET=abcd1234
```

#### **Copying secret files**

```
COPY id_rsa /root/.ssh/id_rsa
```

#### **Using ARG for secrets**

```
ARG PASSWORD=MyPass
RUN echo $PASSWORD
```

(ARG values are visible in image history.)

#### **Passing secrets directly in RUN commands**

```
RUN aws configure set aws_secret_key ABCD
```

These get stored permanently in image layers.

---

### ✔ **3.2 Correct & Secure Approaches**

### **A. Use Docker BuildKit Secrets (for build-time needs)**

BuildKit provides **temporary, non-persistent secrets** available only during build.

Dockerfile:

```
RUN --mount=type=secret,id=awskey cat /run/secrets/awskey
```

Build command:

```
docker build --secret id=awskey,src=awskey.txt .
```

Key points:

* `/run/secrets/` is a **temporary** virtual folder
* Secret is **not stored** in the image
* Used only for that RUN command
* Deleted immediately after

---

### **B. Inject secrets at runtime (not in Dockerfile)**

```
docker run -e DB_PASSWORD=$DB_PASSWORD myapp
```

The secret is provided **after the image is built**, which is safe.

---

### **C. Use Docker Compose Secrets**

```yaml
secrets:
  db_pass:
    file: ./password.txt

services:
  app:
    image: myapp
    secrets:
      - db_pass
```

---

### **D. Use Kubernetes Secrets (EKS)**

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

Runtime-only secrets = safe.

---

### **E. Use AWS Secrets Manager / SSM Parameter Store**

CI/CD pipeline retrieves secret securely:

```
export DB_PASS=$(aws secretsmanager get-secret-value ...)
docker run -e DB_PASS=$DB_PASS myapp
```

---

## **4. Common Issues / Errors**

### **4.1 Secrets accidentally included via COPY . .**

Fix using `.dockerignore`:

```
*.pem
*.key
*.env
*.token
.git/
```

### **4.2 ARG values leaked**

ARG values appear in:

* `docker history`
* Image metadata

### **4.3 Files left on local system**

Temporary secret files must be deleted to avoid risk.

### **4.4 Secrets echoed in CI/CD build logs**

Avoid printing environment variables.

---

## **5. Troubleshooting / Fixes**

### **Check if secrets leaked into image layers**

```
docker history myapp:latest
```

### **Extract image to verify**

```
docker save myapp | tar -x
```

Look inside layer contents.

### **Scan Dockerfile for misconfigurations**

```
trivy config Dockerfile
```

### **Check if secret exists in cached layers**

Use BuildKit to rebuild without cache:

```
docker build --no-cache .
```

---

## **6. Best Practices / Tips**

### ✔ Never put secrets in Dockerfile

### ✔ Use BuildKit for build-time-only secrets

### ✔ Use Kubernetes Secrets or AWS Secrets Manager for runtime

### ✔ Use `.dockerignore` to prevent accidental inclusion

### ✔ Never store secrets permanently in local files

### ✔ Use CI/CD credential stores instead of local files

### ✔ Scan Dockerfile using Trivy

### ✔ Rotate secrets if exposed

---
---

# Docker Security: Docker Capabilities Basics — Detailed Explanation Notes

---

## **1. Concept / What**

**Docker capabilities** are small, granular pieces of Linux root privileges that Docker assigns to containers.

Instead of giving full root power, Docker provides a **limited, controlled subset** of permissions that allow specific actions like:

* Binding to low ports (<1024)
* Managing file ownership
* Using chroot
* Accessing certain network features

Capabilities allow containers to run safely without giving them full system-level access.

---

## **2. Why / Purpose / Real-World Use Cases**

### **2.1 Reduce Attack Surface**

Even if a container runs as root, dangerous capabilities (kernel access, network control, module loading) are removed by default.

### **2.2 Least-Privilege Model**

Only enable capabilities that the application truly needs.

### **2.3 Kubernetes Security Requirements**

Pod Security Standards (PSS) and EKS best practices encourage dropping dangerous capabilities.

### **2.4 Compliance & Hardening**

CIS Benchmarks require minimizing container privileges.

### **2.5 Prevent Container Escape Attacks**

Dropping capabilities helps block host-level modifications.

---

## **3. How It Works / Steps / Syntax**

Docker starts containers with a safe default set of capabilities.
You can:

* Drop capabilities
* Add capabilities
* Run fully privileged containers (not recommended)

---

### **3.1 Drop All Capabilities (Strongest Security)**

```
docker run --cap-drop=ALL nginx
```

Good for normal apps that don’t need special access.

---

### **3.2 Drop All + Add Only Required Capability**

Example: app needs to bind to port 80

```
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```

This is a recommended pattern.

---

### **3.3 Add Dangerous Capabilities (Avoid When Possible)**

Examples of risky capabilities:

* `SYS_ADMIN` → near-root access to host
* `NET_ADMIN` → can modify host network
* `SYS_MODULE` → load kernel modules

Avoid unless absolutely necessary.

---

### **3.4 Privileged Mode (Do NOT Use for Apps)**

```
docker run --privileged ubuntu
```

Gives container **full host capabilities**.
Useful only for:

* Docker-in-Docker
* Kernel debugging
  Not for production apps.

---

### **3.5 Capabilities in Kubernetes (EKS)**

Drop all:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

Add required capability:

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```

This is commonly used in hardened clusters.

---

## **4. Common Issues / Errors**

### **4.1 “Permission denied” when binding to port 80**

Fix by adding:

```
--cap-add=NET_BIND_SERVICE
```

### **4.2 “Operation not permitted” when changing network settings**

Requires:

```
--cap-add=NET_ADMIN
```

(Use with caution.)

### **4.3 Using --privileged unnecessarily**

Creates a major security risk.

---

## **5. Troubleshooting / Fixes**

### **Check container capabilities**

```
docker run --rm alpine capsh --print
```

### **Check capabilities inside running container**

```
cat /proc/self/status | grep Cap
```

### **Validate Kubernetes capabilities**

```
kubectl exec -it pod -- capsh --print
```

---

## **6. Best Practices / Tips**

### ✔ Drop ALL capabilities by default

### ✔ Add only the minimal capabilities required

### ✔ Avoid privileged containers entirely

### ✔ Combine with non-root user for stronger isolation

### ✔ Follow Kubernetes Pod Security Standards

### ✔ Never add SYS_ADMIN unless absolutely required

### ✔ Use capabilities only when the application demands it

---
---

# Docker Security: Minimal Base Images (Alpine / Distroless) — Detailed Explanation Notes

---

## **1. Concept / What**

Minimal base images are lightweight Docker images that contain only the essential components required to run an application. They remove unnecessary utilities, shells, and OS packages, reducing:

* Image size
* Vulnerabilities
* Attack surface

The two most commonly used minimal images are:

* **Alpine** → Tiny (5MB), includes BusyBox + apk
* **Distroless** → Even smaller, contains *only* application runtime (no shell, no package manager)

---

## **2. Why / Purpose / Real-World Use Cases**

### **2.1 Reduce Image Size & Improve Performance**

Smaller images lead to:

* Faster CI/CD builds
* Faster ECR/EKS pull times
* Faster pod start times
* Lower storage cost

### **2.2 Fewer Vulnerabilities (Less CVEs)**

Minimal images contain fewer Linux libraries → fewer vulnerabilities for Trivy/ECR to detect.

### **2.3 Reduced Attack Surface**

* Alpine has minimal utilities
* Distroless has **no shell** → attackers cannot exec into the container

### **2.4 Better Compliance & Security Hardening**

Meets CIS, SOC2, ISO 27001 container security standards.

### **2.5 Ideal for Production Workloads**

Distroless is commonly used in Kubernetes clusters for maximum security.

---

## **3. How It Works / Steps / Syntax**

---

### **3.1 Using Alpine (Node.js Example)**

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

USER node

CMD ["node", "server.js"]
```

Benefits:

* ~5MB base image
* Good security
* Fast build & deploy

---

### **3.2 Python Example (Alpine)**

```dockerfile
FROM python:3.12-alpine

WORKDIR /app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

---

### **3.3 Distroless Example (Java App)**

```dockerfile
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /app
COPY . .
RUN ./gradlew build

FROM gcr.io/distroless/java17
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
CMD ["app.jar"]
```

Distroless Highlights:

* No package manager
* No shell (cannot exec into container)
* Very low CVE count

---

## **4. Common Issues / Errors**

### **4.1 Alpine libc (musl) Compatibility Issues**

Some binaries require `glibc`, leading to errors:

```
Error loading shared library
Segmentation fault
```

Fix: use Debian **slim** base images instead:

```
python:slim
node:slim
```

---

### **4.2 Cannot Debug Distroless Containers**

No `sh`, `bash`, or shell.
Fix options:

* Use a debug sidecar
* Temporarily switch to Alpine for debugging

---

### **4.3 Missing Dependencies in Alpine**

Fix:

```
apk add --no-cache <package>
```

---

### **4.4 Timezone Issues**

Fix:

```
apk add tzdata
```

---

## **5. Troubleshooting / Fixes**

### **Check vulnerabilities in base image**

```
trivy image node:20-alpine
```

### **Compare sizes**

```
docker images
```

### **Check missing shared libraries**

```
ldd <binary>
```

---

## **6. Best Practices / Tips**

### ✔ Prefer Alpine for general lightweight applications

### ✔ Prefer Distroless for highly secure workloads

### ✔ Avoid heavy images (Ubuntu, full Debian) unless necessary

### ✔ Use multi-stage builds for further size reduction

### ✔ Always run containers as non-root

### ✔ Scan minimal images regularly with Trivy

### ✔ Use slim variants when Alpine causes compatibility issues

---
---

# Docker Security: SBOM & Supply Chain Basics — Detailed Explanation Notes

---

## **1. Concept / What**

### **SBOM (Software Bill of Materials)**

An **SBOM** is a detailed inventory of all components inside a Docker image, including:

* OS packages (apk, apt, yum)
* Application dependencies (npm, pip, Maven, Go modules)
* Version numbers
* Licenses
* Hashes
* Transitive/indirect dependencies

SBOM acts like an **"ingredients list"** for your container image.

---

### **Supply Chain Security**

The software supply chain includes everything that contributes to building an image:

* Base images
* Libraries
* CI/CD tools
* Build artifacts
* Registries (Docker Hub, ECR)
* Dependencies pulled from internet

Supply chain security ensures all components are:

* Trusted
* Authenticated
* Verified
* Vulnerability-free
* Traceable

---

## **2. Why / Purpose / Real-World Use Cases**

### **2.1 Vulnerability Tracking (Now & Future)**

Even if an image is clean today, future CVEs can be detected by scanning its SBOM.

### **2.2 Enterprise Compliance**

Required in:

* Banking
* FinTech
* Healthcare
* Government

### **2.3 Audit & Transparency**

Security teams use SBOMs to understand what exactly is inside production images.

### **2.4 Detecting Malicious or Unknown Components**

SBOM reveals:

* Hidden libraries
* Malicious dependencies
* Vulnerable packages

### **2.5 CI/CD Traceability**

Each release can include an SBOM artifact for long-term tracking.

---

## **3. How It Works / Steps / Syntax**

Tools used:

* **Syft** (most common SBOM generator)
* **Trivy** (can generate + scan SBOM)
* **Grype** (vulnerability scanner for SBOM)

---

### **3.1 Generate SBOM Using Syft**

Install:

```
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh
```

Generate SBOM (CycloneDX format):

```
syft myapp:latest -o cyclonedx-json > sbom.json
```

---

### **3.2 Generate SBOM Using Trivy**

```
trivy image --format cyclonedx --output sbom.json myapp:latest
```

---

### **3.3 Scan SBOM for Vulnerabilities (Post-build)**

With Trivy:

```
trivy sbom sbom.json
```

With Grype:

```
grype sbom:sbom.json
```

SBOM scanning helps detect new vulnerabilities even after the image is deployed.

---

## **4. Common Issues / Errors**

### **4.1 Outdated SBOM**

If dependencies change but SBOM is not regenerated → inaccurate information.

### **4.2 SBOM Generated But Never Scanned**

SBOM alone does not detect vulnerabilities. It must be scanned.

### **4.3 Storing SBOM Inside Source Repo**

Best practice is to store SBOM in:

* S3 bucket
* Artifactory
* Nexus
* Jenkins artifacts

### **4.4 Too Many Transitive Dependencies**

SBOM may reveal libraries installed indirectly via npm/pip.

---

## **5. Troubleshooting / Fixes**

### **Check SBOM Format**

Use CycloneDX for best tool compatibility.

### **Validate SBOM Content**

Open the JSON file to inspect components:

```
cat sbom.json | jq .
```

### **Scan SBOM Regularly**

Schedule weekly scans for new vulnerabilities.

### **Regenerate SBOM on Every Release**

Ensures accuracy and traceability.

---

## **6. Best Practices / Tips**

### ✔ Generate SBOM for all production images

### ✔ Store SBOM externally (S3, Artifactory, Nexus)

### ✔ Use Syft for SBOM generation

### ✔ Use Trivy/Grype for SBOM scanning

### ✔ Use CycloneDX SBOM format

### ✔ Don’t store SBOM inside the Docker image

### ✔ Combine SBOM scanning with Trivy image scanning

### ✔ Re-generate SBOM when dependencies change

---
---
---


