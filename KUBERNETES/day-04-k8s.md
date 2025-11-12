# Kubernetes Networking Model & CNI Plugins (Detailed Explanation)

## 🧩 Concept / What

The **Kubernetes Networking Model** defines how Pods, Services, and external systems communicate inside a cluster. It ensures seamless Pod-to-Pod, Pod-to-Service, and Pod-to-External connectivity without the need for manual IP configuration or NAT.

The **Container Network Interface (CNI)** is the underlying mechanism that enables this communication. It assigns IPs to Pods, sets up routes, and ensures that network traffic flows correctly across nodes and Pods.

### Core Principles of Kubernetes Networking:

1. Every **Pod** gets its **own unique IP address**.
2. **All Pods can communicate** with each other **without NAT**.
3. **Nodes and Pods** can communicate directly.
4. **Services** provide stable endpoints (ClusterIP) for Pod groups.

---

## 🎯 Why / Purpose / Real-World Use Cases

Kubernetes networking removes the complexity of managing IPs manually and allows microservices to communicate reliably.

**Key use cases:**

* Communication between microservices (frontend → backend → DB).
* Load balancing between multiple Pods of the same Service.
* Dynamic scaling — new Pods join automatically and start communicating.
* Integration with cloud networking (e.g., AWS VPC CNI for native routing).

**Example:** In an e-commerce app, frontend Pods talk to backend Pods and backend Pods talk to database Pods seamlessly without manual network setup.

---

## ⚙️ How it Works / Steps / Syntax

### 1. Pod Creation

When a Pod is created, Kubernetes asks the container runtime (like containerd) to set up networking for it.

### 2. CNI Plugin Involvement

The runtime calls the **CNI plugin** (AWS VPC CNI, Calico, Flannel, etc.), which:

* Creates a **network namespace** for the Pod.
* Creates a **virtual Ethernet (veth) pair** connecting the Pod to the node.
* Assigns an **IP address** to the Pod.
* Adds **routes** so that the Pod can reach other Pods and external endpoints.

### 3. Pod IP Assignment

* **AWS VPC CNI** → IPs are assigned directly from the **VPC Subnet CIDR**.
* **Calico/Flannel** → IPs are assigned from a **Pod CIDR range** configured during cluster setup.

### 4. Communication Flow

| Type                | Example                                 | Description                                  |
| ------------------- | --------------------------------------- | -------------------------------------------- |
| Pod-to-Pod          | `curl http://10.0.2.4:8080`             | Direct via Pod IP                            |
| Pod-to-Service      | `curl http://backend-service:8080`      | Uses Service cluster IP and kube-proxy rules |
| External-to-Service | Browser → LoadBalancer → NodePort → Pod | Exposes cluster apps to external users       |

### Example Pod Manifest for Testing

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app1
  labels:
    app: app1
spec:
  containers:
  - name: app1
    image: busybox
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: app2
  labels:
    app: app2
spec:
  containers:
  - name: app2
    image: busybox
    command: ["sleep", "3600"]
```

Test connectivity:

```bash
kubectl exec -it app1 -- ping app2
```

If networking is configured correctly, the Pods will communicate using their Pod IPs.

---

## 🧩 Common CNI Plugins

| Plugin          | IP Source    | Advantages                               | Limitations                           |
| --------------- | ------------ | ---------------------------------------- | ------------------------------------- |
| **AWS VPC CNI** | VPC subnet   | Native AWS networking, no NAT, uses ENIs | Limited Pods per node (based on ENIs) |
| **Calico**      | Pod CIDR     | NetworkPolicies, BGP routing             | Slightly more complex setup           |
| **Flannel**     | Pod CIDR     | Simple, lightweight overlay              | No advanced security policies         |
| **Weave Net**   | Overlay mesh | Auto-discovery, encryption               | Slightly higher latency               |

---

## ⚠️ Common Issues / Errors

| Issue                          | Description                                 |
| ------------------------------ | ------------------------------------------- |
| Pods not getting IPs           | CNI DaemonSet crashed or IP range exhausted |
| Cross-node communication fails | Misconfigured routes or subnet overlap      |
| DNS resolution fails           | CoreDNS Pod failure or kube-dns misconfig   |
| IP overlap                     | Conflicting VPC and Pod CIDRs               |

---

## 🧰 Troubleshooting / Fixes

* Check CNI status:

  ```bash
  kubectl get pods -n kube-system -o wide
  ```
* Verify Pod IPs and routes:

  ```bash
  ip addr
  ip route
  ```
* Test DNS resolution:

  ```bash
  kubectl exec -it <pod> -- nslookup kubernetes.default
  ```
* Restart CNI or CoreDNS components:

  ```bash
  kubectl rollout restart daemonset aws-node -n kube-system
  kubectl rollout restart deployment coredns -n kube-system
  ```

---

## 🧠 Best Practices / Tips

✅ Use a stable, supported **CNI plugin** (AWS VPC CNI, Calico, Flannel).
✅ Avoid **overlapping CIDR ranges** between cluster and VPC.
✅ Always verify **CoreDNS health** post-deployment.
✅ Use **NetworkPolicies** for security segmentation.
✅ Monitor **aws-node** / **calico-node** logs for IP or routing issues.
✅ Scale subnets if using AWS VPC CNI to prevent IP exhaustion.

---

## 🧭 Summary

* Kubernetes networking allows seamless communication between Pods, Services, and external systems.
* **CNI plugins** are responsible for Pod IP management, route creation, and communication setup.
* Without a working CNI, Pods won’t have IPs or network connectivity.
* AWS VPC CNI integrates natively with AWS networking for EKS clusters.

---

**End of Detailed Explanation Version**

---
---

# Kubernetes Services (ClusterIP, NodePort, LoadBalancer)

## 🧩 Concept / What

A **Service** in Kubernetes provides a **stable endpoint (IP and DNS name)** to access a group of Pods. Since Pod IPs change when Pods are recreated, a Service ensures that communication inside or outside the cluster remains consistent.

There are three main types of Services:

1. **ClusterIP** – Exposes Pods internally within the cluster.
2. **NodePort** – Exposes Pods externally via a static port on each Node.
3. **LoadBalancer** – Exposes Pods externally using a cloud provider’s load balancer (e.g., AWS ELB/NLB).

Each Service type builds on the previous one:

```
LoadBalancer → NodePort → ClusterIP
```

This means every LoadBalancer internally uses NodePort and ClusterIP, and every NodePort internally uses ClusterIP.

---

## 🎯 Why / Purpose / Use Case

* **ClusterIP:** For internal communication (frontend → backend → database).
* **NodePort:** For accessing applications from outside the cluster using a Node’s IP.
* **LoadBalancer:** For public access through a cloud-managed load balancer (e.g., production web apps).

Example in a microservice app:

* Frontend needs to reach backend (ClusterIP Service)
* Backend needs to reach database (ClusterIP Service)
* Users access frontend from browser via LoadBalancer

---

## ⚙️ How It Works / Steps / Syntax

### 🔹 ClusterIP (Default Type)

* When you create a Service with `type: ClusterIP`, Kubernetes gives it a **virtual IP (ClusterIP)**.
* This ClusterIP acts like a **single stable entry point** to reach multiple Pods behind it.
* Traffic to the ClusterIP is **load-balanced automatically** among all the Pods selected by the Service.

**Example YAML:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 80        # Service port
    targetPort: 8080 # Container port
  type: ClusterIP
```

**How it works:**
If 4 Pods are running for the backend, the ClusterIP Service (say `10.96.0.10`) automatically distributes traffic between all 4 Pods in round-robin fashion.

**Example Access:**

```bash
kubectl exec -it frontend-pod -- curl http://backend-service:80
```

**Visual Flow:**

```
Client Pod → ClusterIP (Virtual IP) → [Pod1, Pod2, Pod3, Pod4]
```

✅ ClusterIP = Virtual IP + Internal Load Balancer among Pods.

---

### 🔹 NodePort (Builds on ClusterIP)

* When you set `type: NodePort`, Kubernetes:

  1. Creates a **ClusterIP**.
  2. Opens a **NodePort** (30000–32767) on each node.
* You can access the app from outside using `http://<NodeIP>:<NodePort>`.

**Example YAML:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Optional static port
  type: NodePort
```

**Flow:**

```
External Client → NodeIP:NodePort → ClusterIP → Pods
```

✅ NodePort = ClusterIP + External Node Access.

---

### 🔹 LoadBalancer (Builds on NodePort + ClusterIP)

* LoadBalancer integrates with cloud providers like AWS, GCP, or Azure.
* It first creates a ClusterIP and NodePort, then provisions a **cloud load balancer (NLB/ELB)** to forward traffic.

**Example YAML:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

**Flow:**

```
Internet → AWS NLB → NodePort → ClusterIP → Pods
```

✅ LoadBalancer = NodePort + ClusterIP + Cloud LB.

---

## 🧠 How Kubernetes Decides Where Traffic Goes

Each Service has its own **ClusterIP** and uses **selectors (labels)** to route to the correct set of Pods.

| Service          | Selector     | ClusterIP  | Routes to     |
| ---------------- | ------------ | ---------- | ------------- |
| frontend-service | app=frontend | 10.96.0.10 | frontend Pods |
| backend-service  | app=backend  | 10.96.0.20 | backend Pods  |
| db-service       | app=db       | 10.96.0.30 | database Pods |

✅ ClusterIP ensures load balancing across multiple Pods behind that Service.

---

## ⚠️ Common Issues / Errors

| Issue                              | Cause                                     |
| ---------------------------------- | ----------------------------------------- |
| `EXTERNAL-IP` stuck as `<pending>` | Cloud controller missing or misconfigured |
| NodePort unreachable               | Security group or firewall blocking port  |
| No traffic reaching Pods           | Label mismatch between Service and Pods   |
| DNS not resolving Service name     | CoreDNS down or wrong namespace           |
| TargetPort mismatch                | Incorrect port mapping                    |

---

## 🧰 Troubleshooting / Fixes

* Check Services and Endpoints:

  ```bash
  kubectl get svc
  kubectl get endpoints <service-name>
  ```
* Verify Pod labels:

  ```bash
  kubectl get pods --show-labels
  ```
* Test DNS:

  ```bash
  kubectl exec -it <pod> -- nslookup backend-service
  ```
* Recreate stuck LoadBalancer:

  ```bash
  kubectl delete svc webapp-service && kubectl apply -f webapp-service.yaml
  ```

---

## 💡 Best Practices / Tips

✅ ClusterIP is used for internal communication only.
✅ NodePort is for simple external testing or internal use.
✅ LoadBalancer is for public, production-grade external access.
✅ Each higher-level Service type internally uses the lower one.
✅ Always verify labels — that’s how Services know which Pods to send traffic to.
✅ Secure LoadBalancers with WAF, SGs, or Ingress if public.

---

## 🧭 Summary

* **ClusterIP** = Internal-only + load balances traffic between Pods.
* **NodePort** = ClusterIP + external access via Node IP.
* **LoadBalancer** = NodePort + ClusterIP + external cloud load balancer.
* Kubernetes automatically creates lower layers when higher types are used.
* Ensures stable, scalable, and fault-tolerant Pod communication.

---

**End of Detailed Explanation Version**

---
---

# AWS VPC CNI Plugin (Detailed Explanation)

## 🧩 Concept / What

The **AWS VPC CNI Plugin** (Container Network Interface) is the **default networking plugin** for Amazon EKS. It allows Pods to receive IP addresses **directly from the VPC subnet**, enabling them to behave like native EC2 instances in terms of networking and communication.

With the AWS VPC CNI, Pods can:

* Communicate directly with other AWS services (like RDS, S3, or EC2) using private IPs.
* Appear in the **VPC routing tables and flow logs**.
* Be controlled using **AWS Security Groups**.

---

## 🎯 Why / Purpose / Real-World Use Case

Without the VPC CNI, Kubernetes Pods typically receive IPs from a separate **Pod CIDR** range. This creates a logical separation between cluster networking and AWS networking.

The AWS VPC CNI solves that by integrating Pods into the **VPC’s native IP space**, so:

* You don’t need NAT gateways for Pod-to-AWS communication.
* Pods can directly use private IPs within the subnet.
* Network visibility and security remain consistent across EC2 and Pods.

**Example use case:**
When a backend Pod in EKS connects to an RDS database in the same VPC, it can use the RDS’s private IP directly without NAT translation.

---

## ⚙️ How It Works / Steps / Syntax

### 1️⃣ Node and ENIs

Each **EC2 worker node** has:

* One **primary ENI** (Elastic Network Interface).
* One or more **secondary ENIs** added by the CNI plugin.

Each ENI can have multiple **secondary private IPs**. The CNI assigns these IPs to Pods running on that node.

---

### 2️⃣ IP Assignment Process

1. Node joins the EKS cluster.
2. The `aws-node` DaemonSet (the CNI plugin) runs on that node.
3. The plugin attaches extra ENIs to the node and allocates **secondary IPs** from the VPC subnet.
4. When a new Pod is created, it’s assigned one of these secondary IPs.
5. The Pod can now communicate directly within the VPC — no NAT required.

**Flow:**

```
Subnet CIDR → ENI → Secondary IPs → Assigned to Pods
```

---

### 3️⃣ Communication Paths

| Type                        | How It Works                        |
| --------------------------- | ----------------------------------- |
| Pod → Pod (same node)       | Direct veth connection              |
| Pod → Pod (different nodes) | Routed via ENIs and VPC routing     |
| Pod → AWS services          | Native VPC networking (private IPs) |

---

## 🧱 Example Visualization

```
+------------------------------------+
| EC2 Node (Worker)                 |
|   ENI-1: 10.0.1.10 (primary)      |
|   ENI-2: 10.0.1.11–10.0.1.14      |
|   Pods get IPs:                   |
|     Pod1 → 10.0.1.11              |
|     Pod2 → 10.0.1.12              |
|     Pod3 → 10.0.1.13              |
|     Pod4 → 10.0.1.14              |
+------------------------------------+
```

Each Pod’s IP comes directly from the **subnet CIDR**.

---

## 🧮 ENIs, IPs, and Instance Types

Each EC2 instance type supports a **fixed number of ENIs** and **IPs per ENI**. The total number of Pods that can run on a node is based on this combination.

| Instance Type | Max ENIs | IPs per ENI | Theoretical Total IPs | Realistic Usable Pods | AWS MaxPods (EKS setting) |
| ------------- | -------- | ----------- | --------------------- | --------------------- | ------------------------- |
| t3.medium     | 3        | 6           | 18                    | ~16                   | 17                        |
| m5.large      | 3        | 10          | 30                    | ~28–29                | 31                        |

### 🔹 Why the Difference?

* The **primary ENI**’s primary IP is used by the node itself.
* Each ENI reserves 1 IP for management and warm pool.
* The **AWS VPC CNI plugin** also pre-allocates a few IPs for faster Pod creation.

Hence, real-world usable Pod IPs are always slightly less than the theoretical total.

---

## ⚠️ Common Issues / Errors

| Issue                              | Description / Cause                                |
| ---------------------------------- | -------------------------------------------------- |
| Pods stuck in `ContainerCreating`  | No available IPs in subnet or ENI limit reached    |
| Cross-node Pod communication fails | VPC routing or Security Group issue                |
| IP exhaustion                      | Subnet CIDR too small or node over-scheduled       |
| Version mismatch                   | aws-node DaemonSet not compatible with EKS version |

---

## 🧰 Troubleshooting / Fixes

* Check DaemonSet:

  ```bash
  kubectl get daemonset aws-node -n kube-system
  ```
* View logs:

  ```bash
  kubectl logs -n kube-system -l k8s-app=aws-node
  ```
* Verify ENIs and IPs:

  ```bash
  ip addr
  ```
* Check available IPs in subnet:
  AWS Console → VPC → Subnets → Available IP addresses
* Restart CNI plugin:

  ```bash
  kubectl rollout restart daemonset aws-node -n kube-system
  ```

---

## 🧠 Best Practices / Tips

✅ Choose EC2 instance types with sufficient ENIs and IPs for your expected Pod count.
✅ Monitor IP usage with CloudWatch metrics.
✅ Use larger VPC subnets to avoid IP exhaustion.
✅ Keep the `aws-node` plugin updated with EKS version upgrades.
✅ Use **Security Groups for Pods (SGP)** for fine-grained access control.
✅ Prefer private subnets for worker nodes to keep Pod traffic internal.

---

## 🧾 Key Takeaway

> The number of Pods per node in EKS **depends on the instance type**, because the AWS VPC CNI assigns Pod IPs using the node’s ENIs and their available secondary IPs.
> AWS enforces this through the **maxPods setting**, which varies by instance type and ensures efficient IP allocation.

---

**End of Detailed Explanation Version**

---
---

# CoreDNS Service Discovery (Detailed Explanation)

## 🧩 Concept / What

**CoreDNS** is the **DNS server** used by Kubernetes (and EKS) to enable **Service name resolution** inside the cluster.
It runs as a **Deployment** in the **`kube-system` namespace** and provides DNS-based **Service Discovery** for Pods.

Whenever a Pod tries to connect to another Service using its name (like `backend-service`), CoreDNS resolves that name to the Service’s **ClusterIP**.

✅ **Example:**

```
frontend → http://backend-service → CoreDNS → ClusterIP → backend Pods
```

---

## 🎯 Why / Purpose / Real-World Use Case

Without CoreDNS, Pods would have to connect using IPs, which constantly change when Pods are recreated.
CoreDNS provides a **consistent and dynamic DNS mechanism**, allowing Services and Pods to communicate easily within and across namespaces.

**Real-world examples:**

* Frontend Pod connects to `backend-service` without worrying about Pod IPs.
* Monitoring tools (like Prometheus) dynamically discover new Services.
* DNS queries for external domains (e.g., `google.com`) are also forwarded through CoreDNS.

---

## ⚙️ How It Works / Steps

### 1️⃣ **CoreDNS Deployment in EKS**

CoreDNS is automatically installed in the EKS cluster under the `kube-system` namespace:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Output:

```
coredns-7db4b97b4f-5r6x8   1/1   Running   0   5m
coredns-7db4b97b4f-hc9s2   1/1   Running   0   5m
```

It usually runs **2 replicas** for high availability.

---

### 2️⃣ **CoreDNS Service (ClusterIP)**

CoreDNS listens on a ClusterIP (commonly `10.96.0.10`).
Each Pod’s `/etc/resolv.conf` points to this IP as its DNS resolver.

Example inside any Pod:

```bash
kubectl exec -it mypod -- cat /etc/resolv.conf
```

Output:

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
```

This tells the Pod to use CoreDNS for all DNS lookups.

---

### 3️⃣ **DNS Resolution Flow**

When a Pod connects to another Service by name (e.g., `backend-service`):

```
Pod → /etc/resolv.conf → CoreDNS (10.96.0.10)
      ↓
CoreDNS queries Kubernetes API → Finds matching Service → Returns ClusterIP
```

Then the Pod sends traffic to that ClusterIP, which forwards it to one of the backend Pods.

---

### 4️⃣ **DNS Naming Convention**

Every Service in Kubernetes automatically gets an FQDN (Fully Qualified Domain Name):

```
<service-name>.<namespace>.svc.cluster.local
```

Example:

```
backend-service.default.svc.cluster.local
```

✅ Within the same namespace, you can simply use `backend-service`.
✅ Across namespaces, use the full name like `backend-service.dev.svc.cluster.local`.

---

### 5️⃣ **CoreDNS ConfigMap (Configuration)**

The behavior of CoreDNS is controlled through a ConfigMap located in the `kube-system` namespace:

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

Key directives:

* `kubernetes cluster.local` → Handles Service DNS inside the cluster.
* `forward . /etc/resolv.conf` → Forwards external DNS queries.
* `cache 30` → Caches DNS responses for 30 seconds.

---

## 🧩 **How CoreDNS Works in EKS**

* Runs as a **Deployment** with 2 Pods (HA mode).
* Exposes a **ClusterIP Service** (default: 10.96.0.10).
* Automatically configured in each Pod’s `/etc/resolv.conf`.
* Resolves **Service names** to **ClusterIPs**.
* Optionally, can resolve **Pod hostnames** in headless Services.

---

## ⚠️ Common Issues / Errors

| Issue                          | Possible Cause                                   |
| ------------------------------ | ------------------------------------------------ |
| DNS resolution fails           | CoreDNS Pods not running or misconfigured        |
| Slow name resolution           | Cache TTL too low or CoreDNS overloaded          |
| External domains not resolving | Wrong `forward` config in CoreDNS ConfigMap      |
| FQDN not resolving             | Wrong namespace in Service name                  |
| Internal services unreachable  | ClusterIP mismatch or NetworkPolicy blocking DNS |

---

## 🧰 Troubleshooting / Fixes

* **Check CoreDNS Pods:**

  ```bash
  kubectl get pods -n kube-system -l k8s-app=kube-dns
  ```
* **View CoreDNS logs:**

  ```bash
  kubectl logs -n kube-system -l k8s-app=kube-dns
  ```
* **Test DNS resolution:**

  ```bash
  kubectl run -it dnsutils --image=busybox -- nslookup kubernetes.default
  ```
* **Restart CoreDNS if needed:**

  ```bash
  kubectl rollout restart deployment coredns -n kube-system
  ```

---

## 🧠 Best Practices / Tips

✅ Always run at least **2 replicas** of CoreDNS for HA.
✅ Don’t modify CoreDNS ConfigMap unless necessary.
✅ For large clusters, enable **NodeLocal DNSCache** to improve performance.
✅ Monitor CoreDNS metrics with Prometheus.
✅ Use correct FQDNs for cross-namespace communication.
✅ In EKS, update CoreDNS via managed add-ons (`aws eks update-addon --addon-name coredns`).

---

## 🧾 Key Takeaway

> CoreDNS in EKS runs as a **Deployment** inside the `kube-system` namespace. It resolves **Service names (and FQDNs)** into **ClusterIPs**, enabling Pods to communicate easily within and across namespaces. It’s automatically deployed by EKS during cluster creation and can be customized using the CoreDNS ConfigMap.

---

**End of Detailed Explanation Version**

---
---

# DNS Basics (Detailed Explanation)

## 🧩 Concept / What

**DNS (Domain Name System)** is like the **phonebook of the internet**. It converts human-readable domain names (like `www.google.com`) into IP addresses (like `142.250.182.4`) so that browsers, servers, and applications can communicate with each other.

> Computers understand IPs, humans remember names — DNS connects those two worlds.

---

## ⚙️ How DNS Works (Step-by-Step)

1. You type `www.google.com` in your browser.
2. Your computer asks a DNS resolver for the IP address.
3. The resolver contacts the **root DNS server**.
4. The root server points to the **TLD (Top-Level Domain)** server — for `.com`.
5. The TLD server points to **Google’s authoritative name server**.
6. That server returns the actual IP (e.g., `142.250.182.4`).
7. The resolver caches it and sends it back to your browser.

✅ Result: your browser now knows which server to connect to.

---

## 📘 DNS Record Types (Simplified Table)

| Record Type | Purpose                                            | Example                               |
| ----------- | -------------------------------------------------- | ------------------------------------- |
| **A**       | Maps a name to an **IPv4 address**                 | `app.example.com → 13.126.45.7`       |
| **AAAA**    | Maps a name to an **IPv6 address**                 | `app.example.com → 2406:da1a:1234::1` |
| **CNAME**   | Creates an alias (points one name to another name) | `www.example.com → app.example.com`   |
| **PTR**     | Reverse lookup (IP → domain name)                  | `13.126.45.7 → app.example.com`       |
| **MX**      | Mail routing (for email servers)                   | `example.com → mail.example.com`      |
| **TXT**     | Stores text info (like SPF, verification)          | `v=spf1 include:_spf.google.com`      |
| **NS**      | Points to authoritative name servers               | `example.com → ns1.example.com`       |

---

## 🧱 DNS Structure & Ownership

Example domain: `www.app.example.com`

| Part                  | Type                       | Who Owns / Controls It                   |
| --------------------- | -------------------------- | ---------------------------------------- |
| `.com`                | **Top-Level Domain (TLD)** | Common for everyone, managed globally    |
| `example.com`         | **Second-Level Domain**    | Unique — you buy this from a provider    |
| `app.example.com`     | **Subdomain**              | You create/manage this under your domain |
| `www.app.example.com` | **Nested Subdomain**       | You can create as many as you want       |

✅ You must **purchase your own domain** (like `example.com`) from providers like Route 53, GoDaddy, or Namecheap.
Once you own it, you can create **unlimited custom subdomains** (like `api.example.com`, `dev.example.com`, etc.).

---

## 🧩 Subdomains and CNAME

* The **part before the first dot** (like `www`, `app`, `api`) is called a **subdomain**.
* You can use a **CNAME record** to point one subdomain to another domain name.

Example:

```
www.example.com → (CNAME) → app.example.com
app.example.com → (A record) → 13.126.45.7
```

✅ Meaning: when someone visits `www.example.com`, DNS redirects it to `app.example.com`, which resolves to the server’s IP.

---

## 🧠 Why CNAME is Used (Real-World Cases)

| Use Case                | Description                                                                         |
| ----------------------- | ----------------------------------------------------------------------------------- |
| **Friendly URLs**       | Make user-friendly aliases (e.g., `www.example.com` → `app.example.com`)            |
| **Cloud Hosting**       | Point to AWS/GCP hostnames (e.g., `app.example.com` → `abc123.elb.amazonaws.com`)   |
| **Simplify Management** | When backend IP changes, only update the A record — all CNAMEs follow automatically |
| **Hybrid Environments** | Connect on-prem systems or external services under one domain                       |

❌ **Note:** You cannot use a CNAME for the root domain (`example.com`) — only for subdomains. AWS Route 53 provides an **Alias record** as a workaround for that.

---

## ⚙️ A vs AAAA (IPv4 vs IPv6)

| Record Type       | IP Type                        | Who Assigns the IP                        | Example                                      |
| ----------------- | ------------------------------ | ----------------------------------------- | -------------------------------------------- |
| **A**             | IPv4 (e.g., 13.126.45.7)       | You, if using EC2 or static server        | `app.example.com → 13.126.45.7`              |
| **AAAA**          | IPv6 (e.g., 2406:da1a:1234::1) | You, if your hosting supports IPv6        | `app.example.com → 2406:da1a:1234::1`        |
| **Alias / CNAME** | Name → Name                    | AWS or cloud-managed (auto IP resolution) | `app.example.com → abc123.elb.amazonaws.com` |

### 🔹 How it Works in Route 53:

* If you host an **EC2 instance**, you manually add an **A record** using its IP.
* If you use **AWS Load Balancer / CloudFront**, you add an **Alias record** — AWS automatically manages the IP behind the scenes.
* **AAAA** is only needed if you’re using IPv6-enabled resources.

✅ So: you don’t assign IPs manually in Route 53 — you either enter them (for static servers) or let AWS handle it automatically.

---

## 🧠 PTR Records (Reverse DNS)

PTR (Pointer) records map IP → domain name.
Used mainly for:

* Email servers (to verify sender identity)
* Reverse lookups (for debugging or audits)

Example:

```
nslookup 13.126.45.7
→ app.example.com
```

---

## 🧭 Summary Table

| Record | Points To             | Used For                 |
| ------ | --------------------- | ------------------------ |
| A      | IPv4 address          | Normal web servers       |
| AAAA   | IPv6 address          | IPv6-enabled servers     |
| CNAME  | Another domain        | Aliases, cloud mappings  |
| PTR    | Domain name (reverse) | IP verification          |
| MX     | Mail servers          | Email routing            |
| TXT    | Text data             | Verification, SPF, DKIM  |
| NS     | Nameservers           | Delegating DNS authority |

---

## ✅ Key Takeaways

* `.com`, `.net`, `.in` = shared TLDs for everyone.
* The part before `.com` (like `example`) = your unique domain name (purchased by you).
* You can create unlimited subdomains (like `www`, `api`, `dev`) once you own a domain.
* **A record → IPv4**, **AAAA record → IPv6**, **CNAME → name alias**.
* AWS-managed services (ELB, CloudFront) usually use **CNAME** or **Alias** instead of direct A/AAAA records.
* You don’t assign IPs in Route 53 — you only map names to existing IPs or AWS-managed endpoints.

---

**End of Detailed Explanation Version**

---
---

# Network Troubleshooting Basics (Detailed Explanation)

## 🧩 Concept / What

Network troubleshooting in Kubernetes involves diagnosing and fixing communication issues between **Pods**, **Services**, **Nodes**, and **external systems** (like RDS, S3, or Internet).
It’s about finding where connectivity breaks — inside the cluster (Pod, Service, DNS) or outside (VPC, subnet, load balancer).

Common causes:

* **CNI plugin** misconfiguration (e.g., AWS VPC CNI, Calico, Flannel)
* **Service selector** mismatch
* **DNS (CoreDNS)** issues
* **Security Groups / NetworkPolicies** blocking traffic
* **Subnet IP exhaustion** or **VPC route issues**

---

## 🎯 Why / Purpose / Use Case

In a real environment, network troubleshooting is required when:

* Pods fail to connect to other Pods.
* Services are not reachable.
* DNS name resolution fails.
* Internet or AWS resource connectivity breaks.
* AWS LoadBalancer Service shows `<pending>` or no external IP.

✅ Example: “Frontend Pods can’t reach backend Pods” → We trace connectivity through Pod → Service → CoreDNS → AWS.

---

## ⚙️ How to Troubleshoot (Step-by-Step)

### 🔹 1️⃣ Check Pod Connectivity

Run commands **inside Pods** to test communication:

```bash
kubectl exec -it pod1 -- ping pod2
kubectl exec -it pod1 -- curl http://backend-service:8080
```

✅ **If ping fails:**

* Pods might be in different subnets or nodes where CNI is broken.
* The Pod’s ENI might have lost its IP allocation.
* The CNI plugin (like `aws-node`) might be in CrashLoop state.

**Fix:**

* Restart the CNI DaemonSet: `kubectl rollout restart daemonset aws-node -n kube-system`
* Ensure Pods have correct IPs: `kubectl get pods -o wide`

---

### 🔹 2️⃣ Verify DNS Resolution (CoreDNS)

If Pods can’t reach Services by name:

```bash
kubectl run -it dnsutils --image=busybox -- nslookup backend-service
```

✅ **If DNS fails:**

* CoreDNS Pods might not be running.
* `/etc/resolv.conf` in Pods might not point to CoreDNS IP (10.96.0.10).
* CoreDNS ConfigMap might have syntax errors.

**Fix:**

* Check CoreDNS status:

  ```bash
  kubectl get pods -n kube-system -l k8s-app=kube-dns
  ```
* View logs:

  ```bash
  kubectl logs -n kube-system -l k8s-app=kube-dns
  ```
* Restart CoreDNS:

  ```bash
  kubectl rollout restart deployment coredns -n kube-system
  ```

---

### 🔹 3️⃣ Check Service Endpoints

Ensure your Service is properly linked to backend Pods.

```bash
kubectl describe svc backend-service
```

✅ **If `Endpoints:` is empty:**

* The Service selector doesn’t match any Pod labels.
* Pods might be in a different namespace.

**Fix:**

* Match Service selector with Pod labels.
* Verify namespace consistency.

---

### 🔹 4️⃣ Check Network Policies

NetworkPolicies control Pod-to-Pod communication.

```bash
kubectl get networkpolicy
```

✅ **If traffic blocked:**

* A restrictive NetworkPolicy allows only specific labels or namespaces.

**Fix:**

* Temporarily delete it to test: `kubectl delete networkpolicy <name>`
* Update policy with correct ingress/egress rules.

---

### 🔹 5️⃣ Check AWS Layer (EKS Context)

If Pods or Services can’t reach the Internet:

✅ **Possible causes:**

* NAT Gateway missing in private subnet.
* Route Table doesn’t have `0.0.0.0/0` route.
* Security Groups don’t allow outbound traffic.
* Subnet tags missing for EKS LoadBalancers.

**Fix:**

* Add NAT Gateway for private nodes.
* Check VPC → Route Tables → ensure default route exists.
* Add proper subnet tags:

  ```
  kubernetes.io/role/elb = 1 (public)
  kubernetes.io/role/internal-elb = 1 (private)
  ```

If `EXTERNAL-IP` for LoadBalancer stays `<pending>`:

* EKS worker node IAM role missing `ElasticLoadBalancing` permission.
* The subnet used doesn’t have correct tags.

---

### 🔹 6️⃣ Check CNI Plugin (AWS VPC CNI)

The CNI plugin (`aws-node`) manages Pod IPs and ENIs.

```bash
kubectl get pods -n kube-system -l k8s-app=aws-node
kubectl logs -n kube-system -l k8s-app=aws-node
```

✅ **If IP allocation fails:**

* Subnet has no available IPs.
* Node has reached its ENI/IP limit.

**Fix:**

* Add new subnet with a non-overlapping CIDR.
* Or, add a new VPC secondary CIDR and create subnets under it.
* Use bigger instance types (e.g., m5.large) to allow more Pods.

---

### 🔹 7️⃣ Check Node-Level Routes & Interfaces

If Pods on different nodes can’t talk:

```bash
kubectl get nodes -o wide
ip addr
ip route
```

✅ **If route missing or wrong:**

* AWS route table not propagating Pod CIDRs.
* CNI failed to attach routes for new ENIs.

**Fix:** Restart `aws-node` and verify subnet/VPC routes.

---

### 🔹 8️⃣ Use `traceroute` or `mtr`

To trace where packets stop:

```bash
kubectl exec -it pod1 -- traceroute backend-service
```

✅ This shows if the drop happens inside Kubernetes (Pod/Service/DNS) or outside (VPC/NAT).

---

## ⚠️ Common Issues / Errors (with Expanded Explanation)

| Issue                           | Description                                                                                              | Fix                                                                        |
| ------------------------------- | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| **Pods can’t reach each other** | Often caused by CNI issues, misconfigured routing tables, or NetworkPolicies blocking inter-Pod traffic. | Restart CNI plugin, check routes, verify no restrictive NetworkPolicies.   |
| **DNS resolution fails**        | CoreDNS not running, wrong ConfigMap, or bad `/etc/resolv.conf` inside Pods.                             | Restart CoreDNS, fix config, or redeploy default ConfigMap.                |
| **Service not reachable**       | Usually because Service selector doesn’t match Pod labels or endpoints are missing.                      | Match labels correctly and verify Pod readiness.                           |
| **LoadBalancer `<pending>`**    | Missing IAM permissions, untagged subnets, or AWS LoadBalancer Controller misconfigured.                 | Tag subnets properly, attach correct IAM policy, and verify LB controller. |
| **External access blocked**     | Missing NAT Gateway, incorrect route table, or Security Group restrictions.                              | Add NAT/IGW route, verify SG outbound rules.                               |
| **IP exhaustion**               | Subnet’s CIDR fully used, so no new Pods can get IPs.                                                    | Add new subnet or add secondary CIDR to VPC.                               |

---

## 🧰 Useful Commands Summary

| Purpose                | Command                                                      |
| ---------------------- | ------------------------------------------------------------ |
| List Pods with IPs     | `kubectl get pods -o wide`                                   |
| Describe Service       | `kubectl describe svc <svc>`                                 |
| Test DNS               | `kubectl run -it dnsutils --image=busybox -- nslookup <svc>` |
| View CoreDNS logs      | `kubectl logs -n kube-system -l k8s-app=kube-dns`            |
| Check CNI Pods         | `kubectl get pods -n kube-system -l k8s-app=aws-node`        |
| View node routes       | `ip route`                                                   |
| Check network policies | `kubectl get networkpolicy`                                  |

---

## 🧠 Best Practices

✅ Start troubleshooting **inside-out**: Pod → Service → CoreDNS → CNI → AWS layer.
✅ Always verify **Pod labels vs Service selectors** first.
✅ Run at least **2 CoreDNS replicas** for HA.
✅ Tag AWS subnets properly for ELB and AutoScaler use.
✅ Use **NodeLocal DNSCache** for faster DNS resolution in large clusters.
✅ Monitor Pod IP usage via CloudWatch or EKS metrics.
✅ Regularly upgrade CNI plugin to avoid IP allocation bugs.

---

## 🧭 In Simple Words

> Kubernetes network troubleshooting means tracking how traffic flows from Pod → Service → DNS → Node → AWS network. Identify where it fails — DNS misconfig, CNI issues, Security Groups, or IP exhaustion — and fix accordingly.

---

**End of Detailed Explanation Version**

---
---
---
