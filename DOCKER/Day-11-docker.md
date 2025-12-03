# CrashLoopBackOff (Docker-Level) — Detailed Explanation Notes

## **1. Concept / What**

CrashLoopBackOff at the Docker/container level refers to a situation where a container starts, immediately crashes, and keeps restarting in a loop. Kubernetes shows this as `CrashLoopBackOff`, but the root cause always comes from the container (Docker/CRI) failing due to application-level or configuration issues.

---

## **2. Why / Purpose / Real-World Use Cases**

DevOps teams investigate Docker-level CrashLoopBackOff causes because:

* Kubernetes only displays the *symptom*, not the *root cause*.
* Application startup issues always originate from container execution.
* CI/CD pipeline failures (failed deployments) depend on correct container behavior.
* EKS workloads rely on correctly built and functioning Docker images.
* Local debugging before pushing to ECR requires container-level analysis.

Use cases:

* Microservices crashing instantly on startup.
* Incorrect Dockerfile leading to failed deployments.
* CI/CD deployments failing due to wrong ENTRYPOINT/CMD.
* Containers failing health checks and restarting repeatedly.

---

## **3. How it Works / Steps / Syntax**

Below are the core steps used to identify the underlying reason behind container startup failures.

### **Check container list and status**

```bash
docker ps -a
```

Common statuses:

* `Exited (1)` → application error
* `Exited (137)` → OOMKilled
* `Restarting (X seconds)` → crash loop

### **Check logs**

```bash
docker logs <container>
```

Helps identify:

* Missing modules
* Port binding issues
* Env variable failures
* Permission denied messages

### **Check exit code**

```bash
docker inspect <container> --format='{{.State.ExitCode}}'
```

Important exit codes:

* `1` → generic app failure
* `126` → permission denied
* `127` → command not found
* `137` → OOMKilled
* `139` → segmentation fault

### **Inspect ENTRYPOINT/CMD**

```bash
docker inspect <container> | grep -i entrypoint -A2
```

Common issues:

* Wrong path
* Missing execute permission
* Incorrect shell scripts

### **Run container interactively for debugging**

```bash
docker run -it --entrypoint /bin/sh <image>
```

Useful to verify:

* File paths
* Env variables
* Permissions

### **Verify environment variables**

```bash
docker inspect <container> | grep -i env -A10
```

Typical error:

```
Environment variable not set
```

### **Port conflicts**

```bash
sudo lsof -i -P -n | grep <port>
```

Or check running containers using that port:

```bash
docker ps --filter "publish=<port>"
```

### **Volume mount permission issues**

Example error:

```
permission denied: /app/logs
```

Fix:

```bash
sudo chown -R 1000:1000 /app/logs
```

### **Check Docker daemon logs**

```bash
sudo journalctl -u docker
```

Useful for system-level container runtime issues.

---

## **4. Common Issues / Errors**

* Application exits immediately (bad code/config)
* Wrong ENTRYPOINT or CMD
* Missing files due to incorrect Dockerfile COPY
* Permission denied (scripts not executable)
* Wrong WORKDIR path
* Port conflicts (address already in use)
* Missing or incorrect environment variables
* OOMKilled due to insufficient memory
* Non-root user issues (writing to restricted paths)
* Overlay2 storage errors preventing container startup

---

## **5. Troubleshooting / Fixes**

### **Fix ENTRYPOINT/CMD issues**

```bash
RUN chmod +x entrypoint.sh
```

Ensure correct path and execution permissions.

### **Fix missing environment variables**

```bash
docker run -e DB_URL=mysql://... <image>
```

### **Fix wrong port mappings**

Match application port with Dockerfile EXPOSE and run commands.

### **Fix memory-related crashes**

Increase memory limits (for Docker Desktop) or optimize application.

### **Fix volume permission issues**

```bash
sudo chown -R 1000:1000 /data
```

### **Fix corrupted or outdated images**

```bash
docker pull <image>
```

### **Debug container shell**

```bash
docker run -it <image> sh
```

---

## **6. Best Practices / Tips**

* Use correct ENTRYPOINT + CMD pairing.
* Log everything to stdout/stderr.
* Ensure minimal, efficient Dockerfile.
* Use non-root users correctly with appropriate directory ownership.
* Validate file paths and WORKDIR.
* Test containers locally before pushing to ECR.
* Use multi-stage builds to avoid missing dependencies.
* Always include healthchecks in Kubernetes and Compose.
* Avoid copying unnecessary files (reduce image size and issues).
* Assign correct permissions for volume mounts.

---
---

# OOMKilled Debugging (Docker-Level) — Detailed Explanation Notes

## **1. Concept / What**

**OOMKilled (Out Of Memory Killed)** occurs when the Linux kernel terminates a container because it has exceeded the memory limit assigned via Docker cgroups or Kubernetes resource limits. This results in a container exiting with **code 137**, and in Kubernetes, it may lead to **CrashLoopBackOff**. The root cause is always related to the application inside the container using more memory than allowed.

---

## **2. Why / Purpose / Real-World Use Cases**

Understanding OOMKilled is essential because:

* Kubernetes only reports the *symptom*; the root cause is inside the container.
* CI/CD deployments fail when applications crash due to memory issues.
* EKS workloads rely on correctly configured resource limits.
* Memory misconfiguration is common in Java, Python, Node.js microservices.
* Java applications ignore container limits unless JVM heap (`-Xmx`, `-Xms`) is explicitly set.

Real-world scenarios:

* Spring Boot apps exceeding container memory.
* Node.js memory leaks in microservices.
* Python apps loading large files into memory.
* Containers failing due to low Docker Desktop memory allocation.
* Kubernetes pods restarting repeatedly due to memory thresholds.

---

## **3. How it Works / Steps / Syntax**

### **Check container status**

```bash
docker ps -a
```

Look for:

* `Exited (137)` → OOMKilled.

### **Check exit code**

```bash
docker inspect <container> --format='{{.State.ExitCode}}'
```

`137` confirms the container was killed due to memory exhaustion.

### **Check logs**

```bash
docker logs <container>
```

Common indications:

* Logs end abruptly.
* "Killed" messages.
* Memory-related failures.

### **Monitor live container memory usage**

```bash
docker stats
```

Shows:

* Memory usage
* Memory limits
* Realtime spikes

### **Check Docker memory limits**

```bash
docker run -m 512m <image>
```

If application memory exceeds this → OOMKilled.

### **Test without memory limits**

```bash
docker run -it <image>
```

If it works without limits but fails with limits → confirms memory issue.

### **Java-specific heap settings (critical)**

Java ignores container limits unless explicitly configured:

```bash
java -Xms256m -Xmx256m -jar app.jar
```

Without these flags → JVM may consume more memory than the container limit.

### **Check cgroup memory values inside container**

```bash
cat /sys/fs/cgroup/memory/memory.limit_in_bytes
cat /sys/fs/cgroup/memory/memory.usage_in_bytes
```

Useful for Kubernetes/EKS debugging.

---

## **4. Common Issues / Errors**

* Container exits with **137**.
* App killed without full error logs.
* Java apps consuming large memory due to no heap settings.
* Node.js apps exceeding default memory allocation.
* Python loading large data files fully into memory.
* Poor memory optimization in microservices.
* Too many threads or processes.
* Docker Desktop insufficient memory.
* Misconfigured Kubernetes resource limits.

---

## **5. Troubleshooting / Fixes**

### **Fix 1: Increase memory limits**

Docker:

```bash
docker run -m 1g <image>
```

Kubernetes:

```yaml
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "1Gi"
```

### **Fix 2: Set JVM heap limits (mandatory for Java)**

```bash
-Xms256m
-Xmx256m
```

Or via Dockerfile:

```Dockerfile
ENTRYPOINT ["java", "-Xms256m", "-Xmx256m", "-jar", "app.jar"]
```

### **Fix 3: Optimize application memory usage**

* Stream files instead of reading fully.
* Reduce unnecessary in-memory caching.
* Use smaller data structures.
* Increase GC efficiency.

### **Fix 4: Debug memory leaks**

Use tools like:

* Java Mission Control
* Node.js heap snapshots
* Python tracemalloc

### **Fix 5: Increase Docker Desktop memory**

```
Settings → Resources → Memory
```

### **Fix 6: Use `docker stats` for monitoring**

Identify containers that spike memory usage.

---

## **6. Best Practices / Tips**

* Always set JVM heap (`-Xms` and `-Xmx`) for Java microservices.
* Use proper memory requests and limits in Kubernetes.
* Test containers locally with realistic memory constraints.
* Use lightweight base images to reduce overhead.
* Avoid loading large files entirely into memory.
* Stream large data efficiently.
* Enable memory monitoring dashboards (Prometheus/Grafana).
* Run load testing to predict memory behavior.
* Prefer multi-stage builds to keep image minimal.
* Validate memory before pushing to ECR.

---
---

# Storage Driver Issues (overlay2) — Detailed Explanation Notes

## **1. Concept / What**

`overlay2` is Docker’s default storage driver used to manage container and image filesystems. It implements a **union filesystem**, which merges multiple read-only image layers with a writable container layer. Each layer contains many files, and **every file consumes one inode**. Storage driver issues occur when overlay2 cannot create, mount, or manage these layers due to disk space, inode exhaustion, corruption, or metadata inconsistencies.

---

## **2. Why / Purpose / Real-World Use Cases**

Understanding overlay2 is critical because:

* All Docker images and containers rely on overlay2 for file storage.
* Kubernetes (via containerd) uses overlayfs/overlay2 for pod filesystems.
* Disk or inode exhaustion directly causes container startup failures.
* CI/CD pipelines often break due to layer registration or overlay failures.
* Worker nodes in EKS, Jenkins agents, and EC2 hosts frequently face storage-driver issues.

Real-world situations:

* Pods failing to start due to full `/var/lib/containerd`.
* Docker builds failing due to "no space left on device".
* Nodes going `NotReady` due to overlay mount errors.
* Long-running CI agents accumulating millions of small temp files.

---

## **3. How it Works / Steps / Syntax**

### **Check disk usage**

```bash
df -h
```

Identify full partitions such as:

* `/var/lib/docker`
* `/var/lib/containerd`

### **Check inode usage (Critical)**

```bash
df -i
```

If inode usage reaches 100%, overlay2 cannot create new files, causing failures.

### **Check directory-level usage**

```bash
sudo du -sh /var/lib/docker/*
```

Highlights heavy subdirectories like:

* `overlay2/`
* `containers/`
* `volumes/`

### **Clean Docker and reclaim space**

```bash
docker system prune -a
```

Or individually:

```bash
docker image prune -a
docker container prune
docker volume prune
```

### **Check Docker daemon logs for overlay2 errors**

```bash
sudo journalctl -u docker -f
```

Common logs:

* `overlay2: failed to mount`
* `no space left on device`
* `failed to register layer`

### **Remove corrupted overlay layers**

```bash
sudo rm -rf /var/lib/docker/overlay2/<layer-id>
sudo systemctl restart docker
```

### **Check containerd snapshots (Kubernetes/EKS)**

```bash
sudo du -sh /var/lib/containerd/
```

Clean snapshots cautiously:

```bash
sudo rm -rf /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/*
sudo systemctl restart containerd
```

---

## **4. Common Issues / Errors**

* `no space left on device`
* Inode exhaustion despite free disk space
* `failed to register layer`
* `overlay2: failed to mount lowerdir`
* Corrupted layers due to incomplete writes
* Kubernetes pods failing at `ContainerCreating`
* Docker daemon crashes due to overlay metadata issues
* CI/CD build failures caused by buildup of temp/cache files

---

## **5. Troubleshooting / Fixes**

### **Fix 1: Free space & inodes**

```bash
docker system prune -a
```

Check inodes:

```bash
df -i
```

### **Fix 2: Remove corrupted overlay layers**

```bash
sudo rm -rf /var/lib/docker/overlay2/<layer-id>
sudo systemctl restart docker
```

### **Fix 3: Restart Docker daemon**

```bash
sudo systemctl restart docker
```

### **Fix 4: Expand disk (AWS real-world fix)**

Increase EBS volume size.

### **Fix 5: Move Docker data directory**

Edit `/etc/docker/daemon.json`:

```json
{
  "data-root": "/mnt/docker-data"
}
```

Restart Docker.

### **Fix 6: Fix inode exhaustion**

* Increase disk size.
* Use XFS (dynamic inode allocation).
* Clean temp, cache, and build artifacts.

### **Fix 7: Clean containerd snapshots (K8s)**

```bash
sudo rm -rf /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/*
sudo systemctl restart containerd
```

---

## **6. Best Practices / Tips**

* Allocate sufficient disk to `/var/lib/docker` or containerd stores.
* Use ephemeral CI/CD build agents to avoid layer buildup.
* Regularly prune old images and containers.
* Prefer lightweight base images (alpine/slim) to reduce inode usage.
* Avoid creating excessive temp/log files within containers.
* Use log rotation.
* For Kubernetes nodes, set automated cleanup scripts.
* Monitor disk usage and inode usage via Prometheus/Grafana.
* Reduce layers via efficient Dockerfile design (multi-stage builds).

---
---

# DOCKER DAEMON LOGS
## 1. Concept / What

**Docker Daemon Logs** are the logs generated by the Docker Engine (`dockerd`). These logs show Docker **engine-level issues**, not application/container logs.

* `docker logs <container>` → shows app-level logs **inside** the container.
* `journalctl -u docker` → shows **Docker daemon** errors such as:

  * overlay2 failures
  * mount issues
  * layer extraction failures
  * disk/inode problems
  * daemon crashes or restarts
  * permission/SELinux issues
  * network driver errors

---

# 2. Why / Purpose / Real-World Use Cases

Docker daemon logs are critical when debugging:

* Containers failing to start with **no useful app logs**
* Overlay2 storage driver issues
* Image pull or build failures
* Docker hanging or becoming unresponsive
* Disk or inode exhaustion problems
* CI/CD pipeline failures due to Docker issues
* Kubernetes nodes (older Docker-based clusters) runtime issues

DevOps engineers check daemon logs when container-level logs are **not enough**.

---

# 3. How It Works / Steps / Commands (Full Syntax)

## **3.1 View Docker daemon logs in realtime**

```
sudo journalctl -u docker -f
```

## **3.2 View logs from last 1 hour**

```
sudo journalctl -u docker --since "1 hour ago"
```

## **3.3 View full log history**

```
sudo journalctl -u docker
```

## **3.4 Check Docker service status**

```
sudo systemctl status docker
```

## **3.5 Inspect container failures (exit codes, OOMKilled)**

```
docker inspect <container>
```

## **3.6 Kubernetes nodes using containerd/kubelet**

```
sudo journalctl -u containerd -f
sudo journalctl -u kubelet -f
```

---

# 4. Common Issues / Errors (Real Docker Daemon Problems)

## **4.1 Disk Space Full**

Errors:

```
overlay2: write failed: no space left on device
```

## **4.2 Inode Exhaustion**

Errors:

```
failed to register layer
no space left on device
```

## **4.3 Overlay2 Mount Failures**

Errors:

```
overlay2: failed to mount lowerdir
```

## **4.4 Permission / SELinux Problems**

Errors:

```
OCI runtime error: permission denied
operation not permitted
```

## **4.5 Read-Only Filesystem Issues**

Errors:

```
overlay2: upperdir is read-only
```

---

# 5. Troubleshooting / Fixes (Real DevOps Workflow)

## **5.1 Check Docker daemon logs**

```
sudo journalctl -u docker -f
```

## **5.2 Check disk usage**

```
df -h
```

## **5.3 Check inode usage**

```
df -i
```

## **5.4 Analyze overlay2 directory size**

```
sudo du -sh /var/lib/docker/*
```

## **5.5 Restart Docker service**

```
sudo systemctl restart docker
```

## **5.6 Remove corrupted layers/images**

```
docker image prune
```

## **5.7 For read-only filesystem**

* Reboot node
* Run `dmesg` to check disk errors
* Possibly replace VM/disk

---

# 6. Discussion Summary (From Our Conversation)

* You understood the core difference:

  * `docker logs` = application logs
  * `journalctl -u docker` = Docker engine logs
* When container logs don’t show the error, daemon logs always reveal the true cause.
* This is why DevOps engineers must know daemon-level debugging.
* Training institutes rarely teach this because they use Docker Desktop, not Linux servers.

Your conclusion was correct:

> If container-level logs don’t explain the problem, check daemon-level logs.

---

# 7. Best Practices / Tips

* Always check **both** disk (`df -h`) and inode usage (`df -i`).
* Regularly clean unused Docker data:

```
docker system prune -a
```

* Avoid small root disks for `/var/lib/docker`.
* Use lightweight images to reduce overlay2 pressure.
* Watch for SELinux/AppArmor issues.
* Monitor Docker daemon health on production servers.

---
---

# 1. Concept / What: Docker Disk Cleanup (Prune, Dangling Images, Safe Cleanup)

Docker Disk Cleanup refers to the process of removing **unused Docker data** such as:

* Stopped containers
* Dangling images (`<none>:<none>`)
* Unused image layers
* Build cache
* Unused networks
* Unused volumes

Docker stores everything under:

```
/var/lib/docker
```

This directory grows rapidly due to builds, pulls, CI/CD pipelines, and overlay2 layers.

Cleaning unused data prevents:

* Disk full errors
* Inode exhaustion
* overlay2 mount failures
* Docker daemon crashes
* CI/CD failures
* Kubernetes node DiskPressure

---

# 2. Why / Purpose / Real-World Use Cases

DevOps engineers perform Docker cleanup when:

* Jenkins agents fill with cached layers
* Kubernetes nodes accumulate unused images after deployments
* Docker builds fail with:

  ```
  no space left on device
  failed to write
  ```
* overlay2 errors appear due to storage bloat
* Repeated builds generate dangling layers
* Old application versions remain unused on the node

Proper cleanup ensures:

* Stable CI/CD pipelines
* Healthy worker nodes
* Faster image pulls and builds
* Lower storage usage

---

# 3. How It Works / Commands / Syntax

## 3.1 Remove Stopped Containers (Safe)

```
docker container prune
```

Removes only **stopped** containers.

---

## 3.2 Remove Dangling Images (Safe)

# 1. Concept / What: Permission Issues in Docker Mounts

Permission issues in Docker occur when a container **cannot read or write** to a directory that is mounted from the host. This happens in both:

* **Bind mounts** (`-v /host:/container`)
* **Docker volumes** (`docker volume create`, then mount)

A mount permission issue means:

* The host directory has restrictive permissions
* The container runs as a different (non-root) user
* SELinux/AppArmor policies block access
* The mount is read-only
* Filesystem ownership does not match container user

Host filesystem permissions ALWAYS override container permissions.

---

# 2. Why / Purpose / Real-World Use Cases

Mounts are essential for DevOps workflows:

* Applications writing logs to host directories
* Sharing source code for local development
* Mounting config files or SSL certificates
* Persistent data for databases
* Jenkins workspaces mounted into containers
* hostPath volumes in Kubernetes

Permission failures cause:

* Containers crashing at startup
* CI/CD pipelines failing to write artifacts
* Apps unable to generate logs
* Read-only filesystem errors
* Kubernetes pods going into CrashLoopBackOff

Thus, mount permission issues are one of the **most common real-world Docker/K8s problems**.

---

# 3. How It Works / Steps / Syntax

## 3.1 Bind Mount Example

```
docker run -v /host/app/logs:/var/app/logs myimage
```

If `/host/app/logs` is owned by root and container runs as a non-root user → write fails.

---

## 3.2 Volume Mount Example

```
docker volume create myvol
```

```
docker run -v myvol:/data myimage
```

Still requires correct permissions inside `/data`.

---

## 3.3 Check Host Folder Permissions

```
ls -ld /path/on/host
```

---

## 3.4 Check Container User

```
docker exec -it <container> bash
whoami
id
```

Containers often run as non-root users like 1000 or 1001.

---

## 3.5 Fix Permissions on Host (when safe)

```
sudo chown -R 1001:1001 /host/dir
sudo chmod -R 755 /host/dir
```

---

## 3.6 Read-Only Mount Detection

```
docker inspect <container> | grep -i readonly
```

In Compose or K8s:

```
/host:/container:ro
```

---

## 3.7 SELinux Fix (RHEL/CentOS)

```
-v /host:/container:Z
```

Ensures correct SELinux context.

---

# 4. Common Issues / Errors

## 4.1 Permission Denied

```
open /app/logs/app.log: permission denied
```

Cause: mismatched UID/GID.

---

## 4.2 Cannot Create Directory

```
mkdir: cannot create directory '/data': Permission denied
```

Cause: host folder owned by root.

---

## 4.3 Read-Only Filesystem

```
read-only filesystem
```

Cause: RO mount or corrupted filesystem.

---

## 4.4 SELinux Blocking Writes

```
permission denied (selinux)
```

Cause: missing SELinux context.

---

## 4.5 Wrong Container User

```
User 1001 cannot write to /var/log/app
```

Cause: container uses non-root user.

---

# 5. Troubleshooting / Fixes

## 5.1 Check Host Directory Ownership

```
ls -ld /path/on/host
```

## 5.2 Check Container User Identity

```
whoami
id
```

## 5.3 Fix Ownership on Host

```
sudo chown -R 1001:1001 /path/on/host
```

## 5.4 Fix Permissions

```
sudo chmod -R 755 /path/on/host
```

## 5.5 Ensure Mount Is Not Read-Only

```
/host:/container:rw
```

## 5.6 SELinux Context Update

```
-v /host:/container:Z
```

## 5.7 Debug by Running Container as Root Temporarily

```
docker run -u 0 ...
```

(Not recommended for production.)

---

# 6. Discussion Summary (From Our Conversation)

* We discussed how **host permissions override container permissions**.
* Non-root containers often face write issues due to UID mismatches.
* Docker will not automatically adjust host directory ownership.
* Kubernetes hostPath and Jenkins pipelines frequently hit permission issues.
* SELinux on RHEL/CentOS often causes silent permission denials.
* Using volumes is safer and avoids many bind mount issues.

---

# 7. Best Practices / Tips

* Prefer **Docker volumes** over bind mounts.
* Avoid running containers as root except for debugging.
* Match host directory permissions with the container's user ID.
* For Kubernetes, use **PVCs** instead of hostPath.
* Pre-create host directories with correct ownership.
* Use SELinux flags `:z` or `:Z` when required.
* Never mount entire system directories inside containers.

---

---
# 1. Concept / What: Permission Issues in Docker Mounts

Permission issues in Docker occur when a container **cannot read or write** to a directory that is mounted from the host. This happens in both:

* **Bind mounts** (`-v /host:/container`)
* **Docker volumes** (`docker volume create`, then mount)

A mount permission issue means:

* The host directory has restrictive permissions
* The container runs as a different (non-root) user
* SELinux/AppArmor policies block access
* The mount is read-only
* Filesystem ownership does not match container user

Host filesystem permissions ALWAYS override container permissions.

---

# 2. Why / Purpose / Real-World Use Cases

Mounts are essential for DevOps workflows:

* Applications writing logs to host directories
* Sharing source code for local development
* Mounting config files or SSL certificates
* Persistent data for databases
* Jenkins workspaces mounted into containers
* hostPath volumes in Kubernetes

Permission failures cause:

* Containers crashing at startup
* CI/CD pipelines failing to write artifacts
* Apps unable to generate logs
* Read-only filesystem errors
* Kubernetes pods going into CrashLoopBackOff

Thus, mount permission issues are one of the **most common real-world Docker/K8s problems**.

---

# 3. How It Works / Steps / Syntax

## 3.1 Bind Mount Example

```
docker run -v /host/app/logs:/var/app/logs myimage
```

If `/host/app/logs` is owned by root and container runs as a non-root user → write fails.

---

## 3.2 Volume Mount Example

```
docker volume create myvol
```

```
docker run -v myvol:/data myimage
```

Still requires correct permissions inside `/data`.

---

## 3.3 Check Host Folder Permissions

```
ls -ld /path/on/host
```

---

## 3.4 Check Container User

```
docker exec -it <container> bash
whoami
id
```

Containers often run as non-root users like 1000 or 1001.

---

## 3.5 Fix Permissions on Host (when safe)

```
sudo chown -R 1001:1001 /host/dir
sudo chmod -R 755 /host/dir
```

---

## 3.6 Read-Only Mount Detection

```
docker inspect <container> | grep -i readonly
```

In Compose or K8s:

```
/host:/container:ro
```

---

## 3.7 SELinux Fix (RHEL/CentOS)

```
-v /host:/container:Z
```

Ensures correct SELinux context.

---

# 4. Common Issues / Errors

## 4.1 Permission Denied

```
open /app/logs/app.log: permission denied
```

Cause: mismatched UID/GID.

---

## 4.2 Cannot Create Directory

```
mkdir: cannot create directory '/data': Permission denied
```

Cause: host folder owned by root.

---

## 4.3 Read-Only Filesystem

```
read-only filesystem
```

Cause: RO mount or corrupted filesystem.

---

## 4.4 SELinux Blocking Writes

```
permission denied (selinux)
```

Cause: missing SELinux context.

---

## 4.5 Wrong Container User

```
User 1001 cannot write to /var/log/app
```

Cause: container uses non-root user.

---

# 5. Troubleshooting / Fixes

## 5.1 Check Host Directory Ownership

```
ls -ld /path/on/host
```

## 5.2 Check Container User Identity

```
whoami
id
```

## 5.3 Fix Ownership on Host

```
sudo chown -R 1001:1001 /path/on/host
```

## 5.4 Fix Permissions

```
sudo chmod -R 755 /path/on/host
```

## 5.5 Ensure Mount Is Not Read-Only

```
/host:/container:rw
```

## 5.6 SELinux Context Update

```
-v /host:/container:Z
```

## 5.7 Debug by Running Container as Root Temporarily

```
docker run -u 0 ...
```

(Not recommended for production.)

---

# 6. Discussion Summary (From Our Conversation)

* We discussed how **host permissions override container permissions**.
* Non-root containers often face write issues due to UID mismatches.
* Docker will not automatically adjust host directory ownership.
* Kubernetes hostPath and Jenkins pipelines frequently hit permission issues.
* SELinux on RHEL/CentOS often causes silent permission denials.
* Using volumes is safer and avoids many bind mount issues.

---

# 7. Best Practices / Tips

* Prefer **Docker volumes** over bind mounts.
* Avoid running containers as root except for debugging.
* Match host directory permissions with the container's user ID.
* For Kubernetes, use **PVCs** instead of hostPath.
* Pre-create host directories with correct ownership.
* Use SELinux flags `:z` or `:Z` when required.
* Never mount entire system directories inside containers.

---

---
Dangling images appear as `<none>:<none>`.

List them:

```
docker images -f "dangling=true"
```

Remove:

```
docker image prune
```

---

## 3.3 Remove All Unused Images (More Aggressive)

```
docker image prune -a
```

Removes images **not used by running containers**.

---

## 3.4 Remove Unused Networks

```
docker network prune
```

---

## 3.5 Remove Unused Volumes

```
docker volume prune
```

(Use carefully if volumes store app data.)

---

## 3.6 Full Cleanup (Not Safe on Production)

```
docker system prune -a
```

Removes:

* Stopped containers
* Unused images
* Dangling images
* Unused networks
* Build cache
* Unused volumes

**This is NOT recommended on production systems.**

---

# 4. Common Issues / Errors

## 4.1 Disk Full

```
overlay2: write failed: no space left on device
```

Cause: too many images, layers, logs.

---

## 4.2 Inode Exhaustion

```
no space left on device
failed to register layer
```

Cause: millions of small files.

---

## 4.3 "Device or Resource Busy"

```
unable to remove layer: resource busy
```

Cause: container still using the layer.

---

## 4.4 Read-Only Filesystem

```
overlay2: upperdir is read-only
```

Cause: disk corruption, kernel remount.

---

## 4.5 Permission Denied

```
operation not permitted
```

Cause: missing sudo privileges.

---

# 5. Troubleshooting / Fixes

## 5.1 Check Disk Usage

```
df -h
```

## 5.2 Check Inode Usage

```
df -i
```

## 5.3 Find What Is Using Space

```
sudo du -sh /var/lib/docker/*
```

## 5.4 Safe Cleanup Steps

Start safe:

```
docker image prune
```

Then:

```
docker container prune
```

If still full:

```
docker image prune -a
```

## 5.5 Restart Docker Daemon

```
sudo systemctl restart docker
```

## 5.6 For Read-Only FS

* Reboot
* Check `dmesg` for errors
* Replace instance/disk

---

# 6. Discussion Summary (From Our Conversation)

* You understood that `docker rm` can delete specific containers, while `container prune` is safer.
* We discussed that **dangling images** are untagged leftover layers (`<none>:<none>`) that waste space.
* We clarified the dangerous part:

  * `docker system prune -a` does NOT delete running containers.
  * It deletes **old untagged images** even if running containers originally used them.
  * Running containers keep working because they have their own copy of the filesystem.
  * But restart fails because the original image ID is gone.
* NEW images are never deleted because they still have tags.
* OLD images lose their tags when you rebuild with the same tag.

This causes prune to delete the OLD image → restart failure.

---

# 7. Best Practices / Tips

* **Do NOT run `docker system prune -a` on production nodes.**
* Prefer safe commands:

  ```
  docker image prune
  docker container prune
  ```
* Avoid building large images directly on production servers.
* Enable Docker log rotation in `/etc/docker/daemon.json`:

```
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

* Periodically clean Jenkins agents.
* For Kubernetes nodes, drain the node before aggressive cleanup.

```
kubectl drain <node>
```

```
kubectl uncordon <node>
```

* Always check disk + inode usage together.

---
---

# 1. Concept / What: Permission Issues in Docker Mounts

Permission issues in Docker occur when a container **cannot read or write** to a directory that is mounted from the host. This happens in both:

* **Bind mounts** (`-v /host:/container`)
* **Docker volumes** (`docker volume create`, then mount)

A mount permission issue means:

* The host directory has restrictive permissions
* The container runs as a different (non-root) user
* SELinux/AppArmor policies block access
* The mount is read-only
* Filesystem ownership does not match container user

Host filesystem permissions ALWAYS override container permissions.

---

# 2. Why / Purpose / Real-World Use Cases

Mounts are essential for DevOps workflows:

* Applications writing logs to host directories
* Sharing source code for local development
* Mounting config files or SSL certificates
* Persistent data for databases
* Jenkins workspaces mounted into containers
* hostPath volumes in Kubernetes

Permission failures cause:

* Containers crashing at startup
* CI/CD pipelines failing to write artifacts
* Apps unable to generate logs
* Read-only filesystem errors
* Kubernetes pods going into CrashLoopBackOff

Thus, mount permission issues are one of the **most common real-world Docker/K8s problems**.

---

# 3. How It Works / Steps / Syntax

## 3.1 Bind Mount Example

```
docker run -v /host/app/logs:/var/app/logs myimage
```

If `/host/app/logs` is owned by root and container runs as a non-root user → write fails.

---

## 3.2 Volume Mount Example

```
docker volume create myvol
```

```
docker run -v myvol:/data myimage
```

Still requires correct permissions inside `/data`.

---

## 3.3 Check Host Folder Permissions

```
ls -ld /path/on/host
```

---

## 3.4 Check Container User

```
docker exec -it <container> bash
whoami
id
```

Containers often run as non-root users like 1000 or 1001.

---

## 3.5 Fix Permissions on Host (when safe)

```
sudo chown -R 1001:1001 /host/dir
sudo chmod -R 755 /host/dir
```

---

## 3.6 Read-Only Mount Detection

```
docker inspect <container> | grep -i readonly
```

In Compose or K8s:

```
/host:/container:ro
```

---

## 3.7 SELinux Fix (RHEL/CentOS)

```
-v /host:/container:Z
```

Ensures correct SELinux context.

---

# 4. Common Issues / Errors

## 4.1 Permission Denied

```
open /app/logs/app.log: permission denied
```

Cause: mismatched UID/GID.

---

## 4.2 Cannot Create Directory

```
mkdir: cannot create directory '/data': Permission denied
```

Cause: host folder owned by root.

---

## 4.3 Read-Only Filesystem

```
read-only filesystem
```

Cause: RO mount or corrupted filesystem.

---

## 4.4 SELinux Blocking Writes

```
permission denied (selinux)
```

Cause: missing SELinux context.

---

## 4.5 Wrong Container User

```
User 1001 cannot write to /var/log/app
```

Cause: container uses non-root user.

---

# 5. Troubleshooting / Fixes

## 5.1 Check Host Directory Ownership

```
ls -ld /path/on/host
```

## 5.2 Check Container User Identity

```
whoami
id
```

## 5.3 Fix Ownership on Host

```
sudo chown -R 1001:1001 /path/on/host
```

## 5.4 Fix Permissions

```
sudo chmod -R 755 /path/on/host
```

## 5.5 Ensure Mount Is Not Read-Only

```
/host:/container:rw
```

## 5.6 SELinux Context Update

```
-v /host:/container:Z
```

## 5.7 Debug by Running Container as Root Temporarily

```
docker run -u 0 ...
```

(Not recommended for production.)

---

# 6. Discussion Summary (From Our Conversation)

* We discussed how **host permissions override container permissions**.
* Non-root containers often face write issues due to UID mismatches.
* Docker will not automatically adjust host directory ownership.
* Kubernetes hostPath and Jenkins pipelines frequently hit permission issues.
* SELinux on RHEL/CentOS often causes silent permission denials.
* Using volumes is safer and avoids many bind mount issues.

---

# 7. Best Practices / Tips

* Prefer **Docker volumes** over bind mounts.
* Avoid running containers as root except for debugging.
* Match host directory permissions with the container's user ID.
* For Kubernetes, use **PVCs** instead of hostPath.
* Pre-create host directories with correct ownership.
* Use SELinux flags `:z` or `:Z` when required.
* Never mount entire system directories inside containers.

---
---
---

