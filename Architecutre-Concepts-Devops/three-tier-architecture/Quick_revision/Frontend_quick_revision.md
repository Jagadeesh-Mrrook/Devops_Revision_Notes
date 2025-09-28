# Frontend Overview - Quick Revision

## Concept / What

* Client-side layer of applications (UI/UX).
* Runs in browser or mobile app.
* Technologies: HTML, CSS, JS, frameworks (React, Angular, Vue).

## Why / Purpose / Use Case

* Handles **presentation layer**.
* Ensures **user interactivity**.
* Connects users to backend APIs.

## How it Works

1. User opens app/browser → frontend assets load.
2. Assets served from S3+CloudFront, CDN, or web server.
3. Frontend sends requests (API calls) → backend.
4. Backend responds (JSON/data) → frontend renders UI.

## Common Issues / Errors

* CORS errors.
* Slow asset load (large JS/CSS/images).
* Browser compatibility issues.

## Troubleshooting / Fixes

* Configure CORS properly.
* Optimize + cache static assets (CDN, compression).
* Test across browsers/devices.

## Best Practices / Tips

* Use CDN for faster delivery.
* Secure with HTTPS (TLS certs).
* Follow responsive design principles.
* Monitor frontend errors (Sentry, CloudWatch RUM).

---

# Web App vs Mobile App (React / React Native) - Quick Revision

### Concept / What

* Web App: Browser-based (React.js)
* Mobile App: Installed app (React Native)

### Why / Use Case

* Web: Fast updates, accessible via URL, dashboards/portals.
* Mobile: Offline access, device integration, push notifications.

### Deployment Flow

* Web:

```
User → Browser → Frontend → Backend APIs → Backend Services
```

* Mobile:

```
User → Mobile App → Backend APIs → Backend Services
```

### Common Issues

* Web: Slow load, CORS errors
* Mobile: API compatibility, delayed updates

### Fixes

* Optimize assets, set CORS headers, maintain API backward compatibility, automate CI/CD

### Best Practices

* S3 + CloudFront for web
* Common backend for both apps
* Automate CI/CD for both
* Offline handling only for mobile

---

# Static vs Dynamic Content - Quick Revision

### Concept / What

* Static: Unchanging files (HTML, CSS, JS, images)
* Dynamic: Generated per user request (shopping cart, dashboard)

### Why / Use Case

* Static: Fast, cacheable, landing pages, assets
* Dynamic: Personalized content, real-time data

### Deployment Flow

* Static: User → Browser → CDN/S3/Web Server → Files
* Dynamic: User → Browser → Frontend → Backend → DB → Backend → Frontend → Browser

### Common Issues

* Static: Outdated cached files
* Dynamic: Backend bottlenecks, slow DB queries

### Fixes

* Cache invalidation, versioned files, optimize queries, load balancer + auto-scaling, caching layers

### Best Practices

* Serve static via CDN
* Minimize dynamic computation; cache results
* Separate static frontend & dynamic backend
* SPA/frontend calls dynamic APIs for personalization

---

# UI Assets and Static/Dynamic Content Hosting - Quick Revision

### Concept / What

* UI Assets: HTML, CSS, JS, images, templates
* Static content: Unchanging files (frontend assets)
* Dynamic content: Personalized or real-time data from backend

### Why / Use Case

* UI Assets make frontend interactive and user-friendly
* Static: fast, cacheable, minimal backend load
* Dynamic: real-time user data (cart, orders, product availability)

### Deployment Flow

* Static Assets: Browser → CloudFront → S3 → HTML/CSS/JS/Images
* Dynamic Content: Browser → API calls → EKS Microservices → DB → Response → Browser

### Common Issues

* Static: outdated cache, broken links, wrong MIME types, unoptimized files
* Dynamic: slow API, backend bottlenecks, inconsistent data

### Fixes

* Cache-busting / versioning for static assets
* Optimize images and JS/CSS
* Proper headers for static files
* Backend optimizations, caching, load balancers, auto-scaling

### Best Practices

* Serve static assets via CDN (S3 + CloudFront)
* Separate frontend static assets from dynamic backend microservices
* Automate deployments in CI/CD
* Lazy load images, minify JS/CSS
* SPA frontend calls backend APIs for dynamic content

---

# Serving Static Content - Quick Revision

### Concept / What

* Static content: HTML, CSS, JS, images, templates
* Hosted for fast, reliable delivery to users

### Why / Use Case

* Low latency, high availability
* Reduces backend load
* SPA frontend uses static assets; dynamic content served by backend

### Deployment Options

1. **S3 + CloudFront**

   * Store in S3, distribute via CloudFront CDN
   * No web servers needed, auto-scaling, cache control
2. **Web server on EC2 / EKS pods**

   * Nginx/Apache serving files, routed via ALB/NLB
   * More control, higher operational overhead

### Common Issues

* Old cached assets
* Wrong MIME types or paths
* Load balancing sync issues

### Fixes

* Versioned file names for cache-busting
* Correct headers and CloudFront invalidation
* Sync files across servers if using EC2/EKS

### Best Practices

* Prefer S3 + CloudFront
* Set caching headers properly
* Automate deployments in CI/CD
* Use compression (gzip, Brotli)

---

# Frontend Request Flow - Quick Revision

### Concept / What

* Shows how user requests flow from browser/app → frontend SPA → backend services → DB → response.
* SPA fetches static content first, then dynamic content via API calls.

### Flow Summary

1. **DNS Resolution:** [www.example.com](http://www.example.com) → Route 53 alias → CloudFront DNS
2. **Static content:** CloudFront → S3
3. **Dynamic content (API calls):** CloudFront → ALB → EKS Backend → DB
4. **Response:** Back through ALB/CloudFront → Browser

### Key Points

* Browser decides which requests are static vs dynamic.
* CloudFront routes requests to correct origin (S3 or ALB).
* Multiple HTTP requests per page load: static → S3, dynamic → backend.
* ALB IP is never exposed to the browser; browser only sees CloudFront IP.

### Common Issues / Fixes

* **CORS errors** → fix headers in backend
* **Misrouted requests** → verify CloudFront origin/path
* **Slow backend** → monitoring, autoscaling
* **SSL/TLS issues** → proper termination at CloudFront/ALB

### Best Practices

* Decouple frontend and backend; use API calls
* Serve static assets via CloudFront + S3
* Use ALB/Ingress for backend routing
* Enable logging, monitoring, and health checks
* Version static assets to avoid caching issues


---

# Frontend Pods in Kubernetes - Quick Revision

### Concept / What

* Pods run frontend app (SPA) using Nginx/Apache.
* Serve frontend code; large static assets (images/videos) from S3.

### Flow Summary

1. **Browser → Route 53 → ALB/Ingress → Frontend Pod**
2. **Pod serves HTML/CSS/JS**
3. **Large static assets fetched from S3 → Browser**
4. **Frontend JS → API calls → Backend Pods → DB → Response → Browser**

### Key Points

* Pods handle code and small static content only.
* Large static assets stored in S3.
* Pods need **IAM permissions** (node group role or IRSA) to access S3.
* ALB or Ingress exposes pods externally.

### Common Issues / Fixes

* Misconfigured Ingress → traffic not reaching pods.
* Pod crashes → unavailable frontend.
* S3 access denied → check IAM role.
* SSL/TLS misconfig → ensure correct termination.

### Best Practices

* Multiple replicas for high availability.
* Offload large assets to S3 + CloudFront.
* Use Ingress/ALB for traffic routing.
* Implement liveness/readiness probes.
* Least privilege IAM for S3 access.


---

# HTTPS and TLS - Quick Revision (DevOps Perspective)

### Concept / What

* HTTPS encrypts browser-server traffic; TLS is the modern protocol.
* Certificates (TLS/SSL) enable encryption and authentication.

### Flow

* Browser encrypts request → TLS → ALB decrypts → HTTP → Backend Pods.
* Optional end-to-end TLS: ALB forwards encrypted request → backend decrypts.

### Termination Options

| Option          | Flow                                     | Pros                | Cons                            |
| --------------- | ---------------------------------------- | ------------------- | ------------------------------- |
| ALB Termination | Internet → HTTPS → ALB → HTTP → Backend  | Less CPU on backend | Data unencrypted inside network |
| End-to-End TLS  | Internet → HTTPS → ALB → HTTPS → Backend | Full encryption     | Higher CPU usage                |

### Certificates

* DV: domain ownership, free/low-cost.
* OV: domain + org verification, paid.
* EV: strict verification, shows org in browser, high cost.
* Wildcard: all subdomains.
* SAN: multiple domains.

### Key DevOps Points

* Store paid/private certificates securely in **AWS Secrets Manager**.
* Use **ACM** for automated certificate management.
* Redirect HTTP → HTTPS; enable **HSTS**.
* Prefer ALB termination unless end-to-end TLS is required for sensitive data.
* Focus on TLS encryption/decryption flow, termination, and certificate security.


---

# Frontend Common Issues & Troubleshooting

## Quick Revision Version

### 🔹 Concept

Frontend issues = problems in delivering static/dynamic content smoothly (blocked requests, slow load, stale assets, SSL/DNS errors).

---

### 🔹 Why

* Affects **user experience**.
* DevOps ensures **DNS/CDN/ALB/API configs** + optimization.

---

### 🔹 How / Key Points

1. **CORS** → Different domains (example.com → api.example.com). Need proper `Access-Control-Allow-Origin` headers.
2. **Slow Loading** → Large/uncompressed assets, missing CDN.
3. **Caching Issues** → Old JS/CSS due to missing invalidation.
4. **DNS/SSL Issues** → Wrong Route 53 mapping, expired TLS cert.

---

### 🔹 Common Errors

* `Blocked by CORS policy` → header missing.
* High page latency → unoptimized assets.
* Old content → cache not invalidated.
* SSL error → cert expired/mismatch.
* DNS error → endpoint not resolving.

---

### 🔹 Fixes

* **CORS** → Fix headers at API/ALB.
* **Slow load** → Compress, use CloudFront, lazy load.
* **Cache** → Invalidate CloudFront, version assets.
* **SSL** → ACM managed certs, auto renew.
* **DNS** → Validate Route 53 records.

---

### 🔹 Best Practices

* Always serve static via **CDN**.
* Use **cache versioning** (`style.v2.css`).
* Automate **TLS renewal** (ACM/cert-manager).
* Enable monitoring (CloudWatch/Synthetics).
* Optimize images (WebP, resize).


---


