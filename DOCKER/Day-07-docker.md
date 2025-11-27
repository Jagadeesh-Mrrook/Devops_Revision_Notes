# Docker Compose: Multi-Container Application Setup (Detailed Notes)

## **1. Concept / What**

Docker Compose is a tool used to define and run **multiple containers together** using a `docker-compose.yml` file. It allows you to declare services (app, database, cache, etc.), their configuration, networks, and volumes so the entire application stack can be started with a single command.

---

## **2. Why / Purpose / Real-World Use Cases**

### **Local Development**

* Run full microservice applications on your laptop.
* Start backend + frontend + database together.
* Avoid running multiple long `docker run` commands.

### **CI/CD Pipelines**

* Spin up temporary environments (app + DB) for integration tests.
* Validate multi-service behavior before deploying to EKS.

### **Microservices Architecture**

* Simulate inter-service communication locally.
* Test service dependencies, environment variables, and networking.

### **EKS / Kubernetes Preparation**

* Compose is not used in production, but:

  * It simulates the multi-container environment locally.
  * Helps test interactions before moving to Kubernetes manifests.

---

## **3. How it Works / Steps / Syntax**

### **A. Defining Services in `docker-compose.yml`**

Example: App + MongoDB stack

```yaml
version: "3.9"

services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - db
    environment:
      - DB_URL=mongodb://db:27017/mydb

  db:
    image: mongo:6
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
```

### **B. Common Commands**

| Task                  | Command                         |
| --------------------- | ------------------------------- |
| Start all services    | `docker-compose up -d`          |
| Stop & remove         | `docker-compose down`           |
| Rebuild and start     | `docker-compose up -d --build`  |
| View running services | `docker-compose ps`             |
| View logs             | `docker-compose logs <service>` |

### **C. Networking in Compose**

* Compose automatically creates a **user-defined bridge network**.
* All services join this network.
* Services resolve each other by **service name** (e.g., `db`).

**Example:** App connects to database using:

```
mongodb://db:27017/mydb
```

### **D. Volumes**

Used to persist data outside containers.

Example:

```
volumes:
  mongo-data:
```

---

## **4. Common Issues / Errors**

### **1. App starts before DB is ready**

Error: "Cannot connect to MongoDB".

* Cause: `depends_on` only controls order, not readiness.
* Fix: Add retry logic or healthcheck.

---

### **2. Port Conflicts**

If host port is already in use:

```
bind: address already in use
```

Fix: Change host port under `ports:`.

---

### **3. Volume Permission Issues**

Mongo may fail if the local directory doesn’t have proper permissions.
Fix:

```
sudo chown -R 999:999 ./mongo-data
```

---

### **4. Incorrect Build Context**

Dockerfile missing due to wrong path.
Fix: Ensure build context (`.`) is correct.

---

## **5. Troubleshooting / Fixes**

| Problem                 | Fix                                |
| ----------------------- | ---------------------------------- |
| DB not ready            | Add healthchecks or retries        |
| Networking issues       | Use service name, not container IP |
| Build errors            | Verify Dockerfile & context path   |
| Missing env vars        | Use `environment:` or `.env` file  |
| Volume ownership errors | Adjust user IDs/permissions        |

---

## **6. Best Practices / Tips**

### **Compose File Best Practices**

* Use version `3.9`.
* Use service names for networking.
* Use **named volumes** (good for DB).
* Keep secrets in `.env`, never hardcode.

### **Dockerfile Best Practices**

* Use multi-stage builds to reduce image size.
* Use non-root user for security.
* Keep image lightweight.

### **CI/CD Best Practices**

* Use Compose for local + CI integration tests.
* Validate multi-container setup before pushing to ECR.

### **EKS Readiness**

* Use Compose locally to simulate services.
* Convert Compose setup to Kubernetes manifests when deploying.

---
---

# Docker Compose: Services, Networks, and Volumes (Detailed Notes)

## **1. Concept / What**

Docker Compose is used to define and manage **multi-container applications** using a YAML file (`docker-compose.yml`). The three core components are:

### **Services**

Containers you want to run (app, db, redis, nginx).

### **Networks**

Enable communication between services using **service-name-based DNS**.

### **Volumes**

Provide persistent storage for databases, logs, and application data.

These three components together form the structure of any Compose setup.

---

## **2. Why / Purpose / Real-World Use Cases**

### **A. Local Development**

* Run multiple services with one command.
* Consistent environment for all developers.
* Eliminates multiple `docker run` commands.

### **B. CI/CD Automation**

* Spin up databases, message queues, and APIs during integration tests.
* Lightweight alternative to Kubernetes for pipeline testing.

### **C. Microservices Simulation Before Kubernetes**

* Test service connectivity.
* Validate ports, env variables, volumes.
* Debug issues locally before deploying to EKS.

### **D. Persistent Data Handling**

* Databases require persistent volumes.
* Allows safe restarts without losing data.

---

## **3. How It Works / Steps / Syntax**

### ⭐ **A. Services**

Defines each container's configuration.

```yaml
services:
  app:
    build: ./app
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
    depends_on:
      - db
    networks:
      - backend

  db:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=admin123
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - backend
```

**Key Points:**

* `depends_on` = start order (not readiness).
* Services use DNS names to communicate (`db`).
* Ports, volumes, env vars, and build instructions are defined here.

---

### ⭐ **B. Networks**

Compose auto-creates a **user-defined bridge network**.

You can define custom networks:

```yaml
networks:
  backend:
  frontend:
```

Attach services:

```yaml
services:
  app:
    networks:
      - backend
      - frontend
  db:
    networks:
      - backend
```

**Why Networks Matter:**

* Enable controlled communication.
* Isolate internal services from public-facing ones.
* Mimic Kubernetes-like segmentation.

---

### ⭐ **C. Volumes**

Provide persistent storage.

Declare volumes:

```yaml
volumes:
  db_data:
```

Use in services:

```yaml
services:
  db:
    volumes:
      - db_data:/var/lib/mysql
```

**Real Use Cases:**

* Persist database data (MySQL/Postgres/Mongo).
* Store logs or uploads.
* Speed up builds by caching data.

---

## **4. Common Issues / Errors**

### **1. Services Cannot Communicate**

Cause: Not on the same network.
Fix: Attach both services to a shared network.

### **2. DB Losing Data**

Cause: Missing named volume.
Fix: Always use named volumes in `volumes:` block.

### **3. App Fails to Connect to DB**

Cause: Using `localhost` instead of service name.
Fix: Use `db` as hostname.

### **4. `depends_on` Misunderstood**

Cause: App starts before DB is ready.
Fix: Use healthcheck or retry logic.

### **5. Port Bind Conflicts**

Cause: Same port used by another service.
Fix: Change host port (`8081:8080`).

---

## **5. Troubleshooting / Fixes**

| Issue                       | Fix                                               |
| --------------------------- | ------------------------------------------------- |
| Build failing               | Validate `context:` and Dockerfile path           |
| Permission denied on volume | Adjust directory ownership or UID                 |
| Missing env vars            | Use `.env` or `environment:` properly             |
| App can't reach DB          | Ensure they're on same network & use service name |
| Data wiped on restart       | Use named volumes, not bind mounts                |

---

## **6. Best Practices / Tips**

### **Services**

* Avoid `container_name` (breaks scaling).
* Use healthchecks for DB readiness.
* Keep services small and focused.

### **Networks**

* Use multiple networks for isolation.
* Use clear naming (`frontend`, `backend`).

### **Volumes**

* Use named volumes for DB.
* Avoid bind mounts for databases.

### **General Compose Best Practices**

* Store configuration in `.env` files.
* Use multi-stage Dockerfiles for lightweight images.
* Compose is for local dev & CI/CD, not production.
* Structure project cleanly:

```
project/
 ├── app/
 ├── nginx/
 ├── docker-compose.yml
 └── .env
```

---
---

# Docker Compose: Environment Variables (Detailed Notes)

## **1. Concept / What**

Environment variables in Docker Compose allow configuration values to be injected into containers at runtime. These include database credentials, API keys, URLs, ports, secrets, and application-specific settings. They can be provided using:

* `environment:` block inside Compose
* `.env` files (most common)
* `env_file:` reference
* CI/CD pipeline variables
* Secret managers (in real staging/production)

Environment variables help keep the app configurable without modifying code or rebuilding images.

---

## **2. Why / Purpose / Real-World Use Cases**

### **A. Application Configuration Flexibility**

Developers specify what variables the app needs (e.g., DB_HOST, JWT_SECRET). DevOps injects them via Compose.

### **B. Secure Handling of Sensitive Data**

Credentials stay outside Docker images and Git repositories.

### **C. Consistent Configuration Across Environments**

Allows smooth flow from:

* Local → Dev → Stage → Production

### **D. Simulating Production Behavior Locally**

Compose lets developers run services with realistic variables.

### **E. CI/CD Pipelines**

Pipelines inject temporary secrets during integration tests.

### **F. Kubernetes Preparation**

Environment variable structure maps directly to:

* ConfigMaps (non-secret config)
* Secrets (sensitive values)

---

## **3. How It Works / Steps / Syntax**

### ⭐ **A. Environment Variables in `environment:` Block**

Key-value map format:

```yaml
environment:
  DB_HOST: db
  DB_USER: root
  DB_PASS: ${DB_PASS}
```

This injects variables into the container.

---

### ⭐ **B. Using `.env` File (Most Common in Real Companies)**

`.env` file (NOT committed to Git):

```
DB_USER=root
DB_PASS=localpassword
JWT_SECRET=devkey
```

Compose file:

```yaml
services:
  app:
    env_file:
      - .env
```

**Benefits:**

* Clean and centralized
* Easy switching of environments
* Prevents leaking secrets into Git

---

### ⭐ **C. `.env.example` File (Created by DevOps)**

Contains ONLY variable names:

```
DB_USER=
DB_PASS=
JWT_SECRET=
```

Developers copy it and create a real `.env` for their machine.

---

### ⭐ **D. Shell `export` Method (Local Use Only)**

```bash
export DB_PASS="mypassword"
```

Compose picks it automatically:

```yaml
environment:
  DB_PASS: ${DB_PASS}
```

Used only for local/dev — not for production.

---

### ⭐ **E. Using CI/CD Environment Variables**

Example (GitHub Actions):

```yaml
env:
  DB_PASS: ${{ secrets.DB_PASS }}
```

Compose will receive this value via `${DB_PASS}`.

---

### ⭐ **F. Build-Time Variables (`build.args`)**

```yaml
build:
  context: .
  args:
    APP_ENV: production
```

Used for building different images for dev/stage/prod.

---

## **4. Common Issues / Errors**

### **1. `.env` File Not Loaded**

Cause: wrong filename or wrong path.
Fix: Must be named `.env` and in same folder as Compose.

### **2. Hardcoding Secrets in Compose**

```yaml
DB_PASS: admin123
```

**Never do this.** Commit history exposes secrets.

### **3. Using Localhost Instead of Service Name**

`localhost` won't work between containers.
Fix: Use service name:

```
DB_HOST=db
```

### **4. `depends_on` Misunderstood**

It doesn’t wait for DB readiness.
Fix: Use retry logic or healthchecks.

### **5. CI/CD Overrides**

Global pipeline variables override local `.env`.
Fix: Use unique variable names (`APP_DB_USER` instead of `DB_USER`).

---

## **5. Troubleshooting / Fixes**

| Issue              | Resolution                               |
| ------------------ | ---------------------------------------- |
| Env var missing    | Check YAML indentation or spelling       |
| `.env` ignored     | Ensure correct filename & location       |
| DB auth failure    | Confirm variables are passed to DB image |
| Wrong value        | Check shell overrides (`export`)         |
| App can't reach DB | Use service name, not localhost          |

---

## **6. Best Practices / Tips**

### **Security**

* Never commit `.env` with values.
* Always commit `.env.example` with placeholders.
* Use AWS SSM/Secrets Manager or Vault for stage/prod.

### **Structure & Management**

* Developers fill `.env` with values.
* DevOps defines `.env.example` and Compose configuration.
* Use uppercase variable names.
* Maintain consistent variable names across environments.

### **CI/CD & Prod**

* Secrets come from pipeline secret stores.
* Kubernetes Secrets replace `.env` in production.

### **EKS Readiness**

Compose env variables map directly to:

* Kubernetes Secrets (sensitive values)
* ConfigMaps (non-sensitive values)

This makes migration smooth.

---
---

# Docker Compose: Scaling Services (Detailed Notes)

## **1. Concept / What**

Scaling in Docker Compose allows you to run **multiple instances (replicas)** of the same service using a single command:

```
docker compose up --scale <service>=<count>
```

This is used only for **stateless** services (API servers, web apps, workers). Databases cannot be scaled using Docker Compose.

To support scaled replicas, a **reverse proxy** like **Nginx** or **Traefik** is often used to load balance requests across multiple containers.

---

## **2. Why / Purpose / Real-World Use Cases**

### **A. Testing Load Balancing Locally**

Developers and DevOps can simulate production-like traffic distribution.

### **B. Microservices Local Simulation**

Test how multiple backend replicas behave under concurrency.

### **C. CI/CD Integration Tests**

Run multiple worker containers to process events faster.

### **D. Pre-Kubernetes Simulation**

Mimic Kubernetes `replicas:` behavior before deploying to EKS.

### **E. Testing Stateless Architecture**

Validates service design for horizontal scaling.

---

## **3. How It Works / Steps / Syntax**

### ⭐ **A. Basic Scaling Command**

```
docker compose up -d --scale app=3
```

This launches:

* app-1
* app-2
* app-3

### ⭐ **B. Compose Example for Scalable App**

```yaml
version: "3.9"

services:
  app:
    build: ./app
    networks:
      - backend
    # No ports here (important!)

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - backend
    depends_on:
      - app

networks:
  backend:
```

The `app` service is designed for scaling, and the `nginx` service acts as a load balancer.

---

### ⭐ **C. Why You Must Remove Ports from Scaled Services**

Only **one** container can bind to a host port:

```
8080:8080
```

If you scale to 3 replicas, 2 will fail.

Solution:

* Remove port mapping from the app service.
* Use reverse proxy (Nginx/Traefik) to expose a single port.

---

### ⭐ **D. Reverse Proxy (Nginx) Load Balancing**

Nginx listens on host port **80** and forwards traffic to multiple app replicas.

**nginx.conf:**

```nginx
events {}

http {
    upstream app_cluster {
        zone app_cluster 64k;
        resolver 127.0.0.11 valid=10s;
        server app:8080 resolve;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://app_cluster;
        }
    }
}
```

**Key points:**

* `resolver 127.0.0.11` is Docker's internal DNS.
* `server app:8080 resolve;` dynamically maps **all replicas**.
* No need to manually list app-1, app-2, app-3.

This is how real companies load balance scaled Compose services.

---

## **4. Common Issues / Errors**

### **1. Port Conflicts**

Error:

```
bind: address already in use
```

Cause: multiple containers mapped to the same port.
Fix: remove `ports:` from scalable services.

---

### **2. Scaling Stateful Services Fails**

Example: scaling MySQL/Postgres.

Cause:

* Databases need persistent volumes.
* Compose cannot automatically manage multiple volume sets.

Fix:

* Run only 1 replica.
* Stateless services only.

---

### **3. `container_name` Prevents Scaling**

If you use:

```
container_name: app
```

Scaling fails because names must be unique.
Fix:

* Do NOT use `container_name` for scalable services.

---

### **4. Shared Volumes Across Replicas**

Multiple replicas writing to the same volume causes corruption.
Fix:

* Stateless services only.
* No shared writable volumes.

---

## **5. Troubleshooting / Fixes**

| Issue               | Fix                                                  |
| ------------------- | ---------------------------------------------------- |
| Port already in use | Remove ports in scaled service and use reverse proxy |
| Scaled DB crashes   | Keep DB as single instance                           |
| Nginx not routing   | Check upstream names and Docker DNS                  |
| App unreachable     | Ensure Nginx is connected to same network            |
| Scaling ignored     | Use correct command and remove container_name        |

---

## **6. Best Practices / Tips**

### ✔ Stateless Only

Scale **only**:

* API servers
* Web servers
* Worker jobs
* Consumers

### ✔ Use Nginx or Traefik for Load Balancing

Expose only Nginx:

```
ports:
  - "80:80"
```

Let it balance traffic to multiple replicas.

### ✔ No Static Port Mapping in Scalable Services

Do not use:

```
ports:
  - "8080:8080"
```

### ✔ No `container_name`

Allow Compose to auto-generate names like `app-1`, `app-2`.

### ✔ Use Reverse Proxy configs with DNS-based load balancing

Ensures compatibility with dynamic scaling.

### ✔ Use Scaling for Local Dev & CI — Not Production

In production, scaling is handled by Kubernetes Deployments.

---
---

# Docker Compose: Override Files (Detailed Notes)

## **1. Concept / What**

Override files in Docker Compose are additional YAML files that *modify or extend* the base `docker-compose.yml`. They allow you to apply environment-specific changes—like local development settings, debugging configurations, or CI/testing overrides—without touching the main file.

Docker Compose automatically loads:

* `docker-compose.yml`
* `docker-compose.override.yml`

If you use a custom override file name, you must specify it manually.

Override files merge into the base file, replacing or extending only the parts you specify.

---

## **2. Why / Purpose / Real-World Use Cases**

### **A. Production-like Base Configuration**

The main Compose file should stay **clean**, stable, and close to real production behavior.

### **B. Environment-Specific Customizations**

Override files allow changes for:

* local development
* debugging
* CI integration tests
* temporary developer-specific behaviors

### **C. Avoid Multiple Full Compose Files**

Instead of maintaining separate full Compose files for each environment, you keep:

* one main (production-like)
* multiple small override files

### **D. Cleaner Git Repositories**

Developers do not edit the main file. They only extend it.

### **E. Flexible CI/CD Pipelines**

Testing overrides allow running mocks, alternate commands, or test databases.

---

## **3. How It Works / Steps / Syntax**

### ⭐ **A. Base File: `docker-compose.yml` (Production-like)**

```yaml
version: "3.9"

services:
  app:
    image: myapp:latest
    ports:
      - "8080:8080"
    environment:
      - ENV=production
```

This file remains stable and clean.

---

### ⭐ **B. Override File: `docker-compose.override.yml` (Dev)**

```yaml
version: "3.9"

services:
  app:
    build: .
    volumes:
      - ./src:/app/src
    environment:
      - ENV=development
    ports:
      - "3000:8080"
```

Only differences from the main file need to be listed.

---

### ⭐ **C. Automatic vs Manual Override Loading**

**Auto-loaded:**

```
docker-compose.yml
docker-compose.override.yml
```

**Manual:**

```
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

Order matters: the last file overrides earlier ones.

---

### ⭐ **D. Merging Rules (Important)**

* New values override old ones.
* Only fields provided in the override file replace existing ones.
* Unspecified fields remain unchanged.

Example:
Base:

```yaml
ports:
  - "8080:8080"
```

Override:

```yaml
ports:
  - "3000:8080"
```

Final:

```
3000:8080
```

---

## **4. Common Issues / Errors**

### **1. Override Not Applied**

Cause: wrong filename or missing `-f`.
Fix: Use `docker-compose.override.yml` or specify manually.

### **2. Unexpected Merged Configs**

Cause: wrong order.
Fix: Base comes first, override last.

### **3. Too Many Overrides**

Cause: clutter.
Fix: Follow naming standards: dev/test/debug.

### **4. Mistaking Compose Files for Production Files**

Cause: misunderstanding.
Fix: Compose is ONLY for local dev + testing, NOT production.

---

## **5. Troubleshooting / Fixes**

| Issue                         | Fix                                       |
| ----------------------------- | ----------------------------------------- |
| Ports not updated             | Override must specify ports explicitly    |
| Dev config bleeding into base | Keep base clean, no dev-specific settings |
| Overrides ignored             | Check file names, use correct order       |
| Hard to see final config      | Run `docker compose config`               |

---

## **6. Best Practices / Tips**

### ✔ Base File: Production-Like

* No bind mounts
* No debug tools
* No dev-only env vars
* Close to Kubernetes config

### ✔ Override Files for Dev/Test

* Add bind mounts
* Change ports
* Add debug commands
* Use local builds
* Add testing commands or mocks

### ✔ Naming Standards

* `docker-compose.yml` (base)
* `docker-compose.override.yml` (local dev)
* `docker-compose.dev.yml` (custom dev)
* `docker-compose.test.yml` (CI)
* `docker-compose.debug.yml` (debug)

### ✔ Use `.env` for Values

Store sensitive or environment-specific values in `.env`, not YAML.

### ✔ Validate Final Output

Use:

```
docker compose config
```

This shows the fully merged configuration.

---
---

# Docker Compose in Real CI/CD & Local Development Workflow (Rewritten Detailed Notes)

## **1. Concept / What**

In real DevOps organizations, Docker Compose is used **only for local development and local multi-service testing**, never for production deployments. The main purpose is to allow developers to run complete microservices on their laptops using a single command:

```
docker compose up
```

This local environment uses the **same Docker images that Jenkins builds**, ensuring consistency across all environments (local → dev → QA → stage → prod). The goal is to provide developers a stable, production-like simulation using standard containers rather than building their own images locally.

Docker Compose serves as the **local execution layer**, while Jenkins + ECR + Kubernetes/EKS serve as the **official CI/CD and deployment layer**.

---

## **2. Why / Purpose / Real-World Use Cases**

### **A. Avoid Local Image Building**

Developers do not build Docker images locally. Local image building creates duplication, inconsistency, and version drift. Instead, all official Docker images are built through Jenkins pipelines only.

### **B. Ensure Consistency Across Environments**

By using the exact same Docker image for:

* local developer Compose tests,
* dev environment deployments,
* QA/stage deployments,
* production deployments,

you eliminate all "works on my machine" issues.

### **C. Use Compose for Local Microservices Simulation**

Compose provides developers with a way to:

* run all dependent microservices together,
* simulate production-like networking,
* test interactions between services,
* validate behavior using Jenkins-built images.

### **D. Keep CI/CD Pipeline Clean**

Jenkins is responsible for:

* checking out source code,
* running unit tests,
* running static code analysis (SonarQube),
* performing security scans,
* building Docker images,
* pushing images to ECR,
* deploying to EKS.

Compose is not used in CI/CD. It stays strictly on developer machines.

### **E. Faster and Cleaner Developer Workflow**

Developers focus only on:

* writing code,
* running unit tests locally,
* fixing issues,
* raising PRs.

All heavy tasks (image building, scanning, deployment) happen automatically in Jenkins.

---

## **3. How It Works / Steps / Syntax**

### **A. Developer Workflow Before PR (Feature Branch)**

On feature branches, developers:

* write code,
* run unit tests locally,
* validate logic.

They do **not** build Docker images at this stage.
Developers typically do not run full microservices locally using Compose before the PR because the image is not yet built by Jenkins.

### **B. Jenkins Workflow After PR Merge (Dev Branch)**

When a PR is merged into the dev branch, Jenkins performs:

1. Checkout of code.
2. Unit test execution.
3. Static code analysis (SonarQube).
4. Security scanning.
5. Official Docker image build.
6. Tagging of the image (dev/stage/prod/commit-id).
7. Pushing the image to AWS ECR.
8. Deployment of the application to EKS or lower environment based on parameters.

This Jenkins-built image is the **official and only valid version** of the application.

### **C. Developer Workflow After PR (Using Docker Compose)**

Once Jenkins successfully builds the image and pushes it to ECR, developers can:

* log in to ECR using AWS CLI,
* pull the Jenkins-built image,
* run all microservices locally through Docker Compose.

Example Compose file:

```yaml
services:
  app:
    image: <aws_account>.dkr.ecr.<region>.amazonaws.com/app:dev
    env_file:
      - .env
  db:
    image: postgres:15
```

This ensures local tests use the same image used in dev/stage/prod environments.

### **D. No Double Work**

Because developers don’t build Docker images locally, there is no duplication of effort. All image creation is centralized within Jenkins, which ensures quality, consistency, and proper validation.

---

## **4. Common Issues / Errors**

### **1. Developers Still Build Images Locally**

This leads to inconsistencies and invalid versions of the application.
**Fix:** Always use Jenkins-built images.

### **2. Jenkins Tagging Is Inconsistent**

If images are not tagged properly, local testing becomes unreliable.
**Fix:** Use a consistent tagging strategy (`latest`, `dev`, `stage`, `prod`, `commit-id`).

### **3. Environment Mismatch in Local Testing**

Occurs when developers use local builds instead of ECR images.
**Fix:** Pull the official Jenkins-built image before using Compose.

### **4. SonarQube Failures After PR**

Expected since developers do not run SonarQube locally.
**Fix:** Developer fixes issues reported by Jenkins and pushes changes again.

---

## **5. Troubleshooting / Fixes**

| Problem                               | Root Cause                  | Fix                                    |
| ------------------------------------- | --------------------------- | -------------------------------------- |
| Behavior mismatch between local & dev | Local builds used           | Always use ECR images                  |
| DB or service connection failures     | Wrong host names            | Use service names like `db` in Compose |
| Authentication errors                 | Missing ECR login           | Use AWS CLI to authenticate            |
| Compose running old images            | Not pulling updated version | Run `docker compose pull` before `up`  |

---

## **6. Best Practices / Tips**

### ✔ Developers:

* Do not build Docker images locally.
* Use Jenkins-built images for Compose-based local microservices testing.
* Only focus on writing and testing code.

### ✔ Jenkins:

* Is the single source of truth for building Docker images.
* Handles all analysis, testing, scanning, and deployment.
* Pushes fully validated images to ECR.

### ✔ Docker Compose Usage:

* Should use official images from ECR.
* Should be used for local microservice simulation after PR merges.
* Should not be used to override CI/CD logic.

### ✔ Consistency Across Environments:

The same Jenkins-built Docker image must run in:

* local Docker Compose,
* dev environment,
* QA/stage,
* production.

This ensures full stability and environment parity.

---
---

# Why Docker Compose Is NOT Used in Production (Detailed Notes)

## **1. Concept / What**

Docker Compose is a tool designed for **local development, testing, and small-scale environments**, not for production deployment. It manages multiple containers on **a single machine**, with simple networking, volumes, and environment variable handling.

Production environments require:

* scalability,
* high availability,
* self-healing,
* load balancing,
* multi-node orchestration,
* rolling deployments,
* advanced scheduling.

Docker Compose does **not** provide these capabilities.

Production-grade workloads are run using:

* **Kubernetes (EKS)**,
* **ECS**, or
* **Docker Swarm (older/rare)**.

---

## **2. Why / Purpose / Real-World Reasons**

Organizations do not use Docker Compose in production due to the following limitations:

### **A. Single-Node Limitation**

Compose works on **one machine only**, meaning:

* no cluster,
* no multiple worker nodes,
* no node pools,
* no distribution of workloads.

This makes it unsuitable for any application requiring scalability or high availability.

### **B. No Auto-Scaling**

Compose can scale services locally, but it has **no Horizontal Pod Autoscaler (HPA)** like Kubernetes.

It cannot scale:

* based on CPU,
* based on memory,
* based on traffic,
* based on load.

### **C. No Self-Healing**

Compose does not:

* restart crashed containers automatically in a robust way,
* replace unhealthy instances,
* perform liveness or readiness checks.

Kubernetes continuously monitors containers and self-heals.

### **D. No Rolling Deployments / Zero Downtime Updates**

Compose cannot:

* roll out new versions gradually,
* maintain old versions during updates,
* provide rollback if something fails.

Kubernetes has rolling updates and strategy controls.

### **E. No Service Discovery Across Nodes**

Compose networking is limited to a single host. It cannot:

* create cluster-wide DNS,
* provide cross-node service communication,
* integrate with external load balancers.

### **F. No High Availability (HA)**

If the host running Compose fails:

* entire application goes down,
* no failover,
* no redundancy.

### **G. No Security Features Required for Production**

Compose lacks:

* network policies,
* RBAC,
* admission controllers,
* secrets integration with AWS/KMS,
* IAM roles for containers.

Kubernetes/EKS provides strong production-grade security.

### **H. No Observability / Monitoring**

Compose has no built-in:

* Prometheus integration,
* metrics server,
* logging system,
* distributed tracing.

Production requires deep observability.

### **I. No Scheduling / Resource Management**

Compose cannot:

* schedule containers to the right nodes,
* enforce requests/limits,
* avoid resource starvation.

Kubernetes has a full scheduler.

### **J. No Ingress / Load Balancing**

Compose cannot attach to:

* AWS Load Balancers,
* NGINX Ingress Controllers,
* Service Mesh (Istio).

Production workloads require these features.

---

## **3. How Production Works Instead**

Organizations use Docker Compose only in these phases:

* individual developer laptops,
* local microservice testing,
* small QA checks.

For real production environments, they use:

* **Kubernetes / EKS** for microservices,
* **ECS** for simpler workloads,
* **Terraform** to provision the infrastructure,
* **Jenkins/GitHub Actions** for CI/CD pipelines,
* **ECR** for storing Docker images.

---

## **4. Common Issues / Errors When Using Compose Beyond Dev**

| Issue                 | Description                             |
| --------------------- | --------------------------------------- |
| No HA                 | If the machine dies, everything stops   |
| No scaling            | Cannot scale horizontally automatically |
| No secrets mgmt       | Cannot securely manage credentials      |
| No node orchestration | All containers are stuck on one host    |
| No update strategies  | Application downtime during deployments |
| Limited networking    | Cannot integrate with cloud networking  |

---

## **5. Troubleshooting / Fixes**

If teams attempt to run Compose in production and face issues, the real fix is:

* Move to **Kubernetes (EKS)** or **ECS**.
* Use Helm charts or raw manifests.
* Use load balancers and autoscaling groups.
* Use proper secret management.
* Use monitoring & logging stacks.

---

## **6. Best Practices / Tips**

### ✔ Use Docker Compose only for:

* developer local environments,
* local service simulation,
* testing containers,
* QA/laptop-based integration testing.

### ✔ For production:

* Use Kubernetes (EKS) for orchestration,
* Use ECR for storing images,
* Use Jenkins/GitHub Actions for CI/CD,
* Use Terraform for infrastructure provisioning,
* Use AWS Secrets Manager for secrets.

### ✔ Maintain separate configs:

* Compose for local dev,
* Kubernetes manifests or Helm charts for deployment.

### ✔ Follow cloud-native security practices:

* use IAM roles for service accounts (IRSA),
* use KMS for encryption,
* use RBAC, network policies, PSPs,
* use managed node groups or Fargate.

---
---


