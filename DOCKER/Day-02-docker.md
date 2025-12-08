# Docker Images: Building Images (Detailed Explanation)

## **Concept / What**

A Docker image is a read-only template that contains the application code, dependencies, OS layers, and runtime configuration. Building an image refers to converting a Dockerfile into a fully packaged, runnable blueprint using the `docker build` command.

---

## **Why / Purpose / Real-World Use Cases**

* **CI/CD Pipelines:** Jenkins/GitHub Actions build images on each commit and push them to registries (Docker Hub/ECR) for deployment.
* **Microservices:** Each service is packaged as an independent image.
* **Immutable Infrastructure:** Same image runs across dev → QA → prod without differences.
* **Production Rollbacks:** Old image versions can be deployed instantly.
* **Local-to-Prod Parity:** Same image ensures consistent environments.

---

## **How It Works / Steps / Syntax**

### **1. Dockerfile Example**

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

### **2. Build the Image**

```bash
docker build -t myapp:1.0 .
```

* `-t` → Tag name
* `.` → Build context

### **3. Verify Images**

```bash
docker images
```

### **4. Run the Container**

```bash
docker run -d -p 5000:5000 myapp:1.0
```

---

## **Common Issues / Errors**

* **COPY failures (wrong path):** `no such file or directory`.
* **Permission issues:** File not executable, wrong user.
* **Large image size:** Using heavy base images.
* **Dockerfile missing:** Running build from incorrect directory.

---

## **Troubleshooting / Fixes**

* **Fix wrong paths:** Ensure correct build context and file locations.
* **Reduce size:** Use slim/alpine images where possible.
* **Debug layers:**

```bash
docker build --progress=plain .
```

* **Check container shell:**

```bash
docker run -it <image> bash
```

* **Adjust permissions:**

```dockerfile
RUN chmod +x app.py
```

---

## **Best Practices / Tips**

* Use `slim`/`alpine` images when safe.
* Use multi-stage builds for optimization.
* Add `.dockerignore` to avoid copying unnecessary files.
* Run containers as a non-root user.
* Tag images with version numbers or commit hashes.
* Avoid using `latest` in production.
* Prepare images for EKS: small size, proper entrypoint, no hardcoded secrets.

---
---

# Docker Images: Tagging Convention & Versioning (Detailed Explanation)

## **Concept / What**
---
# Docker Image Tag — Definition (Standalone Note)

## **Concept / What**

A Docker image tag is a **human‑readable label** that points to a specific, immutable version of a Docker image (internally identified by its digest). It allows you to reference, version, and promote images easily across different environments.

---

## **Quick Examples**

```
myapp:1.0
myapp:latest
myapp:build-21
123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:prod
```

---

## **Best Practice**

Use clear, immutable tags like semantic versions, commit hashes, or build numbers instead of `latest`.

---


## **Why / Purpose / Real-World Use Cases**

* **Version control:** Semantic versions (1.0.0), build numbers, commit hashes.
* **CI/CD Pipelines:** Jenkins/GitHub Actions generate tags automatically for builds.
* **Environment separation:** Tags like `dev`, `qa`, `prod` ensure correct image promotion.
* **Rollbacks:** Re-deploy an older tag easily.
* **Registry specificity:** ECR/Docker Hub/GCR each require unique tag formats.
* **Kubernetes Deployments:** K8s must always reference the image using the registry-tag format.

---

## **How It Works / Steps / Syntax**

### **1. Tagging During Build**

```bash
docker build -t myapp:1.0 .
```

### **2. Adding a New Tag to an Existing Image**

```bash
docker tag myapp:1.0 myapp:latest
```

### **3. Tagging for Docker Hub**

```bash
docker tag myapp:1.0 jagga/myapp:1.0
```

### **4. Tagging for AWS ECR**

AWS provides this URL automatically:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp
```

Tag the image for ECR:

```bash
docker tag myapp:1.0 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

### **5. Push to ECR**

```bash
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

### **6. Kubernetes Deployment (must use ECR tag)**

```yaml
containers:
- name: myapp
  image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

---

## **Common Issues / Errors**

* **Using only local tags:** K8s fails with `ImagePullBackOff`.
* **Using `latest` in production:** Hard to track versions.
* **Overwriting tags:** Replacing old versions unintentionally.
* **Wrong ECR URL format:** Push fails.
* **Tags not pushed to registry:** Image unavailable for deployment.

---

## **Troubleshooting / Fixes**

* **Verify correct tags:**

```bash
docker images
```

* **Re-tag correctly for ECR:**

```bash
docker tag <local> <ecr-url>/<repo>:<tag>
```

* **Debug pull failures in Kubernetes:** Check node IAM role and IRSA.
* **Confirm ECR repository exists:** Verify in AWS console.
* **Check ECR login:**

```bash
aws ecr get-login-password | docker login --username AWS --password-stdin <ecr-url>
```

---

## **Best Practices / Tips**

* Use **semantic versioning** for clarity.
* Maintain tags like `dev`, `qa`, `staging`, `prod`.
* Tag with **Git commit hash** for traceability.
* Avoid using `latest` in production.
* For ECR, **always** tag with the registry URL before pushing.
* Keep tags immutable to ensure reliable deployments.
* Ensure Kubernetes always references the **full ECR URL** when pulling images.

---
---

# Docker Images: Pushing & Pulling (Docker Hub / ECR) — Detailed Explanation

## **Concept / What**

Pushing a Docker image means uploading it from your local system or CI/CD pipeline to a remote image registry like Docker Hub or AWS ECR. Pulling means downloading the image from a registry to your system or Kubernetes node. Registries store and manage images for deployment across environments.

---

## **Why / Purpose / Real-World Use Cases**

* **CI/CD pipelines:** Jenkins/GitHub Actions build and push images for deployment.
* **Kubernetes deployments:** Worker nodes must pull images from a registry (cannot use local images).
* **Centralized image management:** Teams access the same tested image.
* **Environment promotion:** Dev → QA → UAT → Prod using promoted tags.
* **Rollback capability:** Older images can be redeployed instantly.
* **Scalability:** Required for auto-scaling workloads in EKS/ECS.

---

## **How It Works / Steps / Syntax**

# 🟦 Docker Hub

### **1. Login**

```bash
docker login
```

### **2. Tag the image**

```bash
docker tag myapp:1.0 jagga/myapp:1.0
```

### **3. Push the image**

```bash
docker push jagga/myapp:1.0
```

### **4. Pull the image**

```bash
docker pull jagga/myapp:1.0
```

---

# 🟥 AWS ECR

### **1. Authenticate to ECR**

```bash
aws ecr get-login-password --region ap-south-1 |
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

### **2. Tag the image with ECR URL**

```bash
docker tag myapp:1.0 \
123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

### **3. Push to ECR**

```bash
docker push \
123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

### **4. Pull from ECR**

```bash
docker pull \
123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

---

## **Common Issues / Errors**

* **Authentication errors:** `no basic auth credentials` — ECR login missing.
* **Access denied:** IAM role does not have `ecr:*` permissions.
* **Image not found:** Wrong tag or image not pushed.
* **Repository missing:** ECR repo not created.
* **Kubernetes ImagePullBackOff:** Worker nodes unable to pull due to incorrect URL or missing auth.

---

## **Troubleshooting / Fixes**

* **Verify repo exists:**

```bash
aws ecr describe-repositories
```

* **Verify tag exists:**

```bash
aws ecr describe-images --repository-name myapp
```

* **Re-authenticate:**

```bash
aws ecr get-login-password | docker login
```

* **Inspect Docker images locally:**

```bash
docker images | grep myapp
```

* **Check K8s pod events:**

```bash
kubectl describe pod <podname>
```

---

## **Best Practices / Tips**

* Use descriptive, immutable tags (e.g., `1.0.3`, `build-21`, `commit-a1b2c3`).
* Avoid using `latest` in production environments.
* Automate push/pull in CI/CD.
* For ECR, always tag with the full ECR registry URL before pushing.
* Ensure EKS uses the exact same ECR tag in Deployment YAML.
* Use IRSA to give Kubernetes service accounts access to pull images securely.
* Maintain separate tags for promotion across environments (e.g., `qa-passed`, `uat-passed`, `prod-approved`).

---
---

# Docker Images: Inspecting Images — Detailed Explanation

## **Concept / What**

Inspecting a Docker image means retrieving detailed metadata about the image such as layers, environment variables, commands, entrypoints, working directory, ports, and configuration details. The `docker inspect` and `docker history` commands show this internal structure in JSON format, helping understand how the image is built and how containers will behave when run.

---

## **Why / Purpose / Real-World Use Cases**

* **Debugging container failures:** Identify incorrect CMD/ENTRYPOINT, wrong paths, or missing files.
* **Reviewing image configuration:** Ensure correct ports, environment variables, and working directory.
* **Security validation:** Confirm if the image runs as non-root and check labels.
* **CI/CD verification:** Validate metadata of images built in Jenkins.
* **Kubernetes troubleshooting:** Check image configuration when pods fail to start.
* **Layer analysis:** Understand how each Dockerfile instruction contributed to the final image.

---

## **How It Works / Steps / Syntax**

### **1. Basic Inspect Command**

```bash
docker inspect myapp:1.0
```

### **2. Inspect Using Full ECR URL**

```bash
docker inspect 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

### **3. Filtering Output with `--format`**

#### **Get ENTRYPOINT**

```bash
docker inspect --format='{{.Config.Entrypoint}}' myapp:1.0
```

#### **Get CMD**

```bash
docker inspect --format='{{.Config.Cmd}}' myapp:1.0
```

#### **Get Environment Variables**

```bash
docker inspect --format='{{.Config.Env}}' myapp:1.0
```

#### **Get Working Directory**

```bash
docker inspect --format='{{.Config.WorkingDir}}' myapp:1.0
```

### **4. Inspect Image Layers**

```bash
docker history myapp:1.0
```

Shows each layer, what instruction created it, and the size.

---

## **Common Issues / Errors**

* **Image not found:** Wrong tag or the image wasn’t pulled locally.
* **Inspecting remote image directly:** Requires pulling first.
* **Incorrect ENTRYPOINT:** Containers exit immediately.
* **Wrong working directory:** Application files not found at runtime.
* **Large layers:** Inefficient Dockerfile causing image bloat.

---

## **Troubleshooting / Fixes**

* **Pull the image before inspecting:**

```bash
docker pull <image>
```

* **Debug ENTRYPOINT/CMD issues:**

```bash
docker inspect <image> | grep -i entrypoint
```

* **Fix wrong paths:** Review `.Config.WorkingDir` and adjust Dockerfile.
* **Optimize image size using layer history:**

```bash
docker history <image>
```

Identify heavy layers and optimize accordingly.

---

## **Best Practices / Tips**

* Inspect images before promoting them to QA/UAT/Prod.
* Review metadata for correct CMD, ports, and environment variables.
* Use `docker history` to improve Dockerfile structure and optimize size.
* Add meaningful labels in Dockerfile (visible in `docker inspect`).
* Use `--format` to extract only the required values in automation scripts.

---
---

# Docker Images: Image History & Layers — Detailed Explanation

## **Concept / What**

Docker images are built in a layered architecture where each Dockerfile instruction creates a new immutable layer. These layers stack on top of each other to form the final image. Commands like `docker history` and `docker inspect` allow you to view the details of these layers, including the commands that created them, their sizes, and how they contribute to caching.

---

## **Why / Purpose / Real-World Use Cases**

* **Image optimization:** Identify large or unnecessary layers that increase image size.
* **Caching:** Faster builds by reusing unchanged layers.
* **Troubleshooting:** Understand which Dockerfile step causes failures.
* **Security audits:** Identify outdated or insecure base layers.
* **Kubernetes performance:** Smaller images reduce pull time and speed up deployments.
* **CI/CD speed:** Optimized layering improves build time across environments.

---

## **How It Works / Steps / Syntax**

### **1. View Image History**

```bash
docker history myapp:1.0
```

Shows: commands, layer sizes, timestamps.

### **2. Inspect Layer Details**

```bash
docker inspect myapp:1.0
```

Check the `RootFS.Layers` section for underlying layer digests.

### **3. Example of Layer Creation in Dockerfile**

```dockerfile
FROM python:3.10-slim        # Layer 1
WORKDIR /app                  # Metadata layer
COPY requirements.txt .       # Layer 2
RUN pip install -r requirements.txt   # Layer 3
COPY . .                      # Layer 4
CMD ["python", "app.py"]     # Metadata layer
```

Every command that modifies the filesystem becomes a separate layer.

### **4. Layer Caching Behavior**

* If a layer changes → that layer and all layers after it rebuild.
* Layers before the change are reused from cache.
* Frequently changing steps should be placed at the bottom of the Dockerfile.

---

## **Common Issues / Errors**

* **Large images:** Caused by improper layering or large base images.
* **Cache invalidation:** Caused by placing frequently changing commands early.
* **Unoptimized Dockerfile:** Too many small layers or multiple RUN commands.
* **Slow CI/CD builds:** Missing caching due to incorrect ordering.

---

## **Troubleshooting / Fixes**

* **Reduce image size:** Use slim/alpine images where possible.
* **Combine RUN commands:**

```dockerfile
RUN apt update && apt install -y curl python3 && apt clean
```

* **Place dependency installation before copying source code.**
* **Use `.dockerignore`** to exclude unnecessary files:

```
node_modules
.git
.vscode
```

* **Use multi-stage builds** for Go, Node, Java, Python apps.

---

## **Best Practices / Tips**

* Put frequently changing instructions (e.g., `COPY . .`) **at the bottom**.
* Put rarely changing instructions (dependencies, OS setup) **at the top**.
* Choose minimal base images (`slim`, `alpine`) when possible.
* Combine multiple RUN commands into one to reduce layers.
* Inspect layers regularly with `docker history` after Dockerfile updates.
* Keep Dockerfile clean, predictable, and optimized for caching.

---
---

# Docker Images: Understanding Registries — Detailed Explanation

## **Concept / What**

A Docker registry is a centralized platform that stores, manages, versions, and distributes Docker images. Registries contain repositories, and each repository holds multiple tagged versions of images. Registries can be public (Docker Hub) or private (AWS ECR, Nexus, JFrog, Harbor).

---

## **Why / Purpose / Real-World Use Cases**

* **CI/CD Pipelines:** Jenkins/GitHub Actions push built images to registries for deployment.
* **Kubernetes Deployments:** Worker nodes pull images only from registries, not from local machines.
* **Environment Promotion:** Identical images are promoted using tags across Dev → QA → UAT → Prod.
* **Security:** Authentication, IAM, scanning, access policies, and audit logs.
* **Scalability:** Registries allow large-scale systems (EKS/ECS) to pull images quickly.
* **Version Control:** Multiple versions of the same image are stored with unique tags.

---

## **How It Works / Steps / Syntax**

### **1. Registry URL Format**

```
<registry-url>/<repository-name>:<tag>
```

### **2. Docker Hub Example**

```
jagga/myapp:1.0
```

Push:

```bash
docker push jagga/myapp:1.0
```

Pull:

```bash
docker pull jagga/myapp:1.0
```

---

### **3. AWS ECR Example**

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

Push:

```bash
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

Pull:

```bash
docker pull 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

---

### **4. Self-Hosted Registry Examples**

#### JFrog Artifactory

```
artifactory.company.com/docker/myapp:1.0
```

#### Nexus

```
nexus.company.com/repository/docker-hosted/myapp:1.0
```

#### Harbor

```
harbor.company.com/library/myapp:1.0
```

---

## **Common Issues / Errors**

* **Authentication failures:** Missing login or incorrect credentials.
* **Access denied:** IAM or registry permissions missing.
* **Repository not found:** Registry repository not created.
* **Image not found:** Wrong tag or image not pushed.
* **Docker Hub rate limits:** Pull limits exceeded.
* **Region/account mismatch in ECR:** Using wrong URL.
* **Kubernetes ImagePullBackOff:** Pull failures due to wrong image URL or authentication issues.

---

## **Troubleshooting / Fixes**

* **Verify repository exists (ECR):**

```bash
aws ecr describe-repositories
```

* **Check available tags:**

```bash
aws ecr describe-images --repository-name myapp
```

* **Re-authenticate:**

```bash
docker login
```

* **Fix Kubernetes pull issues:** Ensure correct registry URL and authentication (IRSA).
* **Use correct region/account for ECR.**
* **Create Docker registry secrets for non-ECR registries.**

---

## **Best Practices / Tips**

* Prefer **AWS ECR** for AWS workloads (EKS/ECS/Lambda).
* Tag images with semantic versions and promotion tags:

  * `1.0.3`
  * `qa-passed`
  * `uat-passed`
  * `prod-approved`
* Enable image scanning for vulnerability detection.
* Use least-privileged access (IAM roles for pulling images).
* Use lifecycle policies to clean old images.
* Avoid using `latest` in production.
* Structure repositories per microservice for clarity.

---
---

# Docker Build Context (Detailed Explanation)

## **Concept / What**

The **build context** is the **entire directory** sent to the Docker daemon when you run:

```
docker build .
```

The `.` means:

> Send **all files and folders** from the current directory to the Docker daemon.

Dockerfile instructions like `COPY` and `ADD` can only access files **inside this build context**.

---

## **Why It Matters**

The size and content of the build context directly affect:

* Build speed
* CI/CD performance
* Image size
* Security
* Layer caching efficiency

---

## **Problems If Build Context Is Too Large**

* **Slow builds** because Docker uploads the entire context to the daemon.
* **Unnecessary files** (node_modules, git history, logs) slow down image creation.
* **Accidental inclusion of sensitive files** (keys, credentials).
* **Inefficient caching** because large contexts cause more frequent cache invalidation.
* **Large images** if unwanted files get copied inside.

---

## **How to Prevent Issues**

Use a `.dockerignore` file to exclude unnecessary files from the build context.

### Example `.dockerignore`:

```
node_modules
.git
.gitignore
logs/
*.pem
*.env
.vscode
.DS_Store
```

### Best Practices

* Keep the build context **minimal and clean**.
* Place your Dockerfile at the root of the application folder.
* Exclude large or irrelevant directories.
* Never include secrets or credentials inside the build context.

---

## **Quick Summary**

* Build context = directory sent to daemon.
* Large context = slow builds + security risk.
* Solution = `.dockerignore`.

---
---
---


