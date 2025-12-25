# Docker Networking – Bridge Network (Detailed Explanation Notes)

## **1. Concept / What**

The **Bridge Network** is Docker’s default network mode for containers. When a container is started without specifying a network, Docker attaches it to the default `bridge` network. A bridge network connects containers on the same Docker host using virtual Ethernet interfaces (veth pairs) and a virtual switch (`docker0`).

* Containers receive an internal IP via their **eth0** interface (veth).
* Default bridge supports **IP-based communication only**.
* **User-defined bridge** networks provide name-based DNS and better isolation.

---

## **2. Why / Purpose / Real-World Use Cases**

Bridge networks are used when containers need to communicate **within the same host**. Common scenarios:

### **DevOps / CI/CD Use Cases**

* Running local microservices: backend ↔ database ↔ frontend.
* Jenkins running builds while interacting with other service containers.
* Local environment simulation before pushing images to ECR/EKS.
* Running databases (MySQL, MongoDB) locally for application testing.

### **Production-Like Local Testing**

* Testing port mappings before deploying behind ALB on EKS.
* Verifying container-network isolation and communication.

---

## **3. How it Works / Steps / Syntax**

### **List existing networks**

```bash
docker network ls
```

### **Inspect the default bridge network**

```bash
docker network inspect bridge
```

### **Run container on default bridge (automatic)**

```bash
docker run -d --name web nginx
```

### **Create a user-defined bridge network** (recommended)

```bash
docker network create app-net
```

### **Run containers on the custom network**

```bash
docker run -d --name db --network app-net mysql:8

docker run -d --name api --network app-net my-api-image
```

### **Container-to-container communication (user-defined bridge)**

Inside `api` container:

```bash
ping db        # Works because DNS is available
```

### **Port Mapping (Host → Container)**

```bash
docker run -p 8080:80 nginx
```

Accessed via:

```
http://HOST-IP:8080
```

---

## **4. Common Issues / Errors**

### **1️⃣ Containers cannot reach each other**

**Cause:** They are on different networks.
**Fix:** Attach both to same user-defined bridge.

---

### **2️⃣ Host cannot access the container**

**Cause:** Port not exposed with `-p`.
**Fix:**

```bash
docker run -p 8080:80 app
```

---

### **3️⃣ Container name ping fails**

**Cause:** Using default `bridge` network.
**Fix:** Use user-defined bridge.

---

### **4️⃣ Container IP changes on restart**

**Fix:** Always use container names, not IPs.

---

### **5️⃣ Port conflicts**

**Cause:** Two containers mapping same host port.
**Fix:** Use different host ports:

```bash
-p 8080:80
-p 8081:80
```

---

## **5. Troubleshooting / Fixes**

### **Check container IP**

```bash
docker inspect container-name | grep IPAddress
```

### **Test network connectivity**

```bash
docker exec -it api ping db
```

### **Check network details**

```bash
docker network inspect app-net
```

### **List network interfaces inside container**

```bash
docker exec -it web ip a
```

You’ll see `eth0` — the container's veth NIC.

---

## **6. Best Practices / Tips**

### **Network Design**

* Prefer **user-defined bridge networks** over default.
* Use **one network per application** for isolation.
* Avoid using IPs; rely on **container names**.

### **Security**

* Do **not** map ports unless required.
* Run applications as a **non-root user** inside the container.
* No direct access from container → host unless necessary.

### **Performance & Production Readiness**

* Keep container count per network reasonable.
* Use multi-stage builds for smaller images before deploying to EKS.
* Verify all ports and DNS work locally before CI/CD deployment.

### **EKS-Specific Tip**

Bridge networks are **local only**. In Kubernetes/EKS, networking is handled by CNI plugins (not Docker bridge). But understanding bridge networking helps troubleshoot:

* local builds
* local Compose setups
* container-to-container connectivity issues before pushing images

---
---

# Docker Networking – Host Network (Detailed Explanation Notes)

## **1. Concept / What**

The **Host Network** mode makes a container share the host machine’s network stack. The container does **not** get its own virtual network interface (no veth, no bridge, no docker0).

* The container uses **host’s IP** directly.
* The container binds directly to **host ports**.
* Port mapping using `-p` is **ignored**.
* No NAT, no bridge, no isolation.

---

## **2. Why / Purpose / Real-World Use Cases**

Host network mode is used when containers need **direct access** to the host's network with minimal overhead.

### **DevOps / CI/CD Use Cases**

* **Monitoring agents** (Node Exporter, Grafana Agent, Promtail, Filebeat).
* **Logging agents** collecting logs from host filesystem and listening on host ports.
* **DNS proxy / Local DNS resolvers**.
* **Metric scrapers** that must bind to `localhost`.

### **Performance Use Cases**

* Apps requiring extremely **low network latency**.
* High-throughput networking tools.

### **Not for Application Exposure**

Production applications (web apps, APIs, microservices) should **not** use host network.
They should use **bridge/custom bridge + port mapping** instead.

---

## **3. How it Works / Steps / Syntax**

### **Run a container with host network**

```bash
docker run --network host -d nginx
```

### What happens:

* Nginx listens directly on **host's port 80**.
* Access via:

```
http://HOST-IP:80
```

* No need for `-p` (ignored).

### **Inspect container networking**

```bash
docker inspect <container> --format '{{json .NetworkSettings}}'
```

You will see:

* No bridge
* No container IP (like 172.x.x.x)
* Uses host network

---

## **4. Common Issues / Errors**

### **1️⃣ Port Conflicts**

Since container uses host ports directly:

```
bind: address already in use
```

**Fix:** Stop conflicting host service or change app port.

---

### **2️⃣ Security Risks**

Host network exposes services more widely.

**Fix:** Bind applications to `127.0.0.1` inside container.

---

### **3️⃣ No Network Isolation**

Container can access all host network interfaces.

**Fix:** Avoid host mode unless required.

---

### **4️⃣ Unsupported on Mac/Windows**

Docker Desktop emulation makes host network unreliable.

**Fix:** Use Linux for host networking.

---

## **5. Troubleshooting / Fixes**

### **Check if a port is already in use**

```bash
sudo lsof -i :80
```

### **Check active listeners**

```bash
ss -tulnp | grep <port>
```

### **Check container process port usage**

```bash
docker exec -it <container> netstat -tulnp
```

### **Check host network details**

Since container shares host network:

* Same network tools used for host apply to container.

---

## **6. Best Practices / Tips**

### ✔ When to Use

* Logging agents (Filebeat, Promtail)
* Metrics collectors (Node Exporter)
* Tools that require access to host ports
* Network utilities where low latency matters

### ✖ When NOT to Use

* Web apps / APIs meant for public or internal consumption
* Microservices communicating via containers
* Anything requiring strong isolation

### ✔ Recommended Alternative for Apps

Use **custom bridge network + port mapping**:

```bash
-p hostPort:containerPort
```

This is the safest and most common method to expose apps externally.

### ✔ Security Considerations

* Avoid exposing privileged host ports
* Ensure firewall rules are applied
* Bind sensitive apps to localhost when using host networking

### ✔ EKS Note

Host networking is not used in Kubernetes pods (except very rare DaemonSet cases).
Understanding this helps troubleshoot local Docker setups but is not typical for container orchestration.

---
---

# Docker Networking – None Network (Detailed Explanation Notes)

## **1. Concept / What**

The **None Network** mode creates a container with **no external network connectivity**. The container has:

* No `eth0` interface
* No access to other containers
* No internet
* No DNS
* **Only** a loopback interface (`lo`) with `127.0.0.1`

This mode fully isolates the container from all network communication.

---

## **2. Why / Purpose / Real-World Use Cases**

The `none` network is used where **absolute isolation** is needed.

### **Security Use Cases**

* Running untrusted code or malware samples
* Sandboxed environments
* Sensitive batch jobs
* Compliance-restricted workflows

### **Build or Test Use Cases**

* CI/CD steps requiring offline builds
* Testing apps for behavior during network failures
* Tools that should never access the network

### **Debugging Use Cases**

* Verify how an application behaves without DNS/internet/host

---

## **3. How it Works / Steps / Syntax**

### **Run a container with no network**

```bash
docker run --network none -d ubuntu sleep infinity
```

### **Check network interfaces inside container**

```bash
docker exec -it <container> ip a
```

Expected output:

```
lo: 127.0.0.1
```

No `eth0` will be present.

### **Network operations will fail**

```bash
ping 8.8.8.8      # fails
curl http://google.com    # fails
```

### **Docker exec still works**

Even with no network, you can run:

```bash
docker exec -it <container> bash
```

Because `docker exec` communicates via Docker daemon, not container network.

---

## **4. Common Issues / Errors**

### **1️⃣ Application cannot connect to database, API, or other services**

Cause: Network mode `none` blocks all outbound/inbound traffic.
**Fix:** Attach container to a proper network:

```bash
docker network connect app-net <container>
```

---

### **2️⃣ Package installation fails inside container**

Cause: No external internet access.
**Fix:** Use bridge network:

```bash
docker run --network bridge ...
```

---

### **3️⃣ DNS lookups fail**

Cause: No DNS because there is no network.
**Fix:** Move container to user-defined or default bridge.

---

### **4️⃣ Logging/Monitoring agents fail**

None network prevents agents from sending data to external systems.

**Fix:** Use host network or bridge for these workloads.

---

## **5. Troubleshooting / Fixes**

### **Check the network mode of a running container**

```bash
docker inspect <container> | grep NetworkMode
```

Expected:

```
"NetworkMode": "none"
```

### **Attach running container to a network** (only user-defined or default bridge)

```bash
docker network connect app-net <container>
```

### **Detach container from a network**

```bash
docker network disconnect app-net <container>
```

### **Note:** You cannot attach a running container to `none` or `host` networks.

You must recreate the container for these modes.

---

## **6. Best Practices / Tips**

### ✔ Use `none` for:

* Isolation-first workloads
* Offline CI/CD steps
* Highly sensitive workloads
* Malware or sandbox testing

### ✖ Avoid `none` for:

* Web apps
* APIs
* Databases
* Microservices
* Anything requiring host or internet access

### ✔ DevOps Best Practices

* Use `docker exec` to access container shell (works even without network)
* Use user-defined bridge for microservices
* Recreate container when switching to host/none network modes

### ✔ EKS Note

Kubernetes does not use `none` mode except for controlled testing scenarios.
Pods require CNI-managed networking for service discovery and routing.

---
---

# Docker Networking – Port Mapping (`-p`) (Detailed Explanation Notes)

## **1. Concept / What**

**Port Mapping** is the mechanism Docker uses to expose applications running inside containers to the **outside world**. It forwards traffic from the **host port** to the **container port**.

Syntax:

```
-p HOST_PORT:CONTAINER_PORT
```

Example:

```
-p 8080:80
```

This means:

* Host listens on port **8080**
* Traffic is forwarded to container port **80**, where the application is running

---

## **2. Why / Purpose / Real-World Use Cases**

Port mapping is used to allow external clients (users, browsers, services) to access applications running inside containers.

### **Key Real-World Use Cases**

* Exposing web apps, APIs, and microservices to users
* Running multiple apps on the same Docker host
* Local development and testing of backend/frontend services
* Jenkins pipelines that need containers to run test servers
* Simulating ALB/NLB target behavior locally

### **Internal Use Case (No Port Mapping Required)**

For container-to-container communication inside the same custom bridge network, containers use:

* **Container name**
* **Container port**
  No `-p` is needed for internal communication.

---

## **3. How it Works / Steps / Syntax**

### **Basic Port Mapping**

```bash
docker run -p 8080:80 nginx
```

Access via:

```
http://HOST-IP:8080
```

### **Expose Multiple Ports**

```bash
docker run -p 8080:80 -p 8443:443 nginx
```

### **Use a Custom Bridge Network with Port Mapping**

```bash
docker network create app-net

docker run -d --name api --network app-net -p 8080:80 my-api
```

### **Check Port Mapping of a Running Container**

```bash
docker port <container>
```

### **Random Port Assignment**

```bash
docker run -P nginx
```

This maps all exposed ports to random host ports.

---

## **4. Real-World Scenarios**

### **1️⃣ Internal DB (no external exposure)**

```bash
docker run --network app-net --name db mysql
```

Containers can reach it using:

```
db:3306
```

But host cannot reach it (secure).

### **2️⃣ Backend App Exposed Internally + Externally**

Internal communication:

```
api:8000
```

External access:

```bash
docker run -p 5000:8000 backend
```

### **3️⃣ Jenkins Interaction with Containers**

```bash
docker run -p 8081:8080 jenkins
```

Jenkins runs on host port 8081.

---

## **5. Common Issues / Errors**

### **1️⃣ Host Port Already in Use**

Error:

```
bind: address already in use
```

Fix:

* Kill the process using the port
* Or use a different host port

### **2️⃣ Container Not Reachable from Host**

Cause: `-p` not specified.
Fix:

```bash
docker run -p 8080:80 ...
```

### **3️⃣ Application Listening Only on 127.0.0.1**

Fix app to listen on:

```
0.0.0.0
```

### **4️⃣ Firewall Blocking Access**

Fix:

```
sudo ufw allow 8080
```

### **5️⃣ Port Mapping Ignored on Host Network**

Because host network bypasses Docker port mapping.
Fix: Use bridge network.

---

## **6. Troubleshooting / Fixes**

### **Check Host Port Usage**

```bash
sudo lsof -i :8080
```

### **Check Container Listening Ports**

```bash
docker exec -it container netstat -tulnp
```

### **Check Mapped Ports**

```bash
docker port container-name
```

### **Check Firewall Status**

```bash
sudo ufw status
```

---

## **7. Best Practices / Tips**

### ✔ Host Ports Must Be Unique

You cannot map two containers to the same host port.

```
-p 80:80
-p 80:8080    # ❌ not allowed
```

### ✔ Container Ports Can Repeat

Multiple containers can expose the same internal port.

```
-p 8080:80
-p 8081:80
```

### ✔ Use Container Names for Internal Communication

On user-defined networks:

```
http://api:8000
```

### ✔ Expose Only Required Ports

Avoid exposing databases or sensitive internal services.

### ✔ Bind Services to 0.0.0.0 Inside Container

Ensures the app is reachable via port mapping.

### ✔ Prefer Bridge + Port Mapping Over Host Network for Apps

More secure, isolated, and flexible.

### ✔ For Load Balancers (EKS), Containers Still Listen on Container Ports

EKS/Kubernetes Services will map ports automatically.

---
---

# Docker Networking – DNS Inside Containers (Detailed Explanation Notes)

## **1. Concept / What**

Docker provides an internal DNS service that runs at **127.0.0.11** inside containers. This DNS automatically resolves **container names → container IPs**, enabling clean, name‑based communication.

### **Key Points**

* Works **only** on **user-defined bridge networks**.
* Does **not** work on default bridge, host network, or none network.
* Container IPs change on each restart, but container **names remain constant**, ensuring stable communication.
* Used for container-to-container service discovery.

---

## **2. Why / Purpose / Real-World Use Cases**

Docker DNS makes internal communication easier, stable, and production-like.

### **DevOps / CI/CD Use Cases**

* Microservices communicating using service names instead of IPs.
* Jenkins pipelines running integration tests between multiple containers.
* Simulating Kubernetes-like service discovery locally.
* Docker Compose setups where services depend on each other.

### **Eliminates IP Dependency**

Container IP changes → DNS ensures names remain consistent.

### **Local Development Use Case**

Frontend → backend → DB can talk using simple names:

```
backend:8000
redis:6379
mysql:3306
```

---

## **3. How it Works / Steps / Syntax**

### **Check DNS server inside a container**

```bash
docker exec -it api cat /etc/resolv.conf
```

Output:

```
nameserver 127.0.0.11
```

### **Create user-defined bridge network**

```bash
docker network create app-net
```

### **Run containers on custom network**

```bash
docker run -d --name api --network app-net backend-app

docker run -d --name db --network app-net mysql
```

### **Container name resolution**

Inside `api`:

```bash
ping db         # Works
nslookup db     # Shows container IP
```

Docker DNS resolves:

```
db → 172.18.0.x
```

### **Docker Compose DNS support**

`docker-compose.yml`:

```yaml
services:
  api:
    build: .
    depends_on:
      - db
  db:
    image: mysql
```

Containers can reach:

```
http://db:3306
```

No configuration needed.

---

## **4. DNS Behavior by Network Type**

| Network Mode            | Container Name DNS | IP Routing | Notes                              |
| ----------------------- | ------------------ | ---------- | ---------------------------------- |
| **User-defined bridge** | ✔ Works            | ✔ Yes      | Best for microservices             |
| **Default bridge**      | ❌ No               | ✔ Yes      | Requires old `--link` (deprecated) |
| **Host network**        | ❌ No               | ✔ Yes      | Container uses host DNS            |
| **None network**        | ❌ No               | ❌ No       | No networking at all               |

---

## **5. Common Issues / Errors**

### **1️⃣ `ping: bad address`**

Cause: Container is on default bridge.
**Fix:** Use user-defined network.

### **2️⃣ IP changes after restart**

Cause: Using IP instead of name.
**Fix:** Always use container names.

### **3️⃣ DNS doesn’t work in host network**

Because container uses host’s DNS.
**Fix:** Move to user-defined bridge.

### **4️⃣ `none` network: DNS not available**

Expected — no network exists.

### **5️⃣ Duplicate container names**

Two containers with the same name break DNS.
**Fix:** Use unique container names.

---

## **6. Troubleshooting / Fixes**

### **Check DNS resolver**

```bash
cat /etc/resolv.conf
```

### **Test name resolution**

```bash
docker exec -it api nslookup db
```

### **Find container IP**

```bash
docker inspect db | grep IPAddress
```

### **Verify active networks**

```bash
docker network inspect app-net
```

---

## **7. Best Practices / Tips**

### ✔ Always use user-defined bridge networks for microservices

### ✔ Use container names instead of IPs for service communication

### ✔ Avoid default bridge; no DNS support

### ✔ Use Docker Compose for multi-service DNS automation

### ✔ Validate DNS with `nslookup` or `dig` during debugging

### ✔ Keep container names unique

### ✔ For external DNS, configure Docker daemon with `--dns` or `/etc/docker/daemon.json`

---
---

# Docker Networking – User-Defined Network (Detailed Explanation Notes)

## **1. Concept / What**

A **user-defined bridge network** is a custom Docker network created by the user. It provides:

* Automatic **DNS-based service discovery** (container name → IP)
* Container-to-container communication using **names** instead of IPs
* Better isolation from other applications
* Ability to attach/detach containers dynamically
* Stable networking for multi-container microservices

This is the **recommended network type** for real-world Docker-based applications.

---

## **2. Why / Purpose / Real-World Use Cases**

User-defined networks solve multiple real DevOps and microservices needs.

### **Key Use Cases**

* Inter-container communication using names:

  * `backend:8000`
  * `db:3306`
* Stable service discovery (IP changes do not break apps)
* Isolation between independent projects
* Running multi-service apps locally before pushing to EKS
* Jenkins integration tests involving multiple containers
* Cleaner DNS and simpler debugging

### **Why Not Default Bridge?**

Default bridge does **not** provide container-name DNS.
User-defined networks **automatically provide DNS**.

---

## **3. How It Works / Steps / Syntax**

### **Create a user-defined network**

```bash
docker network create app-net
```

### **Run containers on the custom network**

```bash
docker run -d --name backend --network app-net backend-image

docker run -d --name db --network app-net mysql
```

### **Container-to-container communication (DNS)**

Inside backend:

```bash
curl http://db:3306
```

Docker DNS resolves:

```
db → 172.18.0.x
```

### **Attach running container to a custom network**

```bash
docker network connect app-net web
```

### **Detach container from a network**

```bash
docker network disconnect app-net web
```

### **Inspect network**

```bash
docker network inspect app-net
```

Shows connected containers, IPs, MAC addresses, DNS entries.

---

## **4. Real-World Scenarios**

### **1️⃣ Microservices Architecture Locally**

* frontend ↔ backend ↔ database
* Communication using names (not IPs)

### **2️⃣ Jenkins CI Pipelines**

Run test containers that interact with each other via DNS.

### **3️⃣ Simulating Kubernetes Locally**

Kubernetes uses service names → IPs.
User-defined networks mimic that behavior.

### **4️⃣ Isolated Networks Per Application**

Different apps use their own custom networks for isolation.

---

## **5. Common Issues / Errors**

### **1️⃣ Name resolution not working**

Cause: Using default bridge network.
Fix:

```bash
docker run --network app-net ...
```

### **2️⃣ Port conflicts on host**

Cause: Exposing multiple containers with same host port.
Fix: Use unique host ports:

```
-p 8080:80
-p 8081:80
```

### **3️⃣ Duplicate container names**

Two containers cannot share the same name → breaks DNS.
Fix: Use unique names.

### **4️⃣ Application listening on localhost only**

Containers cannot bind `127.0.0.1` for external traffic.
Fix: Listen on `0.0.0.0`.

### **5️⃣ Cannot attach to host/none networks dynamically**

Only user-defined and default bridge networks support dynamic connect/disconnect.

---

## **6. Troubleshooting / Fixes**

### **Check DNS**

```bash
docker exec -it backend cat /etc/resolv.conf
```

Should show: `127.0.0.11` (Docker DNS).

### **Test name resolution**

```bash
nslookup db
```

### **Check container IP**

```bash
docker inspect db | grep IPAddress
```

### **List connected containers**

```bash
docker network inspect app-net
```

---

## **7. Best Practices / Tips**

### ✔ Always use user-defined bridge networks for microservices

### ✔ Use container names instead of IPs (IP works but changes frequently)

### ✔ Give networks meaningful names (`app-net`, `dev-net`, `ci-net`)

### ✔ Keep databases internal (do not expose with `-p`)

### ✔ Use dynamic attach/detach for debugging

### ✔ Let Docker Compose handle network creation automatically

### ✔ Maintain network isolation between unrelated apps

---
---

# Docker Networking – Container-to-Container Communication (Detailed Explanation Notes)

## **1. Concept / What**

Container-to-container communication refers to how two or more containers can talk to each other across Docker networks. Communication depends entirely on the **network type** and how the containers are attached.

Containers can communicate using:

* **Container names** (DNS-based – recommended)
* **Container IP addresses** (works, but not recommended)
* **Host IP + Host Port** (rare case – only when necessary)

---

## **2. Why / Purpose / Real-World Use Cases**

Used for communication between services inside Docker, such as:

* Backend → Database
* Frontend → Backend
* API → Redis / MongoDB / MySQL
* Workers → Message Queues (RabbitMQ, Kafka)
* Jenkins integration test services

### **Key Benefits**

* Stable communication without relying on changing IPs
* Service discovery using container names
* Separation of internal traffic vs external traffic
* Easier debugging and multi-container orchestration

---

## **3. How It Works (By Network Type)**

### **A. User-Defined Bridge Network (Recommended)**

Provides **DNS-based container name resolution**.

Example:

```bash
docker network create app-net

docker run -d --name api --network app-net api-image

docker run -d --name db --network app-net mysql
```

Inside `api`:

```bash
curl http://db:3306
```

Docker’s DNS resolves:

```
db → 172.18.0.x
```

---

### **B. Default Bridge Network (Not Recommended)**

* DNS **does not work** → cannot resolve names
* Only IP-based communication possible

Example:

```bash
curl http://172.17.0.3:3306
```

(This breaks when container restarts due to IP changes.)

---

### **C. Host Network**

* Containers share the host network stack
* No internal DNS
* Used mainly for agents, monitoring tools, not microservices

---

### **D. None Network**

* No communication possible
* Only loopback (`lo`) exists

---

## **4. Communication Methods (Detailed)**

### **1️⃣ Container Name (Best Practice)**

Works only in **user-defined networks**.

```
http://backend:8000
http://db:3306
```

Most stable approach.

---

### **2️⃣ Container IP (Possible but Not Recommended)**

```
http://172.18.0.3:80
```

Downside: IP changes when container restarts.

---

### **3️⃣ Host IP + Host Port (Rare Case)**

Only used when:

* Containers are on different networks
* One container needs to simulate an external request
* Testing load balancer flows locally

Example:

```
http://HOST-IP:8080
```

**This is NOT for internal communication** in microservices.
It is used **only** for external access.

---

## **5. Real-World Scenarios**

### **1️⃣ Backend ↔ Database**

Backend uses:

```
db:3306
```

### **2️⃣ Frontend ↔ API**

Frontend uses:

```
api:8080
```

### **3️⃣ Jenkins Test Pipelines**

Test containers communicate via service names.

### **4️⃣ Local Kubernetes Simulation**

Pods use service names → Docker user-defined networks mimic this.

---

## **6. Common Issues / Errors**

### **1️⃣ `ping: bad address`**

Cause: Using default bridge.
Fix:

```bash
docker run --network app-net ...
```

### **2️⃣ Connection refused**

Cause: App inside container not listening on correct port.
Fix:

* Ensure app listens on `0.0.0.0`

---

### **3️⃣ Timeout / unreachable container**

Cause: Containers on different networks.
Fix:

```bash
docker network connect app-net container1
```

---

### **4️⃣ Name conflict**

Duplicate container names break DNS.
Fix: Use unique names.

---

### **5️⃣ Using host network accidentally**

Port mapping ignored → communication breaks.
Fix: Use bridge network.

---

## **7. Troubleshooting / Fixes**

### **Check container IP**

```bash
docker inspect db | grep IPAddress
```

### **Check DNS**

```bash
docker exec -it api cat /etc/resolv.conf
```

Should show:

```
nameserver 127.0.0.11
```

### **Test name resolution**

```bash
docker exec -it api nslookup db
```

### **Check network attachments**

```bash
docker network inspect app-net
```

---

## **8. Best Practices / Tips**

### ✔ Use user-defined networks for microservices

### ✔ Use container names for communication

### ✔ Avoid IPs because they change

### ✔ Expose only necessary ports to host

### ✔ Do not use host network for microservices

### ✔ Keep unrelated microservices on separate networks

### ✔ Use Docker Compose for multi-service DNS handling

---
---

# Docker Networking Basics

## 1. docker0 (Default Bridge Network)

* `docker0` is a **Linux bridge network interface** created automatically by Docker.
* It implements the **default Docker bridge network**.
* It has:

  * A **CIDR/subnet** (commonly `172.17.0.0/16`)
  * A **gateway IP** (usually `172.17.0.1`)
* Containers that do not specify a network are attached to `docker0`.

**Key point:**

* There is **only one docker0** on a host.

---

## 2. Custom (User-Defined) Bridge Networks

* When we create a custom Docker network, Docker **does NOT reuse docker0**.
* Docker creates a **separate Linux bridge interface** with a name like:

  * `br-xxxx`
* Each custom bridge has:

  * Its **own subnet**
  * Its **own gateway IP**
* Containers attached to different bridge networks are **isolated by default**.

**Key point:**

* docker0 is only for the default network.
* Custom networks use **their own bridge interfaces**.

---

## 3. IP Assignment (Who assigns IPs?)

* Docker has an internal **IPAM (IP Address Management)** system.
* Docker assigns container IPs **from the subnet of the bridge**.
* The bridge interface (docker0 or br-xxxx):

  * Acts as the **gateway**
  * Represents the network

---

## 4. veth (Virtual Ethernet)

* `veth` stands for **virtual ethernet**.
* veth interfaces always come in **pairs**.
* A veth pair is used to connect a container to a bridge network.

---

## 5. eth0 (Inside the Container)

* `eth0` is the **network interface inside the container**.
* It is **one end of the veth pair**.
* It receives an IP address from the bridge subnet.
* From the container’s point of view, `eth0` looks like a normal network card.

---

## 6. veth Interface (Host Side)

* The other end of the veth pair exists on the **host**.
* It appears as an interface named like `vethXXXX`.
* This host-side veth is attached to:

  * `docker0` (default network), or
  * `br-xxxx` (custom network)

---

## 7. How Everything Connects Together

```
Container namespace        Host namespace
------------------        ----------------
eth0   <===========>   vethXXXX  --->  docker0 / br-xxxx
```

* `eth0` (container) + `vethXXXX` (host) = **veth pair**
* Bridge (`docker0` or `br-xxxx`) routes traffic between containers and outside networks.

---

## 8. Interview-Ready Summary

* `docker0` is a Linux bridge network interface for the default Docker network.
* Custom Docker networks create separate bridge interfaces (`br-xxxx`).
* Docker assigns container IPs from the bridge subnet using IPAM.
* Containers connect to bridges using a veth pair:

  * `eth0` inside the container
  * `vethXXXX` on the host

---
---
---

