# Dockerfile Core Instructions: FROM, RUN, COPY, WORKDIR, ENV

## **Concept / What**

### **FROM**

Defines the base image from which your image will be built.

### **RUN**

Executes commands during the image build process (install packages, update system, etc.).

### **COPY**

Copies files/directories from the **build context** into the image.

### **WORKDIR**

Sets the working directory for subsequent Dockerfile instructions.

### **ENV**

Sets environment variables inside the image.

---

## **Why / Purpose / Real-World Use Cases**

### **FROM**

* Select OS/runtime (Python, Node, Java, Ubuntu, Alpine).
* Controls security patches and image size.
* Used to prepare production-ready, lightweight images.

### **RUN**

* Install dependencies in CI/CD.
* Configure tools and patch OS during build.

### **COPY**

* Move application source code, configs, requirements files into images.
* Ensures reproducible builds in CI/CD, EKS, microservices.

### **WORKDIR**

* Avoids path/"file not found" issues.
* Ensures all app-related commands run in a consistent directory.

### **ENV**

* Store non-sensitive runtime configuration.
* Works with Kubernetes/EKS `env:` settings.

---

## **How It Works / Steps / Syntax**

### **Basic Example: Python App Dockerfile**

```dockerfile
FROM python:3.10-slim

ENV APP_ENV=prod \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

### **Command Workflow (Build → Tag → Run)**

```
docker build -t myapp:1.0 .
docker run -p 8080:8080 myapp:1.0
```

---

## **Common Issues / Errors**

### ❗ COPY Issues

* "No such file or directory" → File not inside **build context**.
* Wrong relative paths.

### ❗ RUN Errors

* `apt-get install` failing due to missing update.
* Incorrect package names.

### ❗ ENV Issues

* App expects a variable not set in Dockerfile.
* Wrong variable name formatting.

### ❗ WORKDIR Issues

* Commands fail because directory doesn't exist.
* Using incorrect paths before setting WORKDIR.

---

## **Troubleshooting / Fixes**

### Check environment variables in container

```
docker exec -it <container> env
```

### Debug image building

```
docker build --progress=plain .
```

### Fix RUN apt issues

```dockerfile
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

### Fix COPY failures

* Make sure file exists inside build context.
* Correct COPY syntax:

```
COPY src/ /app/src/
```

---

## **Best Practices / Tips**

### **FROM**

* Use minimal base images: `slim`, `alpine`, or language-specific minimal images.

### **RUN**

* Combine multiple commands into one layer.
* Clean up cache to reduce image size.

### **COPY**

* Use `.dockerignore` to avoid copying unnecessary files:

```
.git
node_modules
*.log
venv
```

### **WORKDIR**

* Always define a single working directory.

### **ENV**

* Do not store secrets.
* Use for runtime configs.

---
---

# CMD vs ENTRYPOINT in Docker

## **Concept / What**

### **CMD**

Defines the **default command or default arguments** a container will run **after it starts**.

* Can be **overridden** by user arguments during `docker run`.

### **ENTRYPOINT**

Defines the **main executable** the container must always run.

* **Cannot be overridden** by user arguments.
* User arguments are **appended** to ENTRYPOINT.
* Can be overridden ONLY using `--entrypoint` flag.

---

## **Why / Purpose / Real-World Use Cases**

### **CMD**

* Used for providing **default flags/arguments**.
* Good for simple apps where user may override the command.
* Example in microservices: default port, default JVM flags.

### **ENTRYPOINT**

* Ensures a specific executable always runs.
* Useful in production containers where the app must always be executed.
* Used heavily for Java-based microservices (Spring Boot JARs).

### **CMD + ENTRYPOINT Together**

* ENTRYPOINT = main command
* CMD = default arguments
* User can override CMD but cannot override ENTRYPOINT (without using `--entrypoint`).

---

## **How It Works / Steps / Syntax**

### **1. CMD Examples**

```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY target/myapp.jar .
CMD ["java", "-jar", "myapp.jar"]
```

**Override CMD at runtime:**

```
docker run myapp java -jar myapp.jar --spring.profiles.active=dev
```

---

### **2. ENTRYPOINT Examples**

```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY target/myapp.jar .
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

**User arguments get appended:**

```
docker run myapp --spring.profiles.active=prod
```

Actual execution:

```
java -jar myapp.jar --spring.profiles.active=prod
```

---

### **3. ENTRYPOINT + CMD Together (Recommended Pattern)**

```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY target/myapp.jar .
ENTRYPOINT ["java", "-jar", "myapp.jar"]
CMD ["--server.port=8080"]
```

**CMD Overrides:**

```
docker run myapp --server.port=9090
```

Result:

```
java -jar myapp.jar --server.port=9090
```

---

### **4. Override ENTRYPOINT Explicitly (Rare)**

```
docker run --entrypoint "/bin/bash" myapp
```

This bypasses the ENTRYPOINT entirely.

---

## **Common Issues / Errors**

### ❗ CMD ignored when ENTRYPOINT is present

* ENTRYPOINT overrides the execution path.
* CMD becomes only arguments.

### ❗ User tries to override ENTRYPOINT without `--entrypoint`


Docker won’t override it.

### ❗ Using shell form incorrectly

Shell form:

```
ENTRYPOINT java -jar myapp.jar
```

Exec form (recommended):


```
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

Shell form may cause signal handling issues.

---

## **Troubleshooting / Fixes**

### Fix: ENTRYPOINT not accepting arguments

Use exec form:

```
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

### Fix: Need debugging shell instead of app startup

```
docker run -it --entrypoint sh myapp
```

### Fix: Need different command temporarily

```
docker run --entrypoint /bin/bash myapp
```

---

## **Best Practices / Tips**

### ENTRYPOINT

* Use ENTRYPOINT for main app execution.
* Always in exec form.
* Good for Java microservices:

```
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

### CMD

* Use for default flags.
* Allow users to override JVM or Spring Boot parameters.

### ENTRYPOINT + CMD

* Best production pattern.
* ENTRYPOINT = fixed
* CMD = customizable

### Avoid Shell Form

* Causes PID 1 issues, signal problems, and unexpected behavior.

---
---

# Dockerfile HEALTHCHECK Instruction

## **Concept / What**

**HEALTHCHECK** in a Dockerfile defines a command that Docker will run inside the container to determine whether the application is **healthy**.

* Docker marks containers as: **healthy**, **unhealthy**, or **starting**.
* It is optional and mostly used for standalone Docker deployments.

---

## **Why / Purpose / Real-World Use Cases**

### **When HEALTHCHECK Is Useful**

* When running containers **without Kubernetes** (Docker Swarm, Docker Compose, bare Docker hosts).
* For monitoring app health at the **container runtime level**.
* For restarting unhealthy containers when using Docker's built-in restart policies.

### **When HEALTHCHECK Is *Not* Needed**

* **Not required in Kubernetes/EKS**, because:

  * Kubernetes uses **liveness** and **readiness** probes for health.
  * Kubernetes does not use Dockerfile HEALTHCHECK.
  * EKS decisions are based on probes defined in Pod YAML.

So for EKS deployments, HEALTHCHECK in Dockerfile is usually skipped.

---

## **How It Works / Steps / Syntax**

### **Example: Java Spring Boot App HEALTHCHECK**

```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY target/myapp.jar .

# Main application startup
ENTRYPOINT ["java", "-jar", "myapp.jar"]

# Healthcheck
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
```

### Explanation

* **--interval** → time between checks
* **--timeout** → how long to wait before marking check as failed
* **--retries** → how many failures before marking container *unhealthy*
* **CMD** → actual command executed to check health

Docker marks container:

```
healthy     → if curl returns 200
unhealthy   → if curl fails
```

---

## **Common Issues / Errors**

### ❗ Curl not found

If base image is minimal (Alpine, Distroless), `curl` may not exist.

### ❗ Checking wrong port

If Spring Boot app uses dynamic port or non-standard port, the healthcheck fails.

### ❗ Healthcheck runs before app starts

App takes time to start → first few checks may fail.

---

## **Troubleshooting / Fixes**

### Fix: Install curl

```dockerfile
RUN apk add --no-cache curl
```

(For Alpine-based images)

### Fix: Use correct health endpoint

Adjust to your app:

```
/actuator/health
/health
/status
```

### Fix: Delay first healthcheck

```
HEALTHCHECK --start-period=20s ...
```

---

## **Best Practices / Tips**

### If deploying to **EKS/Kubernetes** → **DO NOT** use Dockerfile HEALTHCHECK.

Use K8s probes instead:

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5

livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

### Dockerfile HEALTHCHECK is useful only when:

* Running in **Docker Compose**
* Running in **Docker Swarm**
* Running containers directly on VMs without Kubernetes

### For Java Apps

* Prefer `/actuator/health` endpoint for health checks.
* Ensure your image includes curl or use Java-based healthcheck:

```dockerfile
HEALTHCHECK CMD wget -qO- http://localhost:8080/actuator/health || exit 1
```

---
---

# Dockerfile Instructions: EXPOSE & USER

## **Concept / What**

### **EXPOSE**

Tells Docker **which port the application inside the container listens on**.

* It does **not** publish or open the port.
* Acts as **documentation** for developers and tools.

### **USER**

Sets the **Linux user** under which the container will run.

* Helps avoid running containers as `root`.
* Improves security for production workloads.

---

## **Why / Purpose / Real-World Use Cases**

### **EXPOSE**

* Helps others understand which port the application uses.
* Useful for tools like Docker Desktop, Compose, and scanners.
* Standard practice in images for Java apps (Spring Boot default port 8080).

### **USER**

* Running containers as root is a **security risk**.
* Kubernetes/EKS best practice: run application as a **non-root** user.
* Helps prevent privilege escalation.

---

## **How It Works / Steps / Syntax**

### **Example: Java Spring Boot App**

```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY target/myapp.jar .

# Expose internal application port
EXPOSE 8080

# Create non-root user
RUN useradd -u 1001 appuser

# Switch to non-root user
USER appuser

ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

### **Port Mapping at Runtime**

EXPOSE does NOT open a port.
You must map ports during `docker run`:

```
docker run -p 8080:8080 myapp
```

### **Running as Non-Root in Kubernetes**

```yaml
securityContext:
  runAsUser: 1001
  runAsNonRoot: true
```

(Kubernetes completely ignores Dockerfile `USER` but it's still a good practice.)

---

## **Common Issues / Errors**

### ❗ EXPOSE misunderstood as port publishing

People think EXPOSE opens ports—it does NOT.
Container won’t be reachable without `-p` mapping.

### ❗ Permission errors when using USER

If paths are owned by root:

```
Permission denied: cannot write
```

Fix:

```
RUN chown -R appuser:appuser /app
```

### ❗ Java application fails to bind privileged ports

Non-root cannot bind ports below 1024.
Examples:

* 80
* 443

---

## **Troubleshooting / Fixes**

### Fix: Permission Denied

Ensure non-root user owns working directory:

```dockerfile
RUN chown -R appuser:appuser /app
```

### Fix: Application runs on wrong port

Set port using environment variables:

```dockerfile
ENV SERVER_PORT=8080
```

Or for Spring Boot:

```
--server.port=8080
```

### Fix: Need root for a specific command

You can briefly switch user:

```dockerfile
USER root
RUN apt-get update && apt-get install -y curl
USER appuser
```

---

## **Best Practices / Tips**

### EXPOSE

* Always document the port the app listens on.
* Does NOT expose to outside world—only documentation.

### USER

* Always run Java applications as **non-root**.
* Create a dedicated user for the application.
* Kubernetes ignores Dockerfile USER but it’s good hygiene.

### For Java Apps

* EXPOSE 8080, 9090, or whatever port your Spring Boot app uses.
* Ensure `/app` directory permissions support non-root user.

---
---

# .dockerignore in Docker

## **Concept / What**

The **.dockerignore** file tells Docker which files or folders to **exclude** from the build context during `docker build`.

* Works similar to `.gitignore`.
* Anything matched in `.dockerignore` will NOT be sent to the Docker daemon.
* Helps keep images small, builds faster, and prevents copying unwanted or sensitive files.

---

## **Why / Purpose / Real-World Use Cases**

### **1. Smaller Build Context → Faster Builds**

If your project has large files (logs, node_modules, build artifacts), ignoring them makes builds much faster.

### **2. Prevent Sensitive/Unwanted Files from Entering the Image**

Avoid sending:

* `.git` directory
* Credentials/config files
* Temporary files
* IDE files

### **3. Avoid Accidental COPY of Unnecessary Files**

Without `.dockerignore`, a `COPY . .` may copy:

* Build folders
* Large artifacts
* Local environment files
* Secret config files

### **4. Required for Java Projects (Maven/Gradle)**

Java applications generate:

* `target/` (Maven build output)
* `.mvn/`
* `*.class`
  These should not be copied unless required.

---

## **How It Works / Steps / Syntax**

Create a `.dockerignore` file in the same directory as the Dockerfile.

### **Java Microservice Example (.dockerignore)**

```
# Ignore git metadata
.git
.gitignore

# Ignore compiled Java output
*.class

target/

# IDE directories
.idea/
.vscode/

# Logs
*.log

# Local environment files
.env

# OS files
.DS_Store
```

### Build Command

```
docker build -t myapp:1.0 .
```

Docker sends build context → excludes everything in `.dockerignore`.

---

## **Common Issues / Errors**

### ❗ App folder missing inside image

If `.dockerignore` accidentally excludes required folders.
Fix: remove required folders from `.dockerignore`.

### ❗ Large builds

If `.dockerignore` is missing, Docker sends the entire folder (including logs, libraries, artifacts).

### ❗ COPY fails

COPY may fail if file excluded by `.dockerignore`:

```
COPY failed: file not found
```

---

## **Troubleshooting / Fixes**

### Debug Build Context

Run:

```
docker build . --progress=plain
```

You will see which files are being sent.

### Remove unnecessary ignores

Avoid over-aggressive patterns.

### Ensure required files remain

Do not ignore your source code or Maven pom.xml.

---

## **Best Practices / Tips**

### For Java Projects

Use a clean `.dockerignore` to reduce build size:

```
.git
*.log
.idea/
.vscode/
.DS_Store

# Maven/Gradle build output
/target
/build

# Cache
*.iml
```

### General

* Maintain `.dockerignore` just like `.gitignore`.
* Always verify `.dockerignore` when builds go wrong.
* Keep Docker build context as small as possible.

---
---

# Production-Ready Dockerfile Patterns (Java Focus)

## **Concept / What**

Production-ready Dockerfile patterns are **best practices** and **design approaches** that make Docker images:

* Smaller
* Faster to build
* Secure
* Suitable for EKS and CI/CD pipelines
* Easy to debug or extend

These patterns go beyond basic instructions and focus on real-world production reliability.

---

## **Why / Purpose / Real-World Use Cases**

### **1. Reduce Image Size**

Smaller images:

* Build faster
* Push/pull faster (CI/CD → ECR → EKS)
* Reduce attack surface

### **2. Improve Security**

* Use non-root users
* Avoid unnecessary tools in image
* Limit packages to reduce CVEs

### **3. Predictable, Reproducible Builds**

Every environment (dev/qa/prod) gets the same image.

### **4. Java Microservices in Kubernetes/EKS**

* Use slim/minimal base images
* Use ENTRYPOINT + CMD pattern
* Set JVM parameters
* Use layered caching

### **5. Faster CI/CD Pipelines**

Optimized Dockerfiles lead to:

* Faster build time
* Less registry storage
* Faster rollback (smaller images)

---

## **How It Works / Steps / Syntax**

### **1. Use Minimal Base Images**

```dockerfile
FROM eclipse-temurin:17-jre-alpine
```

Benefits:

* Smaller
* Fewer vulnerabilities
* Faster startup

---

### **2. Set Working Directory**

```dockerfile
WORKDIR /app
```

Ensures predictable behavior during build + execution.

---

### **3. Copy Only Required Files**

```dockerfile
COPY target/myapp.jar /app/myapp.jar
```

Avoid copying full project with `COPY . .`.

---

### **4. Add Non‑Root User**

```dockerfile
RUN adduser -D appuser
USER appuser
```

Mandatory for security in production.

---

### **5. Use ENTRYPOINT + CMD Pattern**

```dockerfile
ENTRYPOINT ["java", "-jar", "myapp.jar"]
CMD ["--server.port=8080"]
```

ENTRYPOINT → main command
CMD → default arguments (can be overridden).

---

### **6. JVM Optimization for Containers**

Add JVM options suitable for containers:

```dockerfile
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
```

---

### **7. Add Healthcheck (Only for non-K8s environments)**

```dockerfile
HEALTHCHECK CMD wget -qO- http://localhost:8080/actuator/health || exit 1
```

---

### **8. Use .dockerignore**

```
.git
target/
*.log
.idea/
```

Reduces build context → faster Docker builds.

---

### **9. Avoid Caching Issues**

Install dependencies before copying source code:

```dockerfile
COPY pom.xml .
COPY mvnw .
COPY .mvn .mvn
RUN ./mvnw -q dependency:go-offline

COPY src src
```

This allows Docker to cache Maven dependency layers.

---

## **Common Issues / Errors**

### ❗ Large image size

Cause:

* Using full JDK instead of JRE
* Installing unnecessary tools
* Copying full project instead of the JAR

### ❗ Running as root (Security risk)

Fix: create non-root user.

### ❗ Maven dependencies downloaded every time

Fix: use layered caching pattern.

### ❗ Incorrect ENTRYPOINT format (Shell vs Exec)

Always use exec form:

```
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

---

## **Troubleshooting / Fixes**

### Fix: Image too large

* Switch to Alpine base
* Use multi-stage builds (if building inside Docker)

### Fix: Permission denied as non-root

```
RUN chown -R appuser:appuser /app
```

### Fix: Slow build times

Use `.dockerignore`, layered caching.

---

## **Best Practices / Tips**

### For Java Production Images

* Use **Temurin JRE Alpine** or **Distroless Java 17**.
* Always run as **non-root**.
* Set `JAVA_OPTS` via ENV.
* Use ENTRYPOINT + CMD.
* Maintain clean `.dockerignore`.
* Keep image layers minimal.

### For CI/CD (Jenkins → ECR → EKS)

* Push only optimized images.
* Use caching layers for Maven.
* Ensure deterministic builds.

### For Security

* Do not install extra tools unless needed.
* Keep base image updated.
* Scan images for vulnerabilities.

---
---

# Multi-Stage Docker Builds (Java Focus)

## **Concept / What**

A **multi-stage build** uses **multiple FROM stages** in a Dockerfile so you can:

* Build the application in one stage (builder stage)
* Copy only the final output (JAR/WAR) into a minimal runtime image

This keeps the final image **small, secure, and optimized**.

---

## **Why / Purpose / Real-World Use Cases**

### **1. Reduce Final Image Size**

Builder stage contains:

* JDK
* Maven/Gradle
* Dependencies
* Source code

Runtime stage contains only:

* JRE
* Final JAR

Result: **Large → Small**

### **2. Reduce Attack Surface**

Runtime image does **not** contain:

* Build tools
* Source code
* Maven cache
* Shell utilities

### **3. Perfect for CI/CD Pipelines**

In Jenkins pipelines:

* Build app using Maven
* Push artifacts to Nexus
* Use only **JAR from Nexus** in final Docker image

### **4. Consistency for EKS Deployments**

Lightweight images → faster pull times → faster pod startup.

---

## **How It Works / Steps / Syntax**

### **Example: Java App Multi-Stage Dockerfile**

```dockerfile
# ====== STAGE 1: Build Stage ======
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /build
COPY pom.xml .
COPY mvnw .
COPY .mvn .mvn
RUN ./mvnw -q dependency:go-offline
COPY src src
RUN ./mvnw -q package -DskipTests

# ====== STAGE 2: Runtime Stage ======
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /build/target/myapp.jar /app/myapp.jar

ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

---

## **Real CI/CD Case (Jenkins + Nexus)**

### **When Jenkins Builds Artifact**

* Jenkins runs Maven build
* Generates JAR
* Uploads JAR to Nexus
* Dockerfile downloads JAR:

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

ADD https://nexus.mycompany.com/repository/releases/com/app/myapp.jar ./myapp.jar

ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

### **Final Image Size**

Only JRE + JAR → around **100–200 MB**, depending on JRE size.

This is much smaller than building inside Dockerfile, which might result in **600–800 MB** due to Maven, JDK, cache.

---

## **Common Issues / Errors**

### ❗ Wrong path in COPY --from

Incorrect paths cause "file not found".

### ❗ Maven not cached properly

Build becomes slow.
Fix: use dependency caching block.

### ❗ Large final image

Happens when:

* Runtime image uses full JDK instead of JRE
* Using Ubuntu instead of Alpine

### ❗ Nexus unreachable

Docker build fails when `ADD` or `curl`/`wget` cannot download artifact.

---

## **Troubleshooting / Fixes**

### Fix: Nexus download issues

Use `curl` or `wget` with proper credentials.

### Fix: Image too large

* Use JRE images
* Use Alpine runtime images
* Remove build dependencies from final stage

### Fix: Maven slow builds in builder stage

Use docker layer caching:

```dockerfile
COPY pom.xml .
COPY mvnw .
COPY .mvn .mvn
RUN ./mvnw -q dependency:go-offline
```

### Fix: Certificate issues with Nexus

Install CA certificates in Dockerfile runtime stage.

---

## **Best Practices / Tips**

### 1. If CI/CD builds the artifact

Use **single-stage runtime image** (no JDK, no Maven):

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY myapp.jar /app/
```

### 2. If Dockerfile builds the artifact

Use **multi-stage build**.

### 3. Final Image Goal

* Java runtime image: **100–200 MB**
* Avoid oversized images (600–800 MB)

### 4. Use Alpine or Distroless for Minimal Images

For enterprise:

* Distroless Java 17 → most secure
* Alpine JRE → lightest

### 5. Keep Builder Stage Separate

Never run build tools in the runtime stage.

---
---

# Minimal Base Images: Alpine, Slim, Distroless (Java Focus)

## **Concept / What**

Minimal base images are lightweight Docker images used to build small, secure, and fast application containers. The three common minimal image types:

### **1. Alpine**

* Very small (~5 MB)
* Has **apk** package manager
* BusyBox shell included
* Based on musl libc (not glibc)

### **2. Slim**

* Larger than Alpine but smaller than full images
* Based on Debian/Ubuntu
* Includes glibc
* Has shell and package manager (apt)

### **3. Distroless**

* Contains **only** application runtime (no shell, no package manager)
* Extremely small and secure
* Not meant for debugging inside the container

---

## **Why / Purpose / Real-World Use Cases**

### **Alpine**

* Used when image size must be very small
* Works well for static binaries, lightweight apps
* Common for microservices and cloud-native platforms

### **Slim**

* Most used in real-world enterprise environments
* Full glibc support → fully compatible with Java
* Easier to debug than Alpine
* Safer than Alpine for JVM-based workloads

### **Distroless**

* Highest security
* No shell → impossible for attackers to exploit inside
* Ideal for:

  * Highly secured production clusters
  * Environments with strict compliance
  * Final stable versions of microservices

---

## **How They Work (Java Examples)**

### **Alpine Example**

```dockerfile
FROM eclipse-temurin:17-jre-alpine
COPY myapp.jar .
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

### **Slim Example**

```dockerfile
FROM eclipse-temurin:17-jre-jammy
COPY myapp.jar .
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

### **Distroless Example**

```dockerfile
FROM gcr.io/distroless/java17
COPY myapp.jar /app/myapp.jar
ENTRYPOINT ["/app/myapp.jar"]
```

---

## **Common Issues / Errors**

### **Alpine Issues**

* Musl libc causes JVM compatibility problems
* Random crashes in Java apps
* Slower DNS resolution

### **Slim Issues**

* Slightly bigger image (~120–200 MB)
* But stable and reliable

### **Distroless Issues**

* No shell → cannot exec into container
* Debugging becomes difficult
* Requires logs + monitoring tools

---

## **Troubleshooting / Fixes**

### **For Alpine**

* Prefer glibc-based images for Java
* Use slim if you face crashes

### **For Distroless**

Debug using:

* Kubernetes logs:

```
kubectl logs pod
```

* Exec into ephemeral debugging container:

```
kubectl debug
```

* Sidecar for diagnostics

### **For Slim**

* Install minimal packages only if needed

---

## **Best Practices / Tips**

### **Most Used in Real World (Java)**

1. **Eclipse Temurin Slim/JRE** → most common
2. **OpenJDK Slim** → also widely used
3. **Distroless Java 17** → security-focused orgs
4. **Alpine** → rarely used for Java

### **Why Eclipse Temurin is preferred over OpenJDK**

* Maintained by Eclipse Foundation (Adoptium)
* Provides verified, TCK-compliant builds
* Stable for production
* Most Java enterprises shifting to Temurin

### **When to Use What**

* **Slim** → best balance (real-world preference)
* **Distroless** → maximum security
* **Alpine** → not recommended for Java apps

---
---

# Minimal Base Images: Alpine, Slim, Distroless (Java Focus)

## **Concept / What**

Minimal base images are lightweight Docker images used to build small, secure, and fast application containers. The three common minimal image types:

### **1. Alpine**

* Very small (~5 MB)
* Has **apk** package manager
* BusyBox shell included
* Based on musl libc (not glibc)

### **2. Slim**

* Larger than Alpine but smaller than full images
* Based on Debian/Ubuntu
* Includes glibc
* Has shell and package manager (apt)

### **3. Distroless**

* Contains **only** application runtime (no shell, no package manager)
* Extremely small and secure
* Not meant for debugging inside the container

---

## **Why / Purpose / Real-World Use Cases**

### **Alpine**

* Used when image size must be very small
* Works well for static binaries, lightweight apps
* Common for microservices and cloud-native platforms

### **Slim**

* Most used in real-world enterprise environments
* Full glibc support → fully compatible with Java
* Easier to debug than Alpine
* Safer than Alpine for JVM-based workloads

### **Distroless**

* Highest security
* No shell → impossible for attackers to exploit inside
* Ideal for:

  * Highly secured production clusters
  * Environments with strict compliance
  * Final stable versions of microservices

---

## **How They Work (Java Examples)**

### **Alpine Example**

```dockerfile
FROM eclipse-temurin:17-jre-alpine
COPY myapp.jar .
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

### **Slim Example**

```dockerfile
FROM eclipse-temurin:17-jre-jammy
COPY myapp.jar .
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

### **Distroless Example**

```dockerfile
FROM gcr.io/distroless/java17
COPY myapp.jar /app/myapp.jar
ENTRYPOINT ["/app/myapp.jar"]
```

---

## **Common Issues / Errors**

### **Alpine Issues**

* Musl libc causes JVM compatibility problems
* Random crashes in Java apps
* Slower DNS resolution

### **Slim Issues**

* Slightly bigger image (~120–200 MB)
* But stable and reliable

### **Distroless Issues**

* No shell → cannot exec into container
* Debugging becomes difficult
* Requires logs + monitoring tools

---

## **Troubleshooting / Fixes**

### **For Alpine**

* Prefer glibc-based images for Java
* Use slim if you face crashes

### **For Distroless**

Debug using:

* Kubernetes logs:

```
kubectl logs pod
```

* Exec into ephemeral debugging container:

```
kubectl debug
```

* Sidecar for diagnostics

### **For Slim**

* Install minimal packages only if needed

---

## **Best Practices / Tips**

### **Most Used in Real World (Java)**

1. **Eclipse Temurin Slim/JRE** → most common
2. **OpenJDK Slim** → also widely used
3. **Distroless Java 17** → security-focused orgs
4. **Alpine** → rarely used for Java

### **Why Eclipse Temurin is preferred over OpenJDK**

* Maintained by Eclipse Foundation (Adoptium)
* Provides verified, TCK-compliant builds
* Stable for production
* Most Java enterprises shifting to Temurin

### **When to Use What**

* **Slim** → best balance (real-world preference)
* **Distroless** → maximum security
* **Alpine** → not recommended for Java apps

---
---

# Dockerfile Instruction Execution: Layers vs Metadata (Java Example) — Detailed Explanation

## **Concept / What**

Dockerfile instructions are processed **top‑to‑bottom**, and each instruction either:

* **Creates a new image layer** using an **intermediate container**, or
* **Adds only metadata** (no filesystem changes, no container created).

A *layer* is created only when the instruction **modifies the filesystem**.
Metadata-only instructions update image configuration but don’t generate layers.

---

## **Why / Purpose / Real‑World Use Cases**

* **Build optimization:** Knowing which instructions create layers helps place frequently changing steps at the bottom for maximum caching.
* **Understanding build performance:** RUN and COPY instructions create intermediate containers → affect build time.
* **Troubleshooting:** Helps identify which step caused rebuilds or cache invalidation.
* **Image optimization:** Reduces unnecessary layers, improves caching, and decreases image size.
* **Predictable builds in CI/CD:** Ensures faster builds and fewer rebuilds for dependency steps.

---

## **How It Works / Steps / Syntax**

### **1. Instructions That Create Layers (Needs Intermediate Containers)**

These instructions modify the container filesystem → Docker must:

1. Create a temporary container
2. Execute the instruction inside it
3. Commit changed filesystem → new layer
4. Remove the container

#### **Layer‑Creating Instructions:**

* **FROM** – base layer
* **RUN** – executes commands
* **COPY** – copies files
* **ADD** – copies/extracts data
* **WORKDIR** – creates directory if missing
* **ENV** – persists environment variable
* **USER** – defines user context
* **VOLUME** – defines mount point

### **Java Example (Layer‑Creating Instructions)**

```dockerfile
FROM eclipse-temurin:17-jdk       # Layer 1
WORKDIR /app                      # Layer 2
COPY pom.xml .                    # Layer 3
RUN mvn -B dependency:resolve     # Layer 4 (expensive step)
COPY src ./src                    # Layer 5
RUN mvn -B package -DskipTests    # Layer 6
```

Each step creates an intermediate container → commits → moves to next layer.

---

### **2. Instructions That DO NOT Create Layers (Metadata Only)**

These instructions do NOT modify filesystem → no intermediate container is created.

#### **Metadata‑Only Instructions:**

* **CMD** – default container command
* **ENTRYPOINT** – main executable
* **HEALTHCHECK** – health probe definition
* **ARG** – build‑time variable (not saved in final image)
* **LABEL** – descriptive metadata
* **STOPSIGNAL** – shutdown signal
* **SHELL** – change RUN shell

### **Java Example (Metadata Instructions)**

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/target/app.jar"]
CMD ["--server.port=8080"]
HEALTHCHECK CMD curl -f http://localhost:8080/actuator/health || exit 1
```

These do **not** create layers.

---

## **Common Issues / Errors**

* **Slow builds:** Too many RUN steps or wrong order causes layer rebuilds.
* **Cache busting:** COPY placed before dependency installation forces full rebuild.
* **Large images:** Unoptimized layers accumulate unnecessary files.
* **Incorrect metadata:** Wrong ENTRYPOINT/CMD causes container startup failures.

---

## **Troubleshooting / Fixes**

* **Optimize Dockerfile ordering:** Dependencies first → source last.
* **Combine RUN commands:**

```dockerfile
RUN apt update && apt install -y curl unzip && apt clean
```

* **Use `.dockerignore`** to avoid copying unwanted files:

```
target
.git
logs
```

* **Use multi-stage builds** to keep only the final JAR.

### **Java Multi‑Stage Example (Optimized)**

```dockerfile
# ---- Build Stage ----
FROM eclipse-temurin:17-jdk AS build
WORKDIR /app
COPY pom.xml .
RUN mvn -B dependency:resolve
COPY src ./src
RUN mvn -B package -DskipTests

# ---- Runtime Stage ----
FROM eclipse-temurin:17-jre
COPY --from=build /app/target/app.jar /app/app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Only final runtime stage is shipped → minimal layers.

---

## **Best Practices / Tips**

* Place **frequently changing** steps (COPY source) at the **bottom**.
* Place **rarely changing** steps (dependencies) **above**.
* Use **multi-stage builds** for Java (Maven/Gradle).
* Combine RUN commands to reduce layers.
* Avoid unnecessary ADD instructions.
* Always use a `.dockerignore` file.
* Keep ENTRYPOINT/CMD metadata-only to avoid unnecessary layers.

---
---
---





