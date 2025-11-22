# Docker Concept 1: What is Docker & Why It’s Used (Detailed Explanation Notes)

## **Concept / What**

Docker is a **containerization platform** that packages an application along with its **libraries, runtimes, and dependencies** into a portable unit called a **container image**. A container is the **runtime instance** of this image, running as an isolated process using the host's kernel. Docker images are **immutable**, and containers provide consistent execution across all environments.

---

## **Why / Purpose / Real-World Use Cases**

### **Why Docker is Used**

* Ensures **environment consistency** across dev → test → prod.
* Eliminates "works on my machine" issues.
* Provides **lightweight** runtime by sharing the host OS kernel.
* Enables **faster, reproducible CI/CD pipelines**.
* Simplifies microservices deployment on **EKS/Kubernetes**.
* Supports **scaling and portability** across clouds and on-prem servers.

### **Real DevOps Use Cases**

* Building application images through Jenkins pipelines.
* Running microservices in Kubernetes using Docker-based images.
* Creating isolated developer environments without installing heavy runtimes.
* Rolling back or deploying new app versions using immutable images.
* Standardizing build environments across teams.

---

## **How It Works / Steps / Syntax**

### **Basic Docker Workflow**

1. **Pull an image**:

```bash
docker pull nginx
```

2. **Run a container**:

```bash
docker run -d -p 8080:80 nginx
```

* `-d` → detached mode
* `-p` → port mapping

3. **List running containers**:

```bash
docker ps
```

4. **Stop a container**:

```bash
docker stop <container-id>
```

5. **Remove a container**:

```bash
docker rm <container-id>
```

6. **Remove an image**:

```bash
docker rmi nginx
```

### **High-Level Flow**

* Developer writes application code.
* DevOps writes the **Dockerfile** using the correct runtime versions.
* Jenkins builds the image → tags → pushes to registry (ECR/Docker Hub).
* Kubernetes pulls the image → runs the container.

---

## **Common Issues / Errors**

### **1. Permission Denied (Docker Daemon)**

**Error:** `docker: permission denied`
**Fix:**

```bash
sudo usermod -aG docker $USER
```

### **2. Port Already in Use**

**Error:** "Bind for 0.0.0.0:8080 failed: port already allocated"
**Fix:**

```bash
sudo lsof -i :8080
sudo kill <pid>
```

### **3. Cannot Connect to Docker Daemon**

**Fix:**

```bash
sudo systemctl start docker
```

### **4. Image Size Too Large**

**Fixes:**

* Use **Alpine** or **slim** base images.
* Use **multi-stage builds**.
* Add `.dockerignore` to reduce build context.

### **5. Container Crashing Immediately**

Common reasons:

* Wrong entrypoint/command
* Missing dependencies
* Application errors inside the container

Fix by checking logs:

```bash
docker logs <container-id>
```

---

## **Troubleshooting / Fixes**

* Use `docker inspect` to debug container metadata.
* Use `docker exec -it <id> sh` to open a shell inside container.
* Validate Dockerfile syntax using small incremental builds.
* Use health checks to detect container readiness.
* Use ECR/Docker Hub authentication for private repositories.

---

## **Best Practices / Tips**

* Always use **pinned versions** (avoid `latest`).
* Use **multi-stage builds** to reduce image size.
* Run applications as a **non-root user**.
* Use `.dockerignore` to exclude unneeded files.
* Use lightweight base images (`alpine`, `slim`).
* Keep Dockerfile instructions minimal and ordered for caching.
* Keep runtime images minimal (no build tools inside final image).
* Use proper tagging strategy (`v1`, `v1.1`, `commit-sha`).
* Ensure images are compatible with Kubernetes/EKS (health checks, ports, users).

---
---

# Docker Concept 2: Containers vs Virtual Machines (Detailed Explanation Notes)

## **Concept / What**

Comparison of **Containers** and **Virtual Machines**, focusing on OS architecture, resource allocation, isolation, performance, and real DevOps use cases.

---

# 🧩 **Containers vs Virtual Machines — Column-wise Comparison**

| **Category**             | **Containers**                                                                             | **Virtual Machines (VMs)**                                                                |
| ------------------------ | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| **Definition**           | Lightweight, isolated processes created from Docker images and sharing the host OS kernel. | Full OS virtualization with its own kernel running on a hypervisor.                       |
| **OS Architecture**      | Share **host kernel**; have their own user space + filesystem from image layers.           | Have **full guest OS + kernel** separate from host OS.                                    |
| **Startup Time**         | Seconds (no OS boot).                                                                      | Minutes (OS boots fully).                                                                 |
| **Resource Allocation**  | Dynamic; resources assigned via cgroups/requests/limits and reclaimed easily.              | Static; vCPU/RAM allocated when VM is created; harder to reclaim.                         |
| **Resource Usage**       | Very low (no guest OS overhead).                                                           | High (full OS consumes CPU/RAM).                                                          |
| **Isolation Type**       | Process-level isolation via namespaces/cgroups.                                            | Full OS-level isolation via hypervisor.                                                   |
| **Portability**          | Highly portable across environments; ideal for CI/CD & microservices.                      | Less portable; tied to hypervisors/cloud providers.                                       |
| **Filesystem**           | Ephemeral container FS unless using volumes/PVCs.                                          | Persistent virtual disks (EBS, VMDK, qcow2).                                              |
| **Security**             | Lighter isolation; needs runtime hardening (non-root user, seccomp, AppArmor).             | Stronger isolation; ideal for sensitive workloads.                                        |
| **Performance**          | Near-native performance.                                                                   | Slight overhead due to hypervisor + guest OS.                                             |
| **Scalability**          | Easy horizontal scaling (replicas).                                                        | Harder, slower scaling; requires provisioning full VMs.                                   |
| **Kernel Customization** | Cannot change kernel (shared with host).                                                   | Full control over kernel & OS-level tuning.                                               |
| **Lifecycle**            | Short-lived, immutable, disposable.                                                        | Long-lived, stable, stateful.                                                             |
| **Use Cases**            | Stateless apps, microservices, CI/CD agents, API services, ephemeral jobs.                 | Databases, legacy apps, stateful workloads, kernel tuning required, compliance workloads. |
| **Storage Handling**     | Volumes/PVCs through orchestrators.                                                        | Direct persistent disks; stronger durability.                                             |
| **Orchestration**        | Kubernetes/EKS for scaling, rolling updates, health checks.                                | Managed via cloud autoscaling or hypervisor tools.                                        |
| **Image/Template Size**  | MB-level images (e.g., Alpine 5MB).                                                        | GB-level OS templates.                                                                    |
| **Deployment Speed**     | Extremely fast; ideal for rapid CI/CD.                                                     | Slower; provisioning may take minutes.                                                    |

---

## **Why / Purpose / Real-World Use Cases**

### **Containers**

* Ideal for microservices and stateless apps.
* Used in CI/CD pipelines for fast reproducible environments.
* Essential for Kubernetes/EKS deployments.
* Efficient resource usage and rapid scaling.

### **Virtual Machines**

* Used where full OS control is needed.
* Ideal for production databases and stateful apps.
* Necessary for legacy workloads.
* Better isolation/security boundary.

---

## **How It Works / Steps**

### Containers

* Built from Docker images.
* Share host kernel.
* Scheduled dynamically; resources applied via cgroups.
* Best managed with Kubernetes.

### VMs

* Created by hypervisor (KVM, ESXi, Hyper-V).
* Boot full OS.
* Resources allocated at creation.
* Managed via cloud (EC2, Compute Engine) or hypervisor.

---

## **Common Issues / Errors**

### Containers

* Crash due to missing deps or wrong entrypoint.
* Port conflicts.
* Data loss without volumes.
* Image bloat.

### VMs

* Performance issues due to over-allocation.
* Slow boot times.
* OS-level patching required.
* Storage latency under heavy load.

---

## **Troubleshooting / Fixes**

### Containers

* `docker logs` for crash analysis.
* `docker inspect` for metadata.
* Use volumes for persistence.
* Use resource limits/requests.

### VMs

* Resize instance types.
* Tune kernel parameters.
* Use optimized storage.
* Apply hypervisor-level performance adjustments.

---

## **Best Practices / Tips**

### Containers

* Keep images lightweight (Alpine, slim).
* Use non-root user.
* Use multi-stage Dockerfiles.
* Avoid storing secrets inside images.
* Always use proper tagging.
* Deploy via orchestrators (K8s/EKS).

### VMs

* Use for persistent databases.
* Patch OS regularly.
* Use minimal OS builds (Amazon Linux minimal).
* Allocate resources based on usage patterns.

---
---

# Docker Concept 3: Docker Architecture (High-Level + Docker Engine Internal Architecture)

## **Concept / What**

Docker architecture is understood at **two levels**:

1. **High-Level Docker Architecture** → how Docker works for users (client/host/registry).
2. **Docker Engine Internal Architecture** → how Docker actually runs containers internally (dockerd/containerd/runc).

Both are correct and represent different layers of the same system.

---

# 🧩 **1. High-Level Docker Architecture (Client → Host → Registry)**

This is the standard architecture taught in DevOps training.

## **1. Docker Client**

* CLI tool used by users to run Docker commands.
* Sends commands to the Docker daemon.
* Examples:

```bash
docker run
docker build
docker pull
```

## **2. Docker Host**

Contains:

* Docker Daemon (**dockerd**)
* Images
* Containers
* Networks
* Volumes

**Docker Host = Machine where Docker Engine runs.**

## **3. Docker Registry**

* Stores and distributes Docker images.
* Examples:

  * Docker Hub
  * AWS ECR
  * Nexus / Artifactory
  * GCR / ACR

**Flow:**

```
Docker Client → Docker Host → (pull/push) → Registry
```

---

# 🧩 **2. Docker Engine Internal Architecture (dockerd → containerd → runc)**

This is the deeper, professional-level architecture used in real DevOps/Kubernetes work.

## **1. dockerd (Docker Daemon)**

* Main background service for Docker Engine.
* Handles Docker CLI requests.
* Manages:

  * Images
  * Containers
  * Networks
  * Volumes
* Communicates with **containerd** to run containers.

## **2. containerd (Container Daemon / Manager)**

* High-level container manager.
* Responsible for:

  * Pulling images
  * Creating containers
  * Managing container lifecycle
* dockerd depends on containerd internally.
* Kubernetes uses containerd **directly**.

## **3. runc (Low-Level Runtime)**

* Actually creates containers.
* Applies:

  * Namespaces (process isolation)
  * Cgroups (CPU/memory limits)
  * Root filesystem layers
* Called by containerd.

---

# 🔗 **How Both Architectures Map Together**

| Training Architecture | Internal Architecture                               |
| --------------------- | --------------------------------------------------- |
| **Docker Client**     | `docker` CLI                                        |
| **Docker Host**       | `dockerd + containerd + runc + images + containers` |
| **Docker Daemon**     | `dockerd` (part of host)                            |
| **Registry**          | Docker Hub, ECR, etc.                               |

Both are correct — they represent **different layers of detail**.

---

# ⚙️ **How Docker Creates a Container (Full Flow)**

```
Docker CLI → dockerd → containerd → runc → container starts
```

* dockerd receives the command.
* containerd manages download and creation.
* runc applies namespaces/cgroups and launches the process.

---

# 🧩 **Why Kubernetes Removed dockerd**

* Kubernetes interacts only with **CRI (Container Runtime Interface)**.
* containerd supports CRI directly.
* dockerd does NOT support CRI (needed dockershim before).
* So Kubernetes removed dockershim and now uses:

```
kubelet → containerd → runc
```

---

# 🛠️ **Common Issues / Errors**

### **dockerd not running**

```
sudo systemctl start docker
```

### **containerd crash (affects Kubernetes nodes)**

```
sudo systemctl restart containerd
```

### **runc corrupted or missing**

```
sudo apt reinstall runc
```

---

# 🧰 **Troubleshooting Commands**

```bash
systemctl status docker
systemctl status containerd
docker info
docker inspect <container>
docker logs <container>
```

---

# ⭐ **Best Practices**

* Use containerd for Kubernetes nodes.
* Keep Docker Engine updated.
* Use OCI-compliant images.
* Use lightweight base images.
* Avoid unnecessary layers in Dockerfile.
* Store images in secure registries (ECR, Artifactory).

---
---

# Docker Concept 4: Docker Images, Layers, and Caching (Detailed Explanation Notes)

## **Concept / What**

A Docker image is an **immutable, read-only template** used to create containers. It contains the application, runtime, libraries, and dependencies. A Docker image is built as a stack of **layers**, where each Dockerfile instruction creates a new layer. Docker uses **caching** to reuse unchanged layers during future builds, speeding up CI/CD pipelines and reducing storage.

---

# 🧩 **Images, Layers, and Caching — Comparison Summary**

| Component | Description                                                                 |
| --------- | --------------------------------------------------------------------------- |
| **Image** | Immutable blueprint used to run containers; built from layered filesystem.  |
| **Layer** | Each Dockerfile instruction becomes a layer; layers stack to form an image. |
| **Cache** | Docker reuses unchanged layers to avoid rebuilding them.                    |

---

# 🧱 **How Layers Work**

* Each Dockerfile instruction (`FROM`, `RUN`, `COPY`, etc.) creates a **new layer**.
* Layers are **read-only** and stored on the host.
* Layers are **shared** across different images if identical.
* The final image is a **stack of layers**.

### Example Layer Breakdown

```dockerfile
FROM node:18-alpine    # Layer 1
WORKDIR /app           # Layer 2
COPY package*.json .   # Layer 3
RUN npm install        # Layer 4
COPY . .               # Layer 5
CMD ["npm", "start"]   # Metadata (not a layer)
```

---

# 🔁 **How Docker Caching Works**

Docker caches layers based on two rules:

1. **Instruction must be identical**
2. **Content used in the instruction must be identical**

If both match → Docker reuses the cached layer.
If either changes → Docker rebuilds that layer and all layers **below** it.

### ✔️ Base image identical → cache works

### ❌ Base image changed → entire image rebuilds

---

# 🧩 **Examples of Cache Behavior**

## **1. Base image same, one RUN changed → partial rebuild**

```dockerfile
FROM ubuntu:20.04        # Cached
RUN apt update           # Cached
RUN apt install git      # Changed → Rebuild
COPY . .                 # After change → Rebuild
```

## **2. Base image changed → full rebuild**

```dockerfile
FROM ubuntu:22.04        # New → Cache broken
RUN apt update           # Rebuild
RUN apt install curl     # Rebuild
COPY . .                 # Rebuild
```

## **3. Adding a new line right after base → full rebuild**

If you insert a new instruction after `FROM`, all following layers rebuild.

---

# 🧩 **Cross-Image Layer Reuse**

Docker reuses layers **across different images** if:

* The instructions are the same
* The content is the same
* The order is the same

### Example:

If both images start with:

```
FROM python:3.10-slim
```

The base layer is shared for all images.

---

# ⚙️ **How It Works Internally**

* Docker stores layers by their **content hash**.
* If a layer with the same hash already exists, caching is used.
* Kubernetes nodes also reuse layers across many pods.

---

# 🛠️ **Common Issues / Errors**

### **1. Large image size**

**Causes:** heavy base image, unnecessary files, missing `.dockerignore`.
**Fixes:** use Alpine/slim images, multi-stage builds, `.dockerignore`.

### **2. Cache invalidation**

Frequent changes in early instructions ruin caching.
**Fix:** Put dependency steps early and application COPY later.

### **3. Slow builds**

Caused by poor Dockerfile structuring.
**Fix:** Proper layer ordering and caching strategy.

### **4. Wrong base image**

Using `latest` creates inconsistent builds.
**Fix:** Always use pinned versions like `python:3.10-slim`.

---

# 🧰 **Troubleshooting Commands**

```bash
docker history <image>
docker inspect <image>
docker system prune
docker builder prune
```

---

# ⭐ **Best Practices**

* Use lightweight base images (`alpine`, `slim`).
* Place frequently-changing instructions (COPY . .) **at the bottom**.
* Place dependency installation (npm install, pip install) early.
* Avoid using `latest` tag.
* Use multi-stage builds.
* Always use `.dockerignore`.
* Combine RUN commands where possible.
* Keep the Dockerfile small and clean.

---
---

# Docker Concept 5: Container Lifecycle (create / start / stop / remove)

## **Concept / What**

The Docker container lifecycle represents the different stages a container goes through from creation to deletion. Docker has a **simple, four-stage lifecycle**, unlike Kubernetes which has multiple pod phases. The four Docker lifecycle stages are:

* **create** → container object created but not running
* **start** → container begins execution
* **stop** → container process stops
* **remove (rm)** → container deleted from host

Containers in Docker behave like lightweight, isolated processes.

---

# 🧩 **Docker vs Kubernetes Lifecycle (Quick Comparison)**

| Feature        | Docker Lifecycle                    | Kubernetes Pod Lifecycle                                                              |
| -------------- | ----------------------------------- | ------------------------------------------------------------------------------------- |
| **Stages**     | create, start, stop, remove         | Pending, ContainerCreating, Running, Succeeded, Failed, CrashLoopBackOff, Terminating |
| **Managed By** | Docker Engine                       | Kubelet + Scheduler + Controllers                                                     |
| **Scope**      | Single container                    | Pod (group of containers)                                                             |
| **Complexity** | Simple (local container management) | Complex (cluster orchestration)                                                       |
| **Purpose**    | Run a container                     | Run + schedule + monitor + restart containers in cluster                              |

---

# 🧱 **Docker Container Lifecycle Phases**

## **1️⃣ Create**

Creates a stopped container from an image.

```bash
docker create --name myapp nginx
```

* Container filesystem created
* Metadata stored
* Not running yet

---

## **2️⃣ Start**

Starts execution of the container.

```bash
docker start myapp
```

Or create + start in one step:

```bash
docker run --name myapp nginx
```

* runc creates the container process
* containerd manages lifecycle

---

## **3️⃣ Stop**

Gracefully stops container.

```bash
docker stop myapp
```

* Sends SIGTERM, then SIGKILL if needed

Force stop:

```bash
docker kill myapp
```

---

## **4️⃣ Remove (rm)**

Deletes container from host.

```bash
docker rm myapp
```

Remove running container:

```bash
docker rm -f myapp
```

Prune all stopped containers:

```bash
docker container prune
```

---

# 🧩 **Lifecycle Flow**

```
docker create → docker start → docker stop → docker rm
```

Or commonly:

```
docker run → docker stop → docker rm
```

---

# 🧩 **Real DevOps Scenarios**

### **CI/CD Pipeline**

```bash
docker run -d --name test-app myapp
# run tests
docker stop test-app
docker rm test-app
```

Ensures a clean environment for every pipeline run.

### **Debugging Containers**

```bash
docker create --name debug-app myapp
docker inspect debug-app
docker start debug-app
```

Useful when container fails to start.

### **Application Update (Manual)**

* Stop old container
* Remove old container
* Start new container from updated image

Kubernetes does this automatically.

---

# 🛠️ **Common Issues / Errors**

### **1. Cannot remove container (running)**

```bash
docker rm -f <container>
```

### **2. Port already in use**

```
sudo lsof -i :<port>
sudo kill <pid>
```

### **3. Container exits immediately**

Possible reasons:

* Wrong CMD/ENTRYPOINT
* Application crashed
* No foreground process

Check logs:

```bash
docker logs <container>
```

### **4. Long shutdown time**

Use force:

```bash
docker kill <container>
```

---

# 🧰 **Troubleshooting Commands**

```bash
docker ps -a
docker logs <container>
docker inspect <container>
docker events
```

---

# ⭐ **Best Practices**

* Use `docker run --rm` for short-lived containers.
* Always clean up exited containers.
* Use meaningful container names in CI/CD.
* Prefer graceful stop before kill.
* Monitor container health using health checks.
* Avoid running too many stopped/exited containers.

---
---

# Docker Concept 6: Basic Docker Commands (Detailed Explanation Notes)

## **Concept / What**

Basic Docker commands are the foundational CLI operations used to manage Docker images, containers, logs, troubleshooting, and resource cleanup. These commands allow DevOps engineers to build, run, stop, inspect, debug, and clean up Docker environments in everyday development and CI/CD workflows.

---

# 🧩 **Important Docker Commands (Categorized)**

## **1. Version & System Info**

```bash
docker --version
docker info
```

Used to check the Docker Engine version, system details, and runtime configuration.

---

## **2. Image Management**

### Pull an image

```bash
docker pull nginx
```

### List images

```bash
docker images
```

### Remove image

```bash
docker rmi nginx
```

---

## **3. Container Lifecycle Management**

### Run container

```bash
docker run -d -p 8080:80 --name web nginx
```

* `-d`: detached mode
* `-p`: port mapping
* `--name`: custom container name

### Create container (without starting)

```bash
docker create --name test nginx
```

### Start / Stop container

```bash
docker start web
docker stop web
```

### Remove container

```bash
docker rm web
```

Force remove:

```bash
docker rm -f web
```

---

## **4. Execute Commands Inside a Container**

Run interactive shell:

```bash
docker exec -it web sh
```

(Use `bash` for Ubuntu-based images.)

---

## **5. Logs & Debugging**

View logs:

```bash
docker logs web
```

Follow logs:

```bash
docker logs -f web
```

Inspect container metadata:

```bash
docker inspect web
```

Real-time system events:

```bash
docker events
```

---

## **6. Clean Up Docker Resources**

### Remove all stopped containers

```bash
docker container prune
```

### Remove unused images

```bash
docker image prune
```

### Remove unused containers, networks, build cache

```bash
docker system prune
```

* Deletes only **unused** resources
* Does NOT delete running containers
* Does NOT delete volumes unless `--volumes` is added

Aggressive cleanup (unused images included):

```bash
docker system prune -a
```

Prune including volumes (dangerous):

```bash
docker system prune -a --volumes
```

---

## 🧩 **Real DevOps Use Cases**

* Run applications locally for testing
* Run Jenkins pipeline steps inside containers
* Debug failing apps by inspecting logs/exec
* Pull images before building in CI
* Clean up build agents using prune commands
* Start/stop/remove containers during deployments

---

# 🛠️ **Common Issues / Errors & Fixes**

### ❗ Port already in use

```bash
sudo lsof -i :8080
sudo kill <pid>
```

### ❗ Cannot remove image because a container is using it

```bash
docker rm -f <container>
docker rmi <image>
```

### ❗ Exec not working (container not running)

```bash
docker start <container>
docker exec -it <container> sh
```

### ❗ Container exits immediately

Causes:

* Wrong CMD/ENTRYPOINT
* App crashes
* No foreground process

Fix:

```bash
docker logs <container>
```

---

# ⭐ **Best Practices**

* Always name containers using `--name`.
* Use `docker run --rm` for short-lived containers.
* Clean up unused containers/images regularly.
* Use `docker logs -f` for real-time debugging.
* Use version tags (avoid `latest`).
* Use inspect for deep troubleshooting.
* Keep containers minimal and short-lived.

---
---
