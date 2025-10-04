# Application Server / Backend Layer - Quick Revision

## Concept / What

* Backend server running **business logic**
* Generates **dynamic content**, connects **DB and third-party APIs**, exposes **REST APIs**
* Supports **microservices** in multiple languages (Java, Node.js, Python)

## Purpose / Use Case

* Bridges **frontend and data layer**
* Handles **scalable, stateless business logic**
* Integrates with **third-party services** (payment, messaging, APIs)

## How it Works / Steps

1. Client sends REST API request → App Server → DB / 3rd-party API → Response
2. **Security**: JWT for stateless auth, OAuth for delegated access, Sticky Sessions optional
3. **Scaling**: ALB/NLB + Auto Scaling, any pod can validate JWT

## Common Issues

* 500 errors, high latency, session issues, scaling failures, security misconfigurations

## Fixes / Tips

* Monitor logs, metrics, alerts
* Auto-scaling, retries, circuit breakers
* Rotate JWT secrets, secure OAuth keys
* Use HTTPS, WAF, IAM, rate-limiting
* Prefer stateless JWT over sticky sessions for scalability

## AWS/K8s Example

* Backend pods behind ALB Ingress, HPA for scaling
* JWT signed with secret from AWS Secrets Manager
* Third-party API calls with retries
* Monitoring via CloudWatch / Prometheus / Grafana

---

# REST API Quick Revision Notes - DevOps

## Concept

* API enables software components to communicate.
* REST APIs: HTTP-based, stateless, resource-oriented between frontend & backend.
* Backend accesses DB internally; frontend uses REST endpoints.

## Purpose / Use Case

* Frontend ↔ backend communication.
* Stateless → supports scaling & microservices.
* Resource-oriented URLs (`/users/123`) → clean, cacheable, intuitive.

## Key Points / Flow

```
Client → API Gateway/ALB → Backend Pod → DB/3rd-party → Response
```

* Authenticate (JWT/OAuth)
* Validate request
* Call business logic / microservice
* Return JSON/XML response

## API Types

* REST: lightweight, stateless, JSON/XML, preferred for frontend-backend.
* SOAP: XML, enterprise apps.
* GraphQL: flexible queries, single endpoint.
* gRPC/Thrift: fast internal microservices communication.

## Resource-Oriented URLs

* `/users/123` instead of `/getUser?id=123`
* HTTP verbs define action (GET/PUT/DELETE)
* Nested resources: `/users/123/orders`
* Benefits: scalable, cacheable, maintainable.

## Common Issues

* CORS errors, wrong HTTP methods, invalid headers
* Versioning inconsistencies
* Misconfigured load balancers / gateways

## Troubleshooting / Fixes

* Swagger/OpenAPI docs
* Correct CORS headers
* Validate payloads, handle errors consistently
* Monitor latency, errors via CloudWatch/Prometheus
* Apply WAF and rate limiting

## Best Practices (DevOps)

* Use resource-oriented URLs
* Stateless APIs → any pod can serve requests
* JWT/OAuth for auth
* Monitor availability & latency
* Keep backend → DB internal; frontend uses API

## Cloud Example

* EKS pods behind ALB
* REST endpoints via API Gateway
* JWT/OAuth for auth
* Backend → RDS/DynamoDB internally
* CloudWatch for API monitoring


---

# Dynamic Content Generation - Quick Revision Notes (DevOps)

## Concept

* Backend generates **dynamic content** on-the-fly based on user requests.
* Content fetched from **DB or 3rd-party API**, processed via **business logic**, sent as JSON/XML.
* Frontend parses JSON and renders on UI.

## Key Flow

```
Client → ALB/API Gateway → Backend Pod → DB/3rd-party → JSON Response → Frontend UI
```

* Steps: Authenticate → Fetch Data → Apply Logic → Send Response

## Techniques

* **Server-Side Rendering (SSR)**: HTML generated server-side.
* **API-driven**: JSON response rendered by frontend.
* **Template Engines**: EJS, Thymeleaf, Handlebars.

## Common Issues

* High latency, backend overload
* Wrong user-specific data, session/auth errors
* Template or rendering errors

## Troubleshooting / Fixes

* Optimize DB queries & caching
* Load balancing and auto-scaling
* Validate data before rendering
* Use CDNs / edge caching for partial caching
* Monitor performance (CloudWatch/Prometheus)

## Best Practices (DevOps)

* Separate business logic & presentation
* Cache frequently accessed data
* API-driven content for microservices
* Secure user-specific data (JWT/OAuth)
* Monitor backend performance

## Cloud Example

* EKS/EC2 pods serve dynamic content
* ALB distributes requests
* Data from RDS/DynamoDB or 3rd-party
* Partial caching via ElastiCache
* Frontend consumes JSON and renders UI
* CloudWatch monitors latency and errors

---

# Backend Connection with Frontend (API Endpoints) - Quick Revision Notes

## Concept / What

* Backend exposes REST APIs; frontend consumes them.
* Requests go through **ALB** to backend instances/pods.
* Frontend does not expose REST APIs externally.

## Flow

```
Frontend → ALB → Backend port → REST API → DB/3rd-party → ALB → Frontend UI
```

## Key Points

* Port: network-level entry to backend service
* REST API: application-level request handler
* ALB: routes requests, SSL termination, WAF, health checks
* Stateless endpoints enable scaling
* JWT/OAuth for authentication
* API versioning prevents breaking frontend

## Common Issues

* 401 Unauthorized (token issues)
* 400 Bad Request (missing headers/payload)
* 5xx errors if backend overloaded
* CORS errors (cross-domain requests)
* Misconfigured ALB or unhealthy targets

## DevOps Tips

* Monitor API performance, errors, and traffic
* Apply rate limiting / WAF
* Use path-based routing and health checks
* Keep endpoints stateless and secure

---

# Load Balancing (ALB, NLB) and Session Handling - Quick Revision Notes

## Concept / What

* Load balancers distribute traffic across backend servers/pods.
* Session handling maintains user state across requests.
* Supports scalable, highly available applications.

## Key Components

* **ALB (Layer 7)**: HTTP/HTTPS, path-based routing, WAF, SSL termination.
* **NLB (Layer 4)**: TCP/UDP, ultra-low latency, high throughput.

## Session Management Types

1. **Sticky Sessions**: ALB binds user to one pod via cookie (AWSALB).
2. **Centralized Store**: Redis/ElastiCache stores session → any pod can access.
3. **JWT / Stateless**: App issues token → client stores → any pod validates.

## Flow Example (JWT)

```
User → ALB → Backend pod → Auth → JWT issued → Client stores JWT
→ Next requests carry JWT → ALB forwards → any pod validates JWT → Response
```

## Common Issues

* Sticky session server failure → session lost.
* Overloaded pod if many users stick.
* Redis bottleneck if unscaled.
* JWT secret leak → security risk.
* Misconfigured ALB target groups.

## Best Practices

* Prefer JWT/stateless sessions for cloud-native apps.
* Use centralized sessions (Redis) if needed.
* Enable WAF + throttling.
* Keep endpoints stateless.
* Store JWT secrets securely in Secrets Manager/KMS.
* Monitor ALB, pods, session store with CloudWatch.

## DevOps Role

* Configure ALB stickiness.
* Deploy/manage Redis.
* Manage JWT secrets.
* Ensure scaling policies and monitoring are effective.

---

# Auto Scaling of Backend Pods / EC2 Instances - Quick Revision Notes

## Concept / What

* Auto Scaling adjusts EC2 instances or Kubernetes pods automatically based on demand.
* Ensures high availability, performance, and cost efficiency.

## Key Components

* **EC2**: Auto Scaling Group (ASG) + Launch Template.
* **Kubernetes**: HPA (Horizontal Pod Autoscaler), VPA (Vertical Pod Autoscaler), Cluster Autoscaler.
* **Launch Template** preferred over Launch Configuration (supports versioning, overrides).

## Scaling Triggers

* Target tracking (CPU/memory utilization)
* Step scaling (add/remove instances based on thresholds)
* Scheduled scaling (time-based)

## Common Issues

* Over-scaling → high cost
* Under-scaling → latency / 5xx errors
* Cooldown period misconfigurations
* Cold start latency

## Best Practices

* Set min/max limits to control cost.
* Use target-tracking policies.
* Combine HPA + VPA + Cluster Autoscaler in K8s.
* Ensure ALB health checks are accurate.
* Monitor metrics and alerts.
* Schedule scaling for predictable spikes.

## DevOps Role

* Create/manage Launch Templates.
* Define scaling policies and triggers.
* Deploy and monitor HPA/VPA/Cluster Autoscaler.
* Test scaling in staging before production.
* Optimize cost and performance continuously.

---

# Security: WAF, IAM Roles, API Throttling - Quick Revision Notes

## Concept / What

* Security protects backend servers from unauthorized access, attacks, and misuse.
* Focus: Access control, request filtering, traffic management.

## Key Components

* **AWS Shield** → network/transport-level DDoS protection (Layer 3/4) for ALB & NLB.
* **AWS WAF** → application-level firewall (Layer 7) for ALB, protects against SQLi, XSS, malicious bots, HTTP floods.
* **IAM Roles** → access control for EC2, pods, Lambda without hardcoding credentials.
* **API Throttling** → limits requests per client per second to prevent abuse.

## Flow

```
User → AWS Shield → WAF → ALB → Backend
```

* NLB uses **Shield only**, WAF not applicable.

## Common Issues

* WAF misconfigured → blocks legit traffic.
* IAM missing permissions → 403 errors.
* API throttling too strict → 429 errors.
* Large DDoS → need Shield.

## Best Practices

* Use least privilege IAM roles.
* Apply WAF managed rulesets.
* Implement API throttling.
* Monitor with CloudWatch/CloudTrail.
* Test in staging.
* Shield + WAF together for ALB.
* Shield only for NLB.

## Common Attacks (One-line)

* **DDoS** → server overload.
* **SQL Injection** → malicious DB queries.
* **XSS** → steal user sessions.
* **Malicious Bots** → automated abuse.

## DevOps Role

* Configure Shield & WAF.
* Assign/manage IAM roles.
* Implement API throttling.
* Monitor logs & metrics.
* Ensure proper protection for ALB/NLB.

---

# Application Server: Common Issues & Troubleshooting - Quick Revision Notes

## Key Issues & Fixes

### 500 Internal Server Errors

* **Cause:** Code bugs, DB failures, env misconfig, low memory/CPU.
* **Fix:** Check logs, validate DB, review code, scale resources.

### High Latency / Slow Responses

* **Cause:** Heavy DB queries, blocking APIs, resource/network limits.
* **Fix:** Profile queries, caching (Redis), async calls, scale horizontally, monitor metrics.

### Scaling Failures

* **Cause:** Misconfigured Auto Scaling/HPA, dependency bottlenecks.
* **Fix:** Validate scaling triggers, check HPA logs, ensure stateless backend, optimize thresholds.

### Other Common Issues

* Auth/Authorization errors (JWT, IAM roles).
* Connection timeouts (DB or external services).
* Memory leaks/crashes.

## Best Practices

* Centralize logs and metrics.
* Implement health checks.
* Keep backend stateless.
* Monitor resources and app performance.
* Automate alerts for errors/spikes.
* Use caching & async calls.
* Validate scaling configurations regularly.

