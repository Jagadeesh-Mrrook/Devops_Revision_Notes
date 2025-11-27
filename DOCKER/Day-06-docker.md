# Docker Named Volumes — Detailed Explanation Notes

## **Concept / What**

A **named volume** is a Docker-managed persistent storage location that lives on the host machine but is managed internally by Docker. You don't choose the host path; Docker creates and manages it under `/var/lib/docker/volumes`. Named volumes allow containers to store and retrieve persistent data even if containers or images are removed.

---

## **Why / Purpose / Real-World Use Cases**

### **Purpose**

* Persist data across container restarts, rebuilds, crashes.
* Decouple data from the container's lifecycle.
* Avoid dependency on manually defined host directories.
* Ensure consistent behavior across dev, CI/CD, and production.

### **Real Use Cases**

* **Databases**: MySQL, PostgreSQL, MongoDB persistent storage.
* **Stateful microservices**: caching, uploads, logs.
* **Jenkins**: persisting `/var/jenkins_home` when run in Docker.
* **CI/CD**: caching build dependencies to reduce pipeline time.
* **Local dev → Prod migration** without path conflicts.

---

## **How it Works / Steps / Syntax**

### **Create a Named Volume**

```bash
docker volume create mydata
```

This creates a folder on the host: `/var/lib/docker/volumes/mydata/_data`.

### **Run a Container Using a Named Volume**

```bash
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=admin123 \
  -v mydata:/var/lib/mysql \
  mysql:8
```

Docker mounts the volume inside the container at `/var/lib/mysql`.

### **List Existing Volumes**

```bash
docker volume ls
```

### **Inspect a Volume**

```bash
docker volume inspect mydata
```

Gives host path and config.

### **Remove a Volume**

```bash
docker volume rm mydata
```

### **Compose Example (Recommended)**

```yaml
version: "3.9"

services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: admin123
    volumes:
      - dbdata:/var/lib/mysql

volumes:
  dbdata:
```

Docker automatically creates `dbdata` if it doesn't already exist.

---

## **Common Issues / Errors**

### **1. Data Lost After Container Restart**

**Cause:** Wrong mount path or volume not attached.

**Example mistake:**

```yaml
- dbdata:/wrong/path
```

### **2. Typo Creates a New Volume Automatically**

Docker silently creates a new volume if the name is misspelled.

### **3. Storage Gets Full**

Large DBs, logs, or artifacts can fill disk.

### **4. Permission Issues with Non-Root Containers**

Containers running as UID 1000 may not access the volume directory.

### **5. Unused Volumes Accumulating**

Old builds/containers leave unused volumes behind.

---

## **Troubleshooting / Fixes**

### **Fix: Ensure Correct Mount Path**

Check the application's real data path.

### **Fix: Clean Unused Volumes**

```bash
docker volume prune
```

### **Fix: Identify Which Container Uses a Volume**

```bash
docker ps -a --filter volume=mydata
```

### **Fix: Permission Issues (Non-root)**

Run with matching user group:

```bash
docker run --user 1000:1000 -v mydata:/app/data image
```

Or fix permissions inside entrypoint.

### **Fix: Debug Volume Contents**

Temporary container:

```bash
docker run -it --rm -v mydata:/data alpine sh
```

---

## **Best Practices / Tips**

* Use **named volumes** for all stateful container data.
* Avoid bind mounts in production due to path and permission risks.
* Declare volumes explicitly in Compose to avoid accidental volume creation.
* Use clear names like `mysql_data`, `jenkins_home`, `redis_cache`.
* Backup volumes periodically (especially DB volumes).
* Keep containers stateless; move all data directories to volumes.
* Use multi-stage builds and minimal images to reduce storage pressure.
* For production Kubernetes/EKS: move to **EBS/EFS CSI volumes** instead of Docker volumes.

---
---

# Docker Bind Mounts — Detailed Explanation Notes

## **Concept / What**

A **bind mount** directly maps a directory or file from the **host machine** into a **container path**. Unlike named volumes, bind mounts always require you to specify:

```
-v <host_path>:<container_path>
```

Bind mounts allow real-time synchronization: changes on the host immediately reflect inside the container, and changes inside the container reflect back to the host.

---

## **Why / Purpose / Real-World Use Cases**

### **Purpose**

* Enable host ↔ container file sharing.
* Provide live code editing for development.
* Share configs, logs, and scripts with containers.
* Use host directories inside containers during CI/CD jobs.

### **Real Use Cases**

* **Local development:** Editing source code on host while app runs inside container.
* **Jenkins pipelines:** Mapping Jenkins workspace into build containers.
* **Debugging:** Access container logs and artifacts directly from host.
* **Config Injection:** Mapping certificates, config files, kubeconfig, secrets (read-only).
* **CLI Tools in Containers:** Terraform, AWS CLI, Ansible inside containers while reading code from host.

Bind mounts are almost never used in production due to portability, permission, and security concerns.

---

## **How it Works / Steps / Syntax**

### **Basic Bind Mount**

```bash
docker run -d \
  -v /home/user/app:/app \
  node:18
```

Host folder `/home/user/app` is mounted inside container at `/app`.

### **Mount Multiple Host Directories**

```bash
docker run -d \
  -v /home/user/code:/app/code \
  -v /home/user/logs:/app/logs \
  -v /home/user/config:/app/config \
  app-image
```

### **Mount a Single File**

```bash
docker run -v /home/user/config.yaml:/etc/app/config.yaml app
```

### **Read-Only Mount**

```bash
docker run -v /home/user/certs:/app/certs:ro app
```

### **Bind Mount via Docker Compose**

```yaml
version: "3.9"
services:
  web:
    image: nginx
    volumes:
      - ./html:/usr/share/nginx/html
```

---

## **Common Issues / Errors**

### **1. Host Path Doesn't Exist**

**Error:** "No such file or directory"

**Cause:** Host directory missing.

**Fix:**

```bash
mkdir -p /home/user/app
```

### **2. Permission Denied**

**Cause:** Container user cannot read/write the host folder.

**Fix:**

* Change host permissions:

  ```bash
  sudo chmod -R 755 /home/user/app
  ```
* Or run container as matching user:

  ```bash
  docker run --user $(id -u):$(id -g) -v /home/user/app:/app image
  ```

### **3. SELinux Block (RHEL/CentOS)**

**Fix:**

```bash
-v /home/user/app:/app:Z
```

### **4. Typo in Host Path**

The container will see an empty directory because Docker auto-creates missing paths.

### **5. Windows Path Format Errors**

Incorrect path formatting causes mount failures.

### **6. Log Conflicts**

Mapping container log folders to host may cause log overwrites.

---

## **Troubleshooting / Fixes**

### **Fix: Inspect Mounts**

```bash
docker inspect container_name | grep Mounts -A 10
```

### **Fix: Use Temporary Container to Inspect Host Data**

```bash
docker run -it --rm -v /home/user/app:/data alpine sh
```

### **Fix: Run Container as Your User**

Prevents permission issues.

### **Fix: Avoid Sensitive Host Paths**

Do not mount:

* `/`
* `/root`
* `/etc`
* `/var/lib`
* Docker internal paths

---

## **Best Practices / Tips**

* Use bind mounts **only** for development and CI/CD.
* Avoid bind mounts in production; prefer named volumes or cloud storage.
* Always ensure host path exists before mounting.
* Use read-only mounts for configs and certs.
* Avoid mixing logs, data, configs into a single bind mount.
* Keep mounts minimal to reduce security exposure.
* For Kubernetes/EKS: **never** use bind mounts — use PVCs with EBS/EFS.
* Maintain clear separation of concerns: code, logs, configs should each have separate directories.

---
---

# Docker Anonymous Volumes — Detailed Explanation Notes

## **Concept / What**

An **anonymous volume** is a Docker-created volume with **no user-defined name**. Docker automatically generates a random volume ID when:

* A container uses a `VOLUME` instruction from the image’s Dockerfile, or
* You run a container with `-v <container_path>` (without specifying a host path or named volume).

Anonymous volumes store runtime data inside:

```
/var/lib/docker/volumes/<random_id>/_data
```

They are temporary in nature but remain on disk until explicitly removed.

---

## **Why / Purpose / Real-World Use Cases**

### **Purpose**

* Protect important directories inside the image from being overwritten.
* Provide ephemeral storage for container runtime data.
* Automatically create storage required by applications without user intervention.

### **Real Use Cases**

* Images that define:

  ```dockerfile
  VOLUME /data
  ```
* Containers requiring temporary runtime caches.
* Avoiding writes directly inside the container filesystem.
* Situations where persistence is not required long-term.

Anonymous volumes are **not** used for production persistence.

---

## **How it Works / Steps / Syntax**

### **Created Automatically**

When running a container whose image defines `VOLUME`:

```bash
docker run myimage
```

Docker creates:

```
/var/lib/docker/volumes/<random_hash>/_data
```

### **Anonymous Volume via CLI**

```bash
docker run -v /app/data busybox
```

Because no name or host path is provided, Docker generates an anonymous volume.

### **List Anonymous Volumes**

```bash
docker volume ls
```

You will see long random IDs instead of readable names.

### **Inspect**

```bash
docker volume inspect <volume_id>
```

Shows mount path and metadata.

### **Remove a Single Anonymous Volume**

```bash
docker volume rm <volume_id>
```

### **Remove All Unused Anonymous Volumes**

```bash
docker volume prune
```

### **Auto-Remove Anonymous Volume with Container**

```bash
docker run --rm image
```

Deletes container *and* its anonymous volumes after exit.

### **Prevent Anonymous Volumes**

Use named volumes:

```bash
docker run -v mydata:/app/data app
```

Or override Dockerfile VOLUME:

```bash
docker run -v mydata:/data myimage
```

---

## **Common Issues / Errors**

### **1. Disk Space Fills Up**

Anonymous volumes accumulate over time.

### **2. Hard to Identify Source**

Random volume IDs make it difficult to know which container created them.

### **3. Orphaned Volumes**

Deleting containers without `--rm` leaves anonymous volumes behind.

### **4. Image Upgrade Leads to New Anonymous Volumes**

A Dockerfile with `VOLUME` creates new volumes each time the container runs.

### **5. Unexpected Data Persistence**

Some users expect anonymous volumes to be automatically removed, but Docker keeps them unless deleted.

---

## **Troubleshooting / Fixes**

### **Fix: Clean All Unused Anonymous Volumes**

```bash
docker volume prune
```

### **Fix: Remove ALL Unused Docker Data**

```bash
docker system prune -a
```

### **Fix: Inspect Containers for Mounts**

```bash
docker inspect <container> | grep Mounts -A 10
```

### **Fix: Auto-Cleanup for Temporary Containers**

Use:

```bash
docker run --rm
```

### **Fix: Override Anonymous Volume Creation**

Bind a named volume or bind mount instead.

---

## **Best Practices / Tips**

* Avoid anonymous volumes for any persistent data.
* Use named volumes for clear identification and long-term storage.
* Regularly prune anonymous volumes during development.
* Avoid using `VOLUME` in Dockerfiles unless required.
* For CI/CD or testing, run containers with `--rm`.
* For EKS or Kubernetes in general, avoid Dockerfile VOLUME; use Persistent Volume Claims instead.
* Keep development and production storage behavior predictable by declaring all volumes explicitly.

---
---

# Docker Data Persistence Patterns — Detailed Explanation Notes

## **Concept / What**

**Data persistence patterns** define how data created by Docker containers is stored so it survives container restarts, deletion, or image rebuilds. Since container filesystems are ephemeral, persistence requires storing data **outside** the container lifecycle. These patterns include named volumes, bind mounts, anonymous volumes, and external storage (EBS/EFS) mounted on the host.

---

## **Why / Purpose / Real-World Use Cases**

### **Purpose**

* Ensure container data does not disappear after container removal.
* Support stateful applications such as databases.
* Separate container lifecycle from data lifecycle.
* Enable multi-container sharing of data.
* Improve CI/CD performance by caching build artifacts.
* Support production-grade storage in AWS/EKS via EBS/EFS.

### **Real Use Cases**

* **Databases:** MySQL, PostgreSQL, MongoDB, Redis.
* **Microservices:** User uploads, logs, cache directories.
* **CI/CD pipelines:** Maven/npm/pip caching, Jenkins workspaces.
* **Shared storage:** Multiple containers reading/writing the same directory.
* **Production workloads:** External storage mounted from AWS EBS/EFS.

---

## **How it Works / Steps / Syntax**

Docker supports four main patterns for persistent storage.

---

## **1. Named Volumes (Primary Recommended Method)**

**Docker-managed persistent storage** stored under:

```
/var/lib/docker/volumes/<volume_name>/_data
```

### Use

```bash
docker volume create app_data

docker run -d -v app_data:/app/data myimage
```

### Use Cases

* Databases
* Application state
* Uploads/logs
* CI/CD cache directories

---

## **2. Bind Mounts (Host Path → Container Path)**

Map host directories into containers:

```bash
docker run -v /home/user/app:/app myimage
```

### Use Cases

* Local development
* Jenkins pipelines
* Sharing configs/logs
* Debugging

Not recommended for production due to path/permission/security issues.

---

## **3. Anonymous Volumes (Auto-Created & Temporary)**

Automatically created when Dockerfile contains `VOLUME` or when you run:

```bash
docker run -v /container/path myimage
```

Stored under:

```
/var/lib/docker/volumes/<random_id>/_data
```

### Use Cases

* Temporary caches
* Automatically preserved directories
* Runtime-only data

Needs manual cleanup:

```bash
docker volume prune
```

---

## **4. External Storage (EBS/EFS Mounted on Host → Container)**

Docker itself cannot directly talk to EBS/EFS. Instead:

1. **Mount EBS/EFS on the Docker host**, e.g., `/mnt/ebs_data` or `/mnt/efs_data`.
2. **Bind-mount or create a Docker volume** pointing to that host path.

### Example — Bind Mount

```bash
docker run -d -v /mnt/ebs_data/db:/var/lib/postgresql myimage
```

### Example — Named Volume

```bash
docker volume create \
  --driver local \
  --opt type=none \
  --opt o=bind \
  --opt device=/mnt/efs_data/app \
  efs_app_data

docker run -d -v efs_app_data:/app/data myimage
```

### Use Cases

* Production workloads needing AWS storage
* Multi-host shared storage (EFS)
* Block-store databases (EBS)
* Stateful applications outside Kubernetes

---

## **Common Issues / Errors**

### **1. Data Lost After Container Restart**

Cause: Volume not mounted.

**Fix:** Explicitly mount required directories.

### **2. Wrong Path Mounted**

Container writes data to a location different from your mount path.

**Fix:** Verify actual application data directory.

### **3. Permission Errors**

Containers running as non-root may not access mounts.

**Fix:**

```bash
docker run --user 1000:1000 ...
```

Or adjust host permissions.

### **4. Anonymous Volumes Accumulating**

Cause: Dockerfile's VOLUME instruction.

**Fix:**

```bash
docker volume prune
```

### **5. Unexpected Data Overwrite**

Binding an empty host folder over a container path wipes the container's original contents.

**Fix:** Prefer named volumes unless host path is intentionally used.

### **6. Mount Not Available on Boot (EBS/EFS)**

If host reboot happens, containers may start with empty directories.

**Fix:** Proper `/etc/fstab` entries; use `nofail`.

### **7. EFS Latency Issues**

High I/O workloads perform poorly on NFS.

**Fix:** Choose correct EFS performance mode or use EBS instead.

---

## **Troubleshooting / Fixes**

### **Check Container Mounts**

```bash
docker inspect <container> | grep Mounts -A 10
```

### **Debug Volume Content**

```bash
docker run -it --rm -v app_data:/data alpine sh
```

### **Ensure EBS Partition Mounted Properly**

```bash
lsblk
mount | grep ebs
```

### **Ensure EFS NFS Reachable**

Check SG rules (port 2049).

### **Clean Unused Volumes**

```bash
docker volume prune
```

---

## **Best Practices / Tips**

### **For Local and CI/CD**

* Use bind mounts for local development.
* Use named volumes for caching dependencies.
* Avoid accumulating anonymous volumes.

### **For Production Docker**

* Use named volumes for persistent storage.
* Use one volume per folder (`logs`, `uploads`, `db`, `cache`).
* Avoid Dockerfile `VOLUME` unless required.
* Never rely on anonymous volumes for persistence.
* Ensure correct file ownership for application users.

### **For AWS Environments**

* Bind EBS/EFS mounts into containers after mounting them on the host.
* Use EFS for multi-host shared storage.
* Use EBS for single-host, high-performance block workloads.
* Regularly back up EBS (snapshots) and use AWS Backup for EFS.
* For EKS, always use CSI drivers instead of Docker mounts.

---
---

# Docker Volume Backup & Restore — Detailed Explanation Notes

## **Concept / What**

Docker **Volume Backup & Restore** refers to the method of exporting and re-importing data stored inside Docker volumes. Since Docker container filesystems are ephemeral, persistent data stored under:

```
/var/lib/docker/volumes/<volume_name>/_data
```

must be backed up to avoid loss if the host or container fails. Backup creates a `.tar`/`.tar.gz` file, and restore extracts that file back into a volume.

---

## **Why / Purpose / Real-World Use Cases**

### **Purpose**

* Disaster recovery if container/host crashes.
* Migration of applications to new servers or AWS regions.
* Safe upgrades of containers/images.
* CI/CD artifact transfer.
* Environment cloning (dev → stage → prod).
* Protecting persistent data beyond a single host's lifecycle.

### **Real Use Cases**

* Backing up MySQL/PostgreSQL data before upgrading image.
* Copying Jenkins home volume to a new EC2.
* Exporting user-uploaded files from a volume.
* Saving cache directories between CI pipeline runs.
* Moving persistent microservice data to another host.

---

## **How it Works / Steps / Syntax**

Docker does **not** upload backups to S3/EFS. It **only creates backup files on the host**. You later upload them externally with another tool.

---

## **1. Backup Using Temporary Container (Most Common Method)**

This reads the volume data and writes it to a compressed file on the host.

### **Backup**

```bash
docker run --rm \
  -v mydata:/source \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/backup.tar.gz -C /source .
```

### What each `-v` does

* `-v mydata:/source` → mounts Docker **named volume** into container at `/source`.
* `-v $(pwd):/backup` → **bind mount** host directory into container at `/backup`.

### What the `tar` command does

* `-C /source` → change directory into `/source`.
* `.` → include **all files inside `/source`**.
* `/backup/backup.tar.gz` → write backup file to host.

---

## **2. Restore From Backup File**

### Create volume if needed:

```bash
docker volume create mydata
```

### Restore

```bash
docker run --rm \
  -v mydata:/target \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/backup.tar.gz -C /target
```

---

## **3. Manual Backup (Not Recommended)**

Copying volume data directly from the filesystem:

```bash
sudo cp -r /var/lib/docker/volumes/mydata/_data ./backup_folder
```

**Risks:**

* Can corrupt data if DB is running.
* Requires root access.
* Not portable.

Use only for simple file-based apps.

---

## **4. External Storage Workflow (EBS/EFS/S3)**

Docker **only creates the backup file locally**. You must manually upload the `.tar.gz`:

* To S3:

  ```bash
  aws s3 cp backup.tar.gz s3://mybucket/backups/
  ```
* To another server (SCP/rsync)
* To EFS/NFS/External storage

For EBS/EFS used with Docker:

1. Mount EBS/EFS on host (`/mnt/ebs_data` or `/mnt/efs_data`).
2. Bind-mount into container or create Docker volume using `--opt o=bind`.
3. Use the same backup/restore commands.

---

## **Common Issues / Errors**

### **1. Permission Denied**

Occurs when restoring as root but app runs as non-root.

**Fix:**

```bash
sudo chown -R 1000:1000 /var/lib/docker/volumes/mydata/_data
```

### **2. Database Corruption**

Backing up while DB is running.

**Fix:**

```bash
docker stop mysql
```

### **3. Wrong Path or Empty Backup**

If you mount wrong path (`-v mydata:/wrongpath`), tar backs up an empty directory.

### **4. Anonymous Volumes Left Behind**

Containers created without `--rm` leave anonymous volumes.

**Fix:**

```bash
docker volume prune
```

### **5. Large Backup Size**

Exclude unnecessary files:

```bash
tar czf backup.tar.gz --exclude=logs .
```

---

## **Troubleshooting / Fixes**

* Verify mounts:

  ```bash
  docker inspect <container> | grep Mounts -A 10
  ```
* Check backup file integrity:

  ```bash
  tar tzf backup.tar.gz
  ```
* Test restore using a temporary volume:

  ```bash
  docker volume create test_restore
  ```
* For EFS: ensure NFS port 2049 open in SG.
* For EBS: verify correct `/etc/fstab` entry.

---

## **Best Practices / Tips**

* Always backup before upgrading stateful containers.
* Never store backups on the same host permanently.
* Upload backup files to S3 or remote storage.
* Use **named volumes** for clean separation of app data.
* Stop databases before backup for clean snapshots.
* Use timestamps in backup filenames.
* Use EBS snapshots (for EBS-mounted volumes) or AWS Backup (for EFS).
* Validate backups periodically with test restores.

---
---

# Docker Volume Permissions Issues — Detailed Explanation Notes

## **Concept / What**

**Permissions issues in Docker volumes** occur when a container's internal user (UID/GID) cannot read or write to a mounted directory or volume. This happens because host-side file ownership and container user IDs may not match. These issues appear with both **bind mounts** and **named volumes**, causing errors like:

* `Permission denied`
* `Read-only file system`
* Application startup failures
* Database initialization errors

---

## **Why / Purpose / Real-World Use Cases**

### **Why permissions matter**

* Containers often run as **non-root** for security.
* Host directories bring **host permissions** into the container when bind-mounted.
* Named volumes default to **root:root**, but containers may run as other users.
* Database containers (MySQL, PostgreSQL) require writable data directories.
* Web servers (Nginx) need write access to logs.
* EFS/EBS mounts inherit their own UID/GID requirements.

### **Use Cases**

* Bind-mounting application code into Node or Python containers.
* MySQL/Postgres using named volumes.
* Jenkins using `/var/jenkins_home` with UID mismatch.
* Sharing folders across multiple containers.
* Mounting EFS/EBS to store persistent data.

---

## **How It Works / Where Issues Come From**

A permissions issue happens when:

* The **host directory** is owned by root but the container runs as UID 1000.
* A **named volume** is owned by root but the application expects its own user.
* The container user (e.g., `node` user UID 1000) tries to write to a directory it doesn't own.
* EFS/EBS directories have restrictive permissions.

Docker does **not** translate permissions between host and container. UID/GID numbers must match.

---

## **How to Fix — Three Valid Fixing Methods**

Choose the method based on the volume type and the situation.

---

## **1. Fix Permissions on the Host (for Bind Mounts)**

Bind mounts inherit host directory permissions. If the host directory is owned by root, non-root containers cannot use it.

### Use when:

* You bind mount a host path.
* EFS/EBS mount is used.
* Host owns the actual directory.

### Fix:

```bash
sudo chown -R 1000:1000 /home/user/app
sudo chmod -R 775 /home/user/app
```

### Why it works:

The container sees the **host's filesystem**, so fixing the host fixes the container.

---

## **2. Run the Container with Matching UID/GID**

Make container user match the host directory's UID.

### Use when:

* Host folder has fixed UID/GID.
* You want container to "act" like the host user.

### Fix:

```bash
docker run --user 1000:1000 -v /home/user/app:/app myimage
```

### Why it works:

Matching UID allows access without modifying host permissions.

---

## **3. Fix Permissions Inside the Container (for Named Volumes)**

Named volumes are fully controlled by Docker, so adjusting permissions inside the container works.

### Use when:

* Using **named volumes** like `/var/lib/docker/volumes/data/_data`.
* Application initializes its own data directory.

### Fix in entrypoint script:

```bash
chown -R appuser:appuser /app/data
exec "$@"
```

### Or fix in Dockerfile:

```dockerfile
RUN mkdir -p /app/data && chown -R appuser:appuser /app/data
```

### Why it works:

Named volumes are created with root by default, and container can safely adjust ownership.

---

## **Common Issues / Errors**

### **1. Permission Denied**

Container user cannot access host or volume directory.

### **2. Read-Only File System**

Host or EBS mount applied incorrectly (e.g., wrong `fstab` settings).

### **3. MySQL/Postgres Startup Failure**

Cannot initialize `/var/lib/mysql` or `/var/lib/postgresql` due to UID mismatch.

### **4. Nginx/Node Cannot Write Logs**

Log folder is mounted but not writable.

### **5. EFS Permissions Mismatch**

EFS often mounts with `root` owner; containers running as UID 1000 cannot write.

---

## **Troubleshooting / Fixes**

* **Check container user UID:**

  ```bash
  docker exec -it container id
  ```
* **Fix host-folder permissions:**

  ```bash
  sudo chown -R 1000:1000 /mnt/efs/appdata
  ```
* **Validate volume mount:**

  ```bash
  docker inspect <container> | grep Mounts -A 10
  ```
* **Test with temporary Alpine container:**

  ```bash
  docker run -it --rm -v mydata:/data alpine sh
  ```
* **For EFS:** ensure SG allows NFS port 2049.
* **For EBS:** confirm filesystem mounted read-write.

---

## **Best Practices / Tips**

* For **bind mounts**: always fix permissions on the host.
* For **named volumes**: fix inside container or match UID using `--user`.
* Avoid running applications as root unless required.
* Avoid bind mounts in production for persistent data.
* Use separate volumes for logs, data, uploads.
* For EFS/EBS: set correct ownership before mounting into containers.
* In Kubernetes/EKS: use InitContainers to adjust storage permissions.

---
---


