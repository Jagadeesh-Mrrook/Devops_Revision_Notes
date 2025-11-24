# Running Containers (Detached / Interactive)

## **1. Concept / What**

Running containers means starting an instance of a Docker image. Containers can run in two modes:

* **Detached Mode (`-d`)** → runs the container in the background.
* **Interactive Mode (`-it`)** → gives a terminal inside the container for debugging.

---

## **2. Why / Purpose / Real-World Use Cases**

### **Detached Mode**

Used when running long-running applications such as:

* Java/Spring Boot microservices
* Databases (MySQL, Redis)
* API servers (Nginx, Node.js)
* Services managed by CI/CD
* Containers deployed to EKS/k8s

### **Interactive Mode**

Used mainly for:

* Debugging containers
* Executing manual commands
* Checking configurations, libraries, environment variables
* Temporary testing

---

## **3. How It Works / Steps / Syntax**

### **Run a container in detached mode**

```bash
docker run -d --name app1 -p 8080:8080 jagga/java-app:v1
```

* `-d` → run in background
* `--name` → container name
* `-p` → port mapping

### **Run a container in interactive mode**

```bash
docker run -it ubuntu:22.04
```

Gives a shell directly inside the container.

### **Run Java-based image in interactive mode for debugging**

```bash
docker run -it --entrypoint bash jagga/java-app:v1
```

Override the default entrypoint to open a shell.

### **Run a temporary container and auto-remove**

```bash
docker run -it --rm maven:3.9-eclipse-temurin-17 bash
```

### **Debug a running container using docker exec**

(Used only when container was started in detached mode)

```bash
docker exec -it app1 bash
```

---

## **4. Common Issues / Errors**

### **1. Port conflicts**

```
Error: driver failed programming external connectivity
```

**Fix:** free or change the port.

```bash
sudo lsof -i :8080
sudo kill <pid>
```

### **2. Container exits immediately**

Occurs due to:

* Wrong CMD/ENTRYPOINT
* Missing JAR file
* Wrong working directory
* Application crash

Check logs:

```bash
docker logs <container>
```

### **3. Cannot enter stopped container using exec**

`docker exec` works only on running containers.

### **4. Interactive container exits when shell is closed**

Expected behavior: interactive mode stops when you exit.

---

## **5. Troubleshooting / Fixes**

### **Check logs**

```bash
docker logs app1
```

### **Inspect container configuration**

```bash
docker inspect app1
```

### **Get a shell inside a running container**

```bash
docker exec -it app1 bash
```

### **Debug file system**

```bash
docker run -it --entrypoint bash jagga/java-app:v1
```

---

## **6. Best Practices / Tips**

* Always run real applications in **detached mode**.
* Use **interactive mode** only for temporary manual debugging.
* Use `docker exec` only when the container is running in background.
* Expose only required ports.
* Keep Dockerfiles clean and optimized.
* Follow proper tagging (`app:v1`, `app:latest`, `app:<build-id>`).
* For EKS, prefer non-root user images.
* Ensure correct CMD/ENTRYPOINT so the container doesn't exit immediately.

---
---

# Restart Policies

## **1. Concept / What**

Restart policies define how Docker should automatically restart a container if it stops or crashes. By default, Docker does **not** restart containers unless a restart policy is applied.

Docker provides four restart policy types:

* `no` (default)
* `on-failure`
* `always`
* `unless-stopped`

---

## **2. Why / Purpose / Real-World Use Cases**

Restart policies ensure containers continue running without manual intervention. They are useful for:

* Java/Spring Boot microservices
* API servers
* Background agents
* Monitoring tools
* Applications running on EC2 outside Kubernetes
* Auto-starting services after system reboot

They provide basic self-healing for containers in non-Kubernetes environments.

---

## **3. How It Works / Steps / Syntax**

### **1. No Restart (default)**

Container will not restart automatically.

```bash
docker run --restart=no app
```

### **2. Restart on Failure**

Restarts only if container exits with a non-zero status.

```bash
docker run -d --restart=on-failure:5 app
```

* `5` = maximum restart attempts

### **3. Always Restart**

Restarts the container no matter how it stopped (even after manual stop).

```bash
docker run -d --restart=always app
```

### **4. Unless Stopped**

Restarts the container unless **you manually stopped it**.

```bash
docker run -d --restart=unless-stopped app
```

Used most commonly for long-running applications.

---

## **4. Common Issues / Errors**

### **1. Infinite restart loop**

Occurs due to:

* Wrong CMD/ENTRYPOINT
* Missing JAR file
* Crashing application

Check logs:

```bash
docker logs <container>
```

### **2. Container restarts even after manual stop**

Happens with `always` policy.
Use `unless-stopped` instead.

### **3. Container didn't start after system reboot**

Cause: restart policy not set.
Fix:

```bash
docker update --restart=unless-stopped <container>
```

---

## **5. Troubleshooting / Fixes**

* Check container logs

```bash
docker logs app
```

* Inspect restart policy configuration

```bash
docker inspect app
```

* Update restart policy without recreating container

```bash
docker update --restart=on-failure:3 app
```

---

## **6. Best Practices / Tips**

* Use `unless-stopped` for microservices running on EC2.
* Use `on-failure` for unstable workloads with limited retries.
* Avoid `always` unless you fully understand its behavior.
* Ensure correct CMD/ENTRYPOINT to avoid crash loops.
* Use proper logging for crash analysis.

---
---

# Environment Variables

## **1. Concept / What**

Environment variables allow you to pass configuration values to containers at runtime without rebuilding the image. They are used to supply settings like:

* Database host
* Usernames/passwords
* Environment names (dev/qa/prod)
* Ports
* Application flags

Environment variables can be defined in the Dockerfile or supplied during container startup.

---

## **2. Why / Purpose / Real-World Use Cases**

Environment variables are used because:

* One Docker image can run in multiple environments by changing only variables.
* Developers define the variables the application expects, and DevOps passes them.
* CI/CD pipelines (Jenkins) inject dynamic values.
* ECS/EKS override environment variables per environment.
* They avoid hardcoding configuration into Docker images.

Real use cases include:

* Setting DB connection details
* Selecting Spring Boot profiles
* Passing JVM memory options
* Configuring service URLs per environment

---

## **3. How It Works / Steps / Syntax**

### **1. Passing variables using `-e`**

```bash
docker run -d \
  -e DB_HOST=10.0.2.15 \
  -e DB_USER=admin \
  -e SPRING_PROFILES_ACTIVE=prod \
  jagga/payment-app:v1
```

### **2. Using `.env` file**

`.env` file:

```
DB_USER=admin
DB_PASS=secret123
DB_HOST=prod.database.local
```

Run:

```bash
docker run --env-file .env jagga/payment-app:v1
```

### **3. View environment variables inside container**

```bash
docker exec -it payment-app env
```

Or:

```bash
docker exec -it payment-app printenv DB_HOST
```

### **4. Set ENV inside Dockerfile (default values)**

```dockerfile
ENV APP_PORT=8080
ENV JAVA_OPTS="-Xms512m -Xmx1024m"
```

These can be overridden at runtime.

### **5. Runtime Java example using env variables**

```dockerfile
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

Run:

```bash
docker run -d -e JAVA_OPTS="-Xmx2g" app:v1
```

---

## **4. Common Issues / Errors**

### **1. Missing variables**

Application may crash if required variables are not passed.

```
DB_HOST not found
```

### **2. Incorrect formatting in `.env` file**

Wrong:

```
DB_HOST = localhost
```

Correct:

```
DB_HOST=localhost
```

### **3. Variable passed but not picked by app**

Caused by:

* Wrong variable name
* Incorrect application config syntax
* ENTRYPOINT not referencing the variable

### **4. ARG vs ENV confusion**

`ARG` = build-time only (not available at runtime).
`ENV` = available during build and runtime.

---

## **5. Troubleshooting / Fixes**

* Check actual values inside container:

```bash
docker exec -it app env
```

* Verify application logs:

```bash
docker logs app
```

* Ensure correct variable names given by developers.
* Confirm Dockerfile uses `ENV` for runtime variables.

---

## **6. Best Practices / Tips**

* Never hardcode secrets in Dockerfiles.
* Use `--env-file` for multiple variables.
* Use ENV for runtime configuration.
* Use ARG only for build-time parameters.
* Follow consistent naming across all environments.
* Developers decide variable names; DevOps only implements them.

---
---

# Inspecting & Debugging Containers

## **1. Concept / What**

Inspecting and debugging containers means checking:

* Container configuration
* Application logs
* Running processes
* CPU/memory usage
* File system inside container
* Network settings

Using commands like:

* `docker logs`
* `docker exec`
* `docker inspect`
* `docker top`
* `docker stats`

These help identify why a container or the application inside it is failing.

---

## **2. Why / Purpose / Real-World Use Cases**

Used when DevOps engineers need to find:

* Why a Java application crashed
* Missing or wrong environment variables
* Wrong port configuration
* Incorrect CMD/ENTRYPOINT
* Continuous restarts / crash loops
* Internal application errors
* Network or connectivity problems

---

## **3. How It Works / Steps / Syntax**

### **1. View container logs**

Used for **application-level errors**.

```bash
docker logs app1
```

Follow live logs:

```bash
docker logs -f app1
```

### **2. Inspect container configuration**

Used for **container-level issues**.

```bash
docker inspect app1
```

Shows:

* Ports
* Environment variables
* Volumes
* Network details
* CMD/ENTRYPOINT
* Restart policy

### **3. Enter a running container (debug shell)**

```bash
docker exec -it app1 bash
```

Used to:

* View internal files
* Check configs
* Inspect logs inside container
* Test connectivity (`curl`)

### **4. View processes running inside container**

```bash
docker top app1
```

### **5. Check container resource usage**

```bash
docker stats app1
```

Shows container-level:

* CPU
* Memory
* Network I/O
* Block I/O

### **6. Check environment variables inside container**

```bash
docker exec -it app1 env
```

### **7. Inspect container network**

```bash
docker inspect app1 | grep IPAddress
```

---

## **4. Common Issues / Errors**

* Wrong port mapping
* Wrong CMD or ENTRYPOINT
* Missing JAR file
* Wrong working directory
* Incorrect environment variables
* Application throwing errors
* Crash loops due to configuration

---

## **5. Troubleshooting / Fixes**

### **1. Check application logs first**

```bash
docker logs app1
```

### **2. Enter container to debug**

```bash
docker exec -it app1 bash
```

### **3. Validate environment variables**

```bash
docker exec -it app1 env
```

### **4. Verify JAR and config files**

```bash
ls -l /app
cat /app/config/application.properties
```

### **5. Validate resource consumption**

```bash
docker stats app1
```

### **6. Inspect container configuration**

```bash
docker inspect app1
```

---

## **6. Best Practices / Tips**

* Use `docker logs` to debug crashed applications.
* Use `docker exec` to inspect running containers.
* Use `docker stats` for CPU/memory monitoring.
* Inspect configuration using `docker inspect`.
* Ensure correct CMD/ENTRYPOINT in Dockerfile.
* Verify ports and environment variables before running.
* Always check logs before rebuilding images.

---
---

# Resource Limits (CPU / Memory)

## **1. Concept / What**

Docker allows limiting how much CPU and memory a container can use. These limits prevent a single application from consuming all system resources.

Resource limiting is done using Linux **cgroups (Control Groups)**, and Docker provides CLI flags to configure these limits.

Common flags:

* `--memory` → memory limit
* `--cpus` → CPU limit
* `--cpu-shares` → CPU priority

---

## **2. Why / Purpose / Real-World Use Cases**

Resource limits are used to:

* Prevent Java apps from consuming too much RAM
* Avoid container crashes due to OOM (Out Of Memory)
* Keep multiple microservices stable on the same server
* Match local Docker limits with Kubernetes (EKS/ECS) resource settings
* Stop a "noisy neighbor" container from taking all CPU

Used heavily when running:

* Java/Spring Boot services
* Node.js/Go microservices
* Jenkins agents
* Background processing applications

---

## **3. How It Works / Steps / Syntax**

### **1. Set Memory Limits**

```bash
docker run -d --memory=512m app:v1
```

Limits container memory to 512 MB.

### **2. Set CPU Limits**

```bash
docker run -d --cpus=1.5 app:v1
```

Gives the container 1.5 CPU cores worth of processing.

### **3. Combine CPU + Memory Limits**

```bash
docker run -d --memory=1g --cpus=2 jagga/orders:v1
```

### **4. Check Container Resource Usage**

```bash
docker stats app1
```

Shows live CPU/memory usage.

### **5. Check Memory Limit Inside Container**

```bash
docker exec -it app1 cat /sys/fs/cgroup/memory/memory.limit_in_bytes
```

### **6. Tune JVM Memory Based on Limits**

```bash
docker run -d --memory=512m -e JAVA_OPTS="-Xms256m -Xmx400m" app:v1
```

---

## **4. Common Issues / Errors**

* **OOMKilled** → Memory limit too low for Java
* Continuous restarts due to low CPU/memory
* Slow response times due to CPU throttling
* JVM memory flags (`Xmx`) higher than Docker memory limit

---

## **5. Troubleshooting / Fixes**

### **1. Check live resource usage**

```bash
docker stats app1
```

### **2. Check logs for OOM errors**

```bash
docker logs app1
```

### **3. Adjust JVM memory settings**

```bash
-e JAVA_OPTS="-Xmx300m"
```

### **4. Increase container limits**

```bash
docker run --memory=1g --cpus=2 app:v1
```

### **5. Inspect container configuration**

```bash
docker inspect app1
```

---

## **6. Best Practices / Tips**

* Always set memory limits for Java services.
* Match Docker resource limits with Kubernetes limits for consistency.
* Use `docker stats` regularly to monitor container performance.
* Tune JVM (`Xms`/`Xmx`) according to container memory.
* Avoid giving containers unlimited CPU/RAM.
* Remember: Docker uses **Linux cgroups** to enforce resource limits.

---
---

# Logs and Shell Access

## **1. Concept / What**

Applications running inside containers generate logs in **two ways**:

* **Console logs (stdout/stderr)** → viewed using `docker logs`.
* **File-based logs (application log files)** → viewed by entering the container using `docker exec`.

Shell access allows inspecting container internals, configurations, log files, and running processes.

---

## **2. Why / Purpose / Real-World Use Cases**

Logs and shell access are used to:

* Debug application errors
* Find why a Java/Spring Boot app failed
* Check environment variables
* Validate config files
* Inspect filesystem paths
* Verify JAR existence
* Check container-level behavior
* Troubleshoot microservice connectivity issues

Common scenarios:

* App crash
* Missing configuration
* Port issues
* Wrong environment variables
* JVM errors
* Database connection failures

---

## **3. How It Works / Steps / Syntax**

### **1. View console logs (application stdout/stderr)**

```bash
docker logs app1
```

Follow logs live:

```bash
docker logs -f app1
```

Shows:

* Startup output
* Exceptions
* Errors
* Crash logs

### **2. Enter container shell (for internal log files)**

```bash
docker exec -it app1 bash
```

Or:

```bash
docker exec -it app1 sh
```

Used to:

* Navigate to log directories
* Check file-based logs
* Inspect configs
* Check JAR files

### **3. View application log files inside container**

```bash
docker exec -it app1 cat /app/logs/app.log
```

### **4. View running processes inside container**

```bash
docker top app1
```

### **5. Check container resource usage**

```bash
docker stats app1
```

### **6. Check environment variables inside container**

```bash
docker exec -it app1 env
```

### **7. Test internal network connectivity**

```bash
docker exec -it app1 curl http://localhost:8080/actuator/health
```

---

## **4. Common Issues / Errors**

* Container crashes on startup → check `docker logs`
* App not writing logs to stdout → check file-based logs via `docker exec`
* Wrong configuration/path errors
* Missing JAR file
* Incorrect ports
* Wrong environment variables
* JVM initialization errors

---

## **5. Troubleshooting / Fixes**

### **1. Check console logs first**

```bash
docker logs -f app1
```

### **2. Use shell access for deeper debugging**

```bash
docker exec -it app1 bash
```

### **3. Validate application logs (file-based)**

```bash
cat /app/logs/app.log
```

### **4. Confirm environment variables**

```bash
docker exec -it app1 env
```

### **5. Verify JAR/config files**

```bash
ls -l /app
cat application.properties
```

### **6. Check container processes**

```bash
docker top app1
```

### **7. Inspect resource usage**

```bash
docker stats app1
```

---

## **6. Best Practices / Tips**

* Use `docker logs` for all console-level debugging.
* Use `docker exec` only for running containers.
* Check file-based logs for detailed request/transaction logs.
* Avoid using `docker attach` in production.
* Prefer `bash`; use `sh` for lightweight images.
* Always check logs before rebuilding an image.
* Use `curl` inside container to debug microservice connectivity.

---
---

# Copy Files In/Out of Containers

## **1. Concept / What**

`docker cp` allows copying files **from host to container** or **from container to host**. It works even if the container is stopped.

This is used to transfer:

* Log files
* Configuration files
* JARs (temporary debugging)
* Certificates
* Scripts
* Debug output (heap dumps, thread dumps)

---

## **2. Why / Purpose / Real-World Use Cases**

`docker cp` is used for:

* Extracting application logs for troubleshooting
* Copying configuration files into container for testing
* Retrieving JVM heap dump/trace files
* Injecting test scripts or temporary tools
* Debugging file-related issues inside the container
* Backing up important directories from a running/stopped container

Common DevOps scenarios:

* App crash → copy logs out
* Developer gives temporary config → copy inside
* Need to examine JAR contents inside container
* Need to analyze heap dump or thread dump

---

## **3. How It Works / Steps / Syntax**

### **1. Copy file from container → host**

```bash
docker cp app1:/app/logs/app.log .
```

Copies `/app/logs/app.log` to the current directory.

### **2. Copy directory from container → host**

```bash
docker cp app1:/var/logs ./logs_backup
```

### **3. Copy file from host → container**

```bash
docker cp application.properties app1:/app/config/
```

Useful for testing configuration.

### **4. Copy JAR to container (debugging only)**

```bash
docker cp app-fixed.jar app1:/app/
```

(Not recommended for production.)

### **5. Copy script to container**

```bash
docker cp test.sh app1:/tmp/
```

Run it after entering container:

```bash
docker exec -it app1 bash
chmod +x /tmp/test.sh
/tmp/test.sh
```

### **6. Copy from stopped container**

```bash
docker cp stopped_app:/app/logs/app.log .
```

This works because Docker keeps the filesystem.

---

## **4. Common Issues / Errors**

### **1. Path does not exist**

Fix by verifying inside container:

```bash
docker exec -it app1 bash
ls -l /app
```

### **2. Permission denied**

Run shell as root:

```bash
docker exec -u root -it app1 bash
```

### **3. Copying large folders is slow**

Use tar for faster transfer:

```bash
docker exec app1 tar czf /tmp/logs.tar.gz /app/logs
docker cp app1:/tmp/logs.tar.gz .
```

---

## **5. Troubleshooting / Fixes**

* Verify destination paths
* Ensure container name is correct
* Check permissions using root user
* Confirm file exists before copying
* Use tar for large file transfers

---

## **6. Best Practices / Tips**

* Use `docker cp` mainly for debugging.
* Do **not** modify production containers using `docker cp`.
* Always rebuild the image for permanent changes.
* Confirm file paths with `docker exec` before copying.
* Use it to extract logs, heap dumps, and config files safely.
* Remember: `app1` refers to the **container name**.

---
---
---


