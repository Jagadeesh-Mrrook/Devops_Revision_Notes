## 🚀 Quick Revision – Request & Data Flow

* **Flow:**
  User → Browser/App → DNS Resolution → CDN (for static) / ALB (for dynamic) → WAF → AWS Shield → Target Group → EC2/App Server → Database.

* **CDN (CloudFront):**
  Delivers static content (HTML, CSS, JS, images) from edge locations near users.

* **Dynamic Content:**
  Routed via Application Load Balancer (ALB) → forwards traffic to EC2 or containerized app servers.

* **Security Layers:**

  * **AWS Shield:** Protects from DDoS attacks.
  * **AWS WAF:** Filters malicious requests (SQLi, XSS, etc.) before reaching ALB.

---
---
### Quick Revision – JSON in Backend

* **Purpose**: JSON (JavaScript Object Notation) is used to transfer structured data between frontend and backend systems.
* **Format**: Lightweight, text-based key-value pairs — easy for both humans and machines.
* **Usage in Backend**:

  * Receives frontend data in JSON format (e.g., login info, form data).
  * Sends API responses back to frontend as JSON.
  * Stores intermediate configuration or API payloads.
* **Conversion**:

  * Backend frameworks automatically parse JSON to internal objects.
  * Example: In Python (Flask/Django), `request.get_json()` → converts incoming JSON body.
  * In Node.js (Express), `app.use(express.json())` → enables JSON body parsing.
* **Common Use Cases**:

  * API communication (REST APIs send/receive JSON)
  * Configuration files
  * Logging structured data

**Example Flow**:
Frontend → sends JSON → Backend parses it → interacts with DB → returns JSON response.

---
---

## Edge Caching and CDN Interaction

### Concept / What

Edge caching is storing content closer to end users at edge servers of a CDN. AWS CloudFront is a fully managed CDN that handles edge servers automatically.

### Why / Purpose

* Reduces latency by serving content from nearby edge locations.
* Offloads origin servers, improving scalability and availability.
* Handles traffic spikes efficiently.

### How it Works / Flow

```
User Browser
     |
     v
+-------------------+
| CDN Edge Location |  <- Edge Cache
+-------------------+
     |
     | Cache Hit -> Serves directly to user
     |
     v Cache Miss
+-------------------+
| Origin Server /   |
| Application Load  |
| Balancer / Backend|
+-------------------+
     |
     v
   Database
```

* Browser requests content (e.g., `/images/logo.png`).
* DNS resolves to nearest CloudFront edge.
* Edge checks cache: serves if hit, fetches from origin if miss.
* Cache is stored temporarily at edge with configured TTL.

### Key Concepts

* TTL (Time to Live) determines how long content stays in cache.
* Cache-Control headers define cache rules.
* Cache invalidation updates outdated content.
* Origin can be S3, ALB, or backend server.
* Lambda@Edge can modify requests/responses at edge.

### DevOps vs Developer Responsibilities

**Developer:**

* Decide what to cache and how long (TTL, headers, invalidation).
* Ensure dynamic content is bypassed or has proper cache rules.

**DevOps:**

* Deploy and maintain CloudFront CDN.
* Ensure origin servers are reachable (S3, ALB, backend).
* Monitor CloudFront metrics via CloudWatch (cache hit ratio, latency, errors).
* Monitor CloudFront logs for cache vs origin requests.
* Maintain security: WAF, AWS Shield, HTTPS/TLS.
* Troubleshoot distribution-level issues (not individual cache entries).

### Trade-offs / Scenarios

| Scenario                | Trade-off                                  |
| ----------------------- | ------------------------------------------ |
| Very short TTL          | More origin requests, less caching benefit |
| Long TTL                | Risk of serving stale content              |
| Dynamic content caching | Requires smart rules or cache bypass       |
| Large media files       | High caching benefit, reduces origin load  |

### Common Issues / Errors

| Issue                              | Cause                            | Fix                                           |
| ---------------------------------- | -------------------------------- | --------------------------------------------- |
| Cache serving outdated content     | TTL too long or no invalidation  | Configure proper TTL or invalidate cache      |
| CORS errors on cached assets       | Missing CORS headers             | Add headers at origin or edge                 |
| Cache misses for frequent requests | Edge locations not populated yet | Pre-warm cache or use Lambda@Edge to prefetch |

### Best Practices / Tips

✅ Serve static content through CDN edge servers.
✅ Use cache-control headers wisely.
✅ Invalidate cache selectively after updates.
✅ Use origin failover (CloudFront origin groups).
✅ Cache only idempotent GET requests for dynamic APIs.
✅ Monitor cache hit ratio and TTL effectiveness.

### Real-World AWS Example

* Frontend static assets: React app → S3 → CloudFront.
* Dynamic API: `/api/products` → ALB → EKS pods → DB.
* Edge caching: frequently requested images cached at CloudFront edge.
* TTL & invalidation: updated image → invalidate cache → next request serves updated version.
* Security layers: WAF + Shield protect traffic to origin and ALB.

---
---

## Quick Revision – Error Handling and Retries

* **What:** Detect and manage errors; retries automatically resend failed requests.

* **Why:** Improve reliability, reduce manual intervention, prevent crashes, graceful degradation.

* **Flow:**

```
Browser → Frontend → CloudFront CDN → ALB/WAF/Shield → Backend → DB
Error detected → Retry/Fallback → Response to user
```

* **Error Codes:**

  * Frontend: network/JS (timeout, CORS)
  * CDN: 503, 504 (edge/origin)
  * Backend: 5xx (500, 502, 503)
  * Database: timeout, deadlock

* **Patterns:** Exponential backoff, circuit breaker, fallback, bulkhead.

* **Best Practices:**

  * Retry idempotent ops only
  * Exponential backoff + jitter
  * Circuit breakers
  * Log failures and retries (CloudWatch/ELK)
  * Graceful degradation
  * Use cache for transient failures

* **Dev Responsibilities:** Implement retry logic, backoff, circuit breakers, fallback, idempotency.

* **DevOps Responsibilities:**

  * Monitor metrics and logs (CloudWatch)
  * Alerting on failures
  * Troubleshoot infrastructure and systemic issues
  * Coordinate with developers for persistent errors

* **AWS Example:** React frontend → CloudFront → ALB → EKS pods → RDS; retries with exponential backoff; circuit breaker prevents continuous retries; CloudWatch monitors errors and metrics.

---
---
---

