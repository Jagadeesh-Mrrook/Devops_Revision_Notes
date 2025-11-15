# Ingress & Ingress Controllers — Detailed Explanation Notes

## **Concept / What**

Ingress is a Kubernetes object used to expose HTTP/HTTPS applications from inside the cluster to external users. It provides Layer 7 routing rules such as host-based and path-based routing. An Ingress Controller implements these rules and configures the underlying load balancer or reverse proxy (e.g., AWS ALB Load Balancer Controller).

---

## **Why / Purpose / Use Case in Real‑World**

* Expose multiple microservices using a **single external load balancer**.
* Save cost by avoiding separate LoadBalancers for each service.
* Central place for **TLS/SSL termination**, routing rules, rewrites, and traffic control.
* Useful for domain‑based routing (`api.example.com`, `app.example.com`).
* Useful for path‑based routing (`/api`, `/ui`).
* Integrated with AWS ALB for production‑grade L7 routing.

---

## **How It Works / Steps / Syntax**

1. Deploy the **Ingress Controller** (on AWS, this is the AWS Load Balancer Controller).
2. Create an **Ingress** YAML manifest defining host/path routing rules.
3. Controller watches the Ingress object and creates an external **ALB** with listeners and target groups.
4. ALB forwards traffic to backend Kubernetes Services.

### **Sample Ingress Manifest**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-ingress
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

### **Important Fields Explanation**

* **ingress.class**: tells Kubernetes to use AWS ALB Ingress Controller.
* **host**: domain exposed to the user.
* **pathType**: how the path is matched (Prefix/Exact).
* **backend.service**: target Kubernetes Service.

---

## **Common Issues / Errors**

* ALB not created (missing annotations or RBAC/IRSA issues).
* 503 errors due to failed target group health checks.
* Wrong service port mappings causing backend failure.
* DNS not pointing to ALB.
* SSL errors due to invalid/wrong certificate.
* Incorrect path rules leading to 404 responses.

---

## **Troubleshooting / Fixes**

* Use `kubectl describe ingress` to check events and errors.
* Verify ALB target group health checks in AWS console.
* Ensure service ports match container ports.
* Validate correctness of Route53 DNS mappings.
* Check controller logs: `kubectl logs -n kube-system deployment/aws-load-balancer-controller`.
* Verify ACM certificate region and validity.

---

## **Best Practices / Tips**

* Use host-based routing for multi-domain microservices.
* Use path-based routing for UI/API split.
* Always terminate TLS/SSL at ALB using ACM.
* Avoid unnecessary annotations—keep configuration minimal.
* Use readiness/liveness probes for healthy backend routing.
* Use least privilege IAM for the AWS Load Balancer Controller.

---
---

# AWS ALB Ingress Controller (AWS Load Balancer Controller) — Detailed Explanation Notes

## **Concept / What**

The AWS Load Balancer Controller is a Kubernetes controller that manages AWS load balancers for applications running on EKS. It:

* Creates **Application Load Balancers (ALB)** when an **Ingress** (or Gateway API object) is defined.
* Creates **Network Load Balancers (NLB)** when a **Service of type LoadBalancer** is defined.

It continuously watches Kubernetes resources and provisions the required AWS components such as ALBs, NLBs, listeners, listener rules, target groups, security groups, and health checks.

---

## **Why / Purpose / Use Case in Real-World**

* Automates the creation and management of ALB/NLB on EKS.
* Automatically configures routing rules, certificates, and health checks based on Kubernetes manifests.
* Reduces manual AWS configuration (no need to create ALB or NLB manually in the AWS console).
* Provides production-grade L7/L4 load balancing for microservices hosted on Kubernetes.
* Enables advanced ALB features like host-path routing, SSL termination, WAF integration, stickiness, and IP-target mode.

**Typical Use Cases:**

* Exposing multiple microservices using a single ALB with path/host-based routing.
* Creating internal or internet-facing ALBs automatically.
* Running modern NLBs with IP-target support using Service type LoadBalancer.
* Managing SSL certificates with ACM through Ingress annotations.

---

## **How It Works / Steps / Syntax**

### **1. Install the AWS Load Balancer Controller**

* Installed via Helm chart.
* Requires IAM role via IRSA for AWS permissions.
* Runs in the `kube-system` namespace.

### **2. Two Separate Functions Inside the Controller**

| Function        | Trigger                   | Creates                         |
| --------------- | ------------------------- | ------------------------------- |
| **ALB Manager** | Ingress object            | Application Load Balancer (ALB) |
| **NLB Manager** | Service type=LoadBalancer | Network Load Balancer (NLB)     |

### **3. ALB Creation (via Ingress)**

* Define annotations inside the **Ingress** object.
* Controller creates ALB, listeners, rules, health checks, SGs.

**Sample ALB Ingress Manifest:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: javaapp-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - host: java.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: java-service
            port:
              number: 80
```

### **4. NLB Creation (via Service)**

* Define annotations inside a **Service type LoadBalancer**.
* Controller creates modern NLB with IP targets and advanced features.

**Sample NLB Service Manifest:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nlb-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb-ip"
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: myapp
```

---

## **Common Issues / Errors**

### **Ingress/ALB Related:**

* ALB not created due to missing `ingress.class: alb`.
* IRSA misconfiguration causing permission errors.
* 503 on ALB due to failing target group health checks.
* Wrong service port mapping.
* SSL configuration issues due to wrong/invalid ACM certificate.

### **NLB Related:**

* Fallback to legacy NLB if Load Balancer Controller is not installed.
* Incorrect NLB annotations ignored due to wrong syntax.
* Health checks failing due to port mismatch.

---

## **Troubleshooting / Fixes**

* `kubectl describe ingress <name>` to check ALB event errors.
* `kubectl describe service <name>` to inspect NLB annotations and events.
* Check logs of the controller:

  ```
  kubectl logs -n kube-system deployment/aws-load-balancer-controller
  ```
* Validate ALB/NLB target groups and health checks in AWS console.
* Verify DNS mapping in Route53.
* Ensure correct IAM permissions for AWS Load Balancer Controller.

---

## **Best Practices / Tips**

* Use **Ingress only for ALB** (HTTP/HTTPS routing).
* Use **Service type LoadBalancer only for NLB** (L4 routing).
* Keep annotations minimal and correct.
* Always use **IP target mode** for modern NLB.
* Use ACM for SSL certificates with ALB.
* Validate proper IAM permissions using IRSA.
* Use readiness/liveness probes for healthy targeting.

---
---

# Path-Based & Host-Based Routing — Detailed Explanation Notes

## **Concept / What**

Path-based and host-based routing are Layer 7 (HTTP/HTTPS) routing techniques used by Application Load Balancers (ALB) through Kubernetes Ingress. They allow multiple backend services to be exposed using a single ALB by matching either the URL path or the hostname.

---

## **Why / Purpose / Use Case in Real-World**

### **Path-Based Routing:**

* Separate multiple microservices under a single domain.
* Useful for frontend + backend separation (`/ui`, `/api`).
* Supports versioning (`/v1`, `/v2`).
* Reduces cost by avoiding multiple LoadBalancers.

### **Host-Based Routing:**

* Route traffic based on different domains or subdomains.
* Useful for clearly separated applications (`api.example.com`, `app.example.com`).
* Ideal for multi-tenant SaaS (`tenant1.company.com`, `tenant2.company.com`).
* Better cookie, authentication, and CORS isolation.

---

## **How It Works / Steps / Syntax**

### **1. Path-Based Routing**

* ALB examines the URL path after the domain.
* Matches the path to the correct backend service.

**Request Flow:**
User → ALB → Path Rule → Target Group → Kubernetes Service → Pods

**Example:** `/api` → API service, `/ui` → UI service

**Manifest Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /ui
        pathType: Prefix
        backend:
          service:
            name: ui-service
            port:
              number: 80
```

---

### **2. Host-Based Routing**

* ALB checks the Host header in the request.
* Routes based on domain/subdomain.

**Request Flow:**
User → DNS → ALB → Host Rule → Target Group → Kubernetes Service → Pods

**Example:** `api.example.com` → API, `app.example.com` → UI

**Manifest Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ui-service
            port:
              number: 80
```

---

## **Common Issues / Errors**

### Path-Based:

* 404 errors due to incorrect Prefix/Exact pathType.
* Wrong or overlapping paths causing routing mismatch.
* Backend service unavailable leading to 503.

### Host-Based:

* DNS not mapped correctly to ALB.
* Missing ACM certificate when using HTTPS.
* Wrong hostnames defined in the Ingress manifest.
* CORS issues due to different domains.

---

## **Troubleshooting / Fixes**

* `kubectl describe ingress <name>` to verify rules and errors.
* Check ALB listener rules in AWS console.
* Validate DNS routing with `nslookup`.
* Ensure correct ACM certificates when using TLS.
* Confirm target group health checks in ALB.

---

## **Best Practices / Tips**

* Use **path-based** routing for simple frontend-backend separation.
* Use **host-based** routing for multi-domain or microservice isolation.
* Always use `PathType: Prefix` unless you need Exact matching.
* Avoid ambiguous paths like `/app` vs `/app/` which may confuse rule matching.
* Map all subdomains correctly in Route53.
* Ensure frontend developers correctly map UI buttons → backend paths.

---
---

# End-to-End Request Flow: User → DNS → CloudFront → S3/ALB → EKS Pods → Back to User

---

## **Concept / What**

This document explains the complete flow of how a user's request travels through:

* Browser & DNS resolution
* CloudFront
* S3 (static content)
* ALB (dynamic/API traffic)
* Kubernetes Ingress
* EKS Services & Pods
* And returns back to the user

This is a complete, production-level sequence used in modern microservices applications.

---

## **Why / Purpose**

Understanding this flow helps in interviews and real-world troubleshooting:

* How DNS works with Route53
* How CloudFront improves latency and caching
* How ALB handles L7 routing (path/host-based)
* How EKS services route traffic to Pods
* How static and dynamic content is separated

---

## **How the Request Flows (Step-by-Step)**

### **1️⃣ User Enters the Website URL**

Example:

```
www.example.com
```

User types this OR clicks the link from Google search.

---

### **2️⃣ Browser Triggers DNS Resolution**

Browser → OS Resolver → ISP DNS / Google DNS / Cloudflare DNS.

These DNS resolvers eventually query **Route53**, because Route53 is the **authoritative DNS server** for the domain.

**Route53 returns:**

```
d1234abcd.cloudfront.net
```

(CloudFront distribution domain)

---

### **3️⃣ Browser Connects to CloudFront (Nearest Edge Location)**

CloudFront:

* Performs SSL handshake
* Serves cached static content
* Forwards dynamic/API traffic to the origin (ALB)

---

### **4️⃣ CloudFront Decides the Origin Based on the Request**

CloudFront uses origin rules such as:

* `/*` → S3 Bucket (HTML, CSS, JS, Images)
* `/api/*` → ALB (backend applications)

---

### **5️⃣ CloudFront → Forwards Dynamic Requests to ALB**

For all **API or backend** requests:

```
Origin = <ALB-DNS-Name>
```

CloudFront adds headers like:

* `X-Forwarded-For`
* `Host`

---

### **6️⃣ ALB Receives the Request**

Application Load Balancer performs Layer 7 routing using rules defined in **Ingress**:

* Path-based routing (`/api`, `/cart`, `/payment`)
* Host-based routing (`api.example.com`, `app.example.com`)

---

### **7️⃣ ALB Selects the Correct Target Group**

ALB checks rule → picks correct target group:

* `tg-api`
* `tg-cart`
* `tg-payment`

---

### **8️⃣ Target Group Forwards Traffic to Pod IPs (IP Mode)**

ALB sends the request directly to **Pod IPs** through the worker node ENIs using:

* IP Target Mode (recommended)

---

### **9️⃣ Kubernetes Node Forwards to the Right Pod**

`kube-proxy` routes traffic from node → correct Pod matching the Service selector.

---

### **1️⃣0️⃣ Pod Processes the Request**

Backend container (Java, Python, NodeJS, Go, etc.) handles:

* Authentication
* Database queries
* Business logic

---

### **1️⃣1️⃣ Pod Sends Response Back**

Response goes back through:
Pod → Node → ALB → CloudFront → Browser

CloudFront may cache the response if allowed.

---

## **Common Issues / Errors**

* Wrong DNS records in Route53
* Incorrect CloudFront origin configuration
* ALB listener rules mismatching paths
* Ingress path or host rules incorrect
* Service port/targetPort mismatch
* Pod not Ready causing 503 errors

---

## **Troubleshooting Tips**

* `nslookup` or `dig` to check DNS
* CloudFront cache invalidation for updated UI files
* Check ALB listener rules in AWS Console
* `kubectl describe ingress` for routing issues
* `kubectl get endpoints` for Service-Pod mapping
* Check Pod readiness/liveness probes

---

## **Best Practices**

* Use CloudFront for global caching and low latency
* Serve static UI from S3
* Route all API traffic through ALB
* Use IP target mode for ALB → Pods
* Use proper TTLs in Route53
* Use HTTPS everywhere (ACM certificates)
* Keep clear path rules in Ingress

---
---

# Service Mesh — Detailed Explanation Notes

## **Concept / What**

A **Service Mesh** is a dedicated infrastructure layer inside Kubernetes that manages **service-to-service communication** using distributed **sidecar proxies** (typically Envoy). It offloads communication features (security, routing, observability, retries, etc.) from the application code to the mesh.

The application focuses only on business logic, while the mesh handles reliability, security, and traffic control.

---

## **Why / Purpose / Use Case in Real-World**

### **1. mTLS (Mutual TLS)**

* Encrypts all internal Pod-to-Pod traffic.
* Provides identity-based authentication between services.
* Enables Zero Trust networking inside the cluster.

### **2. Advanced Traffic Management**

* Canary deployments
* Blue/Green deployments
* Weighted routing (e.g., 10% → 50% → 100%)
* A/B testing
* Fault injection

All without changing application code.

### **3. Observability & Monitoring**

Automatically generates:

* Request metrics (latency, errors, throughput)
* Distributed tracing
* Service dependency graphs
* Access logs

### **4. Reliability Features**

* Automatic retries
* Timeouts
* Circuit breakers
* Rate limiting
* Load balancing between services

### **5. Security Policies**

* Allow/deny rules between services
* Restrict which microservices can talk to which
* Helps enforce Zero Trust architecture

---

## **How It Works / Steps / Syntax**

### **1. Sidecar Proxy Injection**

Each Pod gets a sidecar proxy container (Envoy). All traffic flows through it:

```
Pod A → Envoy A → Envoy B → Pod B
```

### **2. Control Plane**

Service mesh control plane (e.g., Istio Pilot, AWS App Mesh Controller) configures:

* Routing rules
* mTLS policies
* Retry/timeouts
* Canary weights

### **3. Data Plane**

Sidecar proxies apply the policies and forward traffic accordingly.

---

## **Common Tools / Technologies**

* **Istio** (most popular)
* **Linkerd** (lightweight)
* **AWS App Mesh** (AWS-native)
* **Consul Connect**

---

## **Common Issues / Errors**

* High latency due to sidecar overhead.
* Resource consumption increases (CPU/memory for each sidecar).
* Misconfigured mTLS causing service-to-service failures.
* Complex debugging when sidecar rules conflict.
* Hard migrations for legacy applications.

---

## **Troubleshooting / Fixes**

* Check sidecar logs (Envoy logs) for routing or mTLS issues.
* Validate mesh policies using CLI tools (e.g., `istioctl`, `linkerd check`).
* Ensure correct certificate rotation for mTLS.
* Verify sidecar injection is enabled for the namespace.
* Use tracing tools (Jaeger/Zipkin) to find communication failures.

---

## **Best Practices / Tips**

* Use a mesh only when microservices are large and communication is complex.
* Avoid using mesh for small clusters due to overhead.
* Enable mTLS cluster-wide for Zero Trust security.
* Use traffic shifting for safe canary deployments.
* Monitor mesh resource usage (sidecars consume CPU/memory).
* Document mesh policies clearly to avoid accidental blocks.

---
---

# Multi-Service Routing — Detailed Explanation Notes

## **Concept / What**

**Multi-Service Routing** is the practice of using a **single Application Load Balancer (ALB)** to route traffic to **multiple backend Kubernetes services** based on rules such as:

* URL paths (`/api`, `/cart`, `/payment`)
* Hostnames (`api.example.com`, `app.example.com`)
* Combined host + path conditions

In EKS, multi-service routing is implemented through **Ingress** and the **AWS Load Balancer Controller**, which automatically creates ALB listener rules and target groups.

---

## **Why / Purpose / Use Case in Real-World**

* **Cost optimization:** One ALB can serve many microservices.
* **Unified entry point:** Centralized routing, logging, and monitoring.
* **Microservices architecture:** Each service gets its own path or domain.
* **Frontend + backend split:** `/ → UI`, `/api → backend`.
* **Versioning:** `/v1`, `/v2`, etc.
* **High scalability:** Adding new services only requires updating Ingress.

Without this setup, Kubernetes would create **one NLB/CLB per service**, which becomes costly and complex.

---

## **How It Works / Steps / Syntax**

### **1. Ingress Resource Defines Routing Rules**

The Ingress object defines:

* Path-based routing
* Host-based routing
* TLS
* Target services

### **2. AWS Load Balancer Controller Reads Ingress**

The controller creates:

* One ALB
* Listener rules
* Target groups
* Health checks

### **3. Single ALB → Multiple Services**

The ALB forwards traffic to different backend services based on routing logic defined in Ingress.

---

## **Manifest Examples**

### **Path-Based Multi-Service Routing**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing-ingress
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

      - path: /cart
        pathType: Prefix
        backend:
          service:
            name: cart-service
            port:
              number: 80

      - path: /payment
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 80
```

### **Host-Based Multi-Service Routing**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing-ingress
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### **Combined Host + Path Routing**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: combined-routing
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 80

      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 80
```

---

## **Common Issues / Errors**

* **Multiple conflicting rules** (overlapping paths/hosts).
* **Missing `ingress.class: alb` annotation** → ALB not created.
* **Service port mismatch** between Ingress and Service.
* **Unhealthy target groups** due to failing readiness probes.
* **Incorrect DNS pointing to ALB DNS name**.

---

## **Troubleshooting / Fixes**

* Check Ingress events:

  ```
  kubectl describe ingress <name>
  ```
* Validate ALB listener rules in the AWS console.
* Check target group health in EC2 → Target Groups.
* Verify Route53 DNS mapping.
* Check logs of AWS Load Balancer Controller:

  ```
  kubectl logs -n kube-system deployment/aws-load-balancer-controller
  ```

---

## **Best Practices / Tips**

* Use **Prefix** pathType for most microservices.
* Prefer **host-based routing** for large systems.
* Keep path names clear and simple.
* Use readiness probes for stable ALB health.
* Ensure all backend services are **ClusterIP** for ALB routing.
* Maintain a single Ingress file per application for easier management.

---
---

# Common Ingress Errors & Troubleshooting — Detailed Notes

## **Concept / What**

Common Ingress errors occur when the Kubernetes Ingress resource, AWS Load Balancer Controller, ALB configuration, or DNS setup is misconfigured. These errors lead to issues like:

* ALB not getting created
* 503 Service Unavailable
* 404 errors
* Incorrect routing to backend services
* SSL/TLS failures
* DNS not resolving

This section explains real-world Ingress issues and how to troubleshoot them step-by-step.

---

## **Why / Purpose / Use Case in Real-World**

Ingress-related issues are extremely common in production EKS environments. When customers complain about:

* Application downtime
* Incorrect paths
* APIs not working
* HTTPS failures
* Wrong service routing
* ALB health check failures

…it almost always relates to Ingress. Understanding these problems and fixing them quickly is a core DevOps responsibility.

---

## **How It Works / Errors, Causes & Fixes**

### **1. ALB Not Created**

**Causes:**

* Missing annotation `kubernetes.io/ingress.class: alb`
* AWS Load Balancer Controller not installed or failing
* Wrong IAM permissions for the controller
* Wrong `clusterName` during controller installation
* Ingress YAML syntax errors

**Fix:**

* Run `kubectl describe ingress <name>` and check Events
* Check controller logs:

  ```
  kubectl logs -n kube-system deployment/aws-load-balancer-controller
  ```
* Fix IAM or annotations accordingly

---

### **2. 503 Service Unavailable**

**Causes:**

* ALB cannot reach backend Pods
* Service → Pod selector mismatch
* Pods not Ready
* Wrong port/targetPort mapping
* Health checks failing
* No endpoints for the Service

**Fix:**

* Check endpoints:

  ```
  kubectl get endpoints <service>
  ```
* Check pod readiness:

  ```
  kubectl describe pod <pod>
  ```
* Validate target group health in AWS

---

### **3. 404 Not Found**

**Causes:**

* Wrong path in Ingress
* Incorrect `pathType`
* Overlapping paths (e.g., `/` catching everything before `/api`)
* Wrong hostname

**Fix:**

* Describe Ingress:

  ```
  kubectl describe ingress <name>
  ```
* Validate ALB listener rules in AWS
* Ensure longest-prefix paths are placed correctly

---

### **4. ALB Created but No Rules Added**

**Causes:**

* Empty `spec.rules` in Ingress
* YAML indentation or missing backend config

**Fix:**

* Correct YAML structure and reapply

---

### **5. Traffic Routed to Wrong Service**

**Causes:**

* Overlapping paths
* Generic path like `/` defined before `/api`
* Incorrect rule priorities

**Fix:**

* Remember ALB routing order:

  1. Host rules
  2. Longest path prefix
  3. Default rule

---

### **6. SSL/TLS Errors**

**Causes:**

* Wrong ACM certificate ARN
* Certificate in different region
* Missing HTTPS listener
* Certificate CN/SAN mismatch

**Fix:**

* Check Ingress TLS annotations
* Confirm ALB listener and certificate in AWS console

---

### **7. DNS Not Resolving**

**Causes:**

* Route53 A-record not pointing to ALB DNS
* Old ALB recreated but DNS still pointing to previous one
* Wrong hosted zone

**Fix:**

* Verify using:

  ```
  nslookup example.com
  ```
* Update Route53 mapping

---

### **8. Load Balancer Controller CrashLoopBackOff**

**Causes:**

* Wrong IRSA configuration
* IAM policy missing permissions
* Wrong region or clusterName

**Fix:**

* Check controller logs and fix IAM bindings

---

### **9. Conflicts with Other Ingress Controllers**

**Causes:**

* NGINX Ingress or Gateway API controller also installed
* Ingress not tagged with correct class

**Fix:**

* Add explicitly:

  ```
  kubernetes.io/ingress.class: alb
  ```

---

### **10. Node Security Group Blocks ALB Traffic**

**Causes:**

* Worker node SG does not allow inbound from ALB SG

**Fix:**

* Add inbound rule:

  * Source: ALB Security Group
  * Destination: Node Security Group
  * Port: targetPort

---

## **Troubleshooting Checklist**

### **1. Check Ingress events**

```bash
kubectl describe ingress <name>
```

### **2. Check ALB in AWS Console**

* Listener rules
* Target groups
* Health checks

### **3. Check Service → Pod mapping**

```bash
kubectl get endpoints <service>
```

### **4. Check Pods**

```bash
kubectl describe pod <pod>
```

### **5. Check Controller Logs**

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

### **6. Check DNS**

```bash
nslookup example.com
```

---

## **Best Practices / Tips**

* Always use `Prefix` pathType unless Exact is needed.
* Keep paths non-overlapping and predictable.
* Ensure backend services are **ClusterIP** when using ALB.
* Use readiness/liveness probes to maintain healthy targets.
* Keep IAM permissions updated for the controller.
* Explicitly set `ingress.class: alb` to avoid conflicts.
* Validate certificates in ACM and ensure correct region.

---
---


