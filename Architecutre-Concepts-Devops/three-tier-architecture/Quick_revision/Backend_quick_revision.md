# AWS & Web Server Quick Revision Notes

## NLB & WAF/Shield

* NLB uses **Shield only**, WAF not applicable.

### Common Issues

* WAF misconfigured → blocks legit traffic
* IAM missing permissions → 403 errors
* API throttling too strict → 429 errors
* Large DDoS → need Shield

### Best Practices

* Use least privilege IAM roles
* Apply WAF managed rulesets
* Implement API throttling
* Monitor with CloudWatch/CloudTrail
* Test in staging
* Shield + WAF together for ALB
* Shield only for NLB

### Common Attacks (One-line)

* **DDoS** → server overload
* **SQL Injection** → malicious DB queries
* **XSS** → steal user sessions
* **Malicious Bots** → automated abuse

### DevOps Role

* Configure Shield & WAF
* Assign/manage IAM roles
* Implement API throttling
* Monitor logs & metrics
* Ensure proper protection for ALB/NLB

---

## Application Server: Common Issues & Troubleshooting

### 500 Internal Server Errors

* **Cause:** Code bugs, DB failures, env misconfig, low memory/CPU
* **Fix:** Check logs, validate DB, review code, scale resources

### High Latency / Slow Responses

* **Cause:** Heavy DB queries, blocking APIs, resource/network limits
* **Fix:** Profile queries, caching (Redis), async calls, scale horizontally, monitor metrics

### Scaling Failures

* **Cause:** Misconfigured Auto Scaling/HPA, dependency bottlenecks
* **Fix:** Validate scaling triggers, check HPA logs, ensure stateless backend, optimize thresholds

### Other Common Issues

* Auth/Authorization errors (JWT, IAM roles)
* Connection timeouts (DB or external services)
* Memory leaks/crashes

### Best Practices

* Centralize logs and metrics
* Implement health checks
* Keep backend stateless
* Monitor resources and app performance
* Automate alerts for errors/spikes
* Use caching & async calls
* Validate scaling configurations regularly

---

## 2.2 Web Server - Quick Revision Notes

### Web Server

* **Definition:** Software/hardware serving content via HTTP/HTTPS
* **Purpose:** Deliver static/dynamic content, offload backend, enable HTTPS, act as reverse proxy
* **Flow:** Client → Web Server → (Static: direct / Dynamic: backend) → Response
* **Best Practices:** HTTPS, gzip/brotli, CDN, dedicated static directory

### Static Content Serving (Nginx, Apache)

* **Definition:** Serve HTML, CSS, JS, images directly
* **Purpose:** Fast delivery, reduce backend load, scalable
* **Examples:** Nginx (`/var/www/html`), Apache (`DocumentRoot`), AWS S3 + CloudFront
* **Common Issues:** 403/404, cache issues, MIME mismatch
* **Fixes:** Correct permissions, clear cache, correct MIME types
* **Best Practices:** CDN, cache-busting, HTTPS, compression
* **Responsibilities:**

  * **DevOps:** Install/configure server, SSL/TLS, caching, reverse proxy
  * **Developer:** Provide build artifacts, recommend caching or headers

### Reverse Proxy Configuration to Backend

* **Definition:** Server (Nginx/Apache) between clients and backend, forwards requests, hides backend IPs
* **Purpose:** Load balancing, security, SSL termination, caching, centralized logging
* **Flow:** Client → Reverse Proxy → Backend → Reverse Proxy → Client
* **Nginx Example:**

```nginx
upstream backend_servers {
  server 10.0.0.1:3000;
  server 10.0.0.2:3000;
}
server {
  listen 80;
  server_name example.com;
  location / {
    proxy_pass http://backend_servers;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

* **Common Issues:** 502/504 errors, lost client IP, SSL issues, backend unreachable
* **Fixes:** Check logs, verify backend health, set `X-Forwarded-For`, validate SSL
* **Best Practices:** Health checks, sticky sessions if needed, SSL offloading, caching, monitoring
* **Responsibilities:**

  * **DevOps:** Install, maintain, implement config, ensure SSL/connectivity/performance
  * **Developer:** Provide routing rules/backend mapping

---

## 2.2 Web Server - SSL/TLS Quick Revision Notes

### TLS Encryption

• Encrypts traffic between client and server for confidentiality, integrity, authentication.
• Symmetric encryption for data transfer, asymmetric for key exchange.

### TLS Handshake

• Establishes secure connection and negotiates encryption.
• Steps: Client Hello → Server Hello → Server Certificate → Key Exchange → Handshake Complete.
• Encrypted communication begins after handshake.

### TLS/SSL Termination

• Web server or reverse proxy decrypts HTTPS traffic.
• End-to-End TLS: traffic remains encrypted to backend.
• Flow:

* Normal: Client → Web Server → HTTP → Backend
* End-to-End: Client → Web Server → HTTPS → Backend
  • Nginx config example included for end-to-end TLS.

### Certificates / CA

• CA issues digital certificates; authenticate server identity.
• Stored on server directories (`/etc/ssl/certs/`, `/etc/ssl/private/`) or in AWS ACM/Secrets Manager.
• Used during TLS handshake for authentication.
• Best Practices: TLS 1.2/1.3, strong ciphers, secure storage, automated renewals.

### Common Issues & Fixes

• Handshake failure → update TLS versions/ciphers
• Expired/invalid certificate → renew/replace
• Wrong path → correct server config
• Weak cipher → disable

### DevOps Role

• Install/configure server/load balancer for TLS termination
• Deploy/manage certificates
• Monitor handshake logs
• Ensure backend connectivity and optional end-to-end TLS

---

## 2.2 Web Server – Caching, Compression, and CDN – Quick Revision Notes

### **Caching (Web Server & CDN)**

* **Definition:** Temporarily store frequently accessed content to reduce backend load and speed up delivery.
* **Purpose:** Fast responses, reduced backend load, global access via CDN.
* **Flow:** Client → Cache (Web Server/CDN) → Backend (if cache miss) → Client.
* **Common Issues:** Stale cache, misconfigured TTL, CDN cache miss.
* **Fixes:** Invalidate cache, adjust TTL, monitor cache hit ratio.
* **Best Practices:** Cache static content, selective dynamic caching, monitor and tune.

### **Compression**

* **Definition:** Reduce size of responses (HTML, CSS, JS) before sending.
* **Purpose:** Faster delivery, lower bandwidth, better UX.
* **How:** Enable gzip/brotli in NGINX/Apache, compress text-based content.
* **Common Issues:** Compression not applied, wrong MIME type.
* **Fixes:** Enable modules, verify MIME, test with curl/dev tools.
* **Best Practices:** Always compress text, combine with caching, monitor CPU usage.

### **CDN Integration**

* **Definition:** Global servers caching content near users.
* **Purpose:** Reduce latency, offload origin, improve availability.
* **How:** Configure origin, define cache behavior, serve cached content, fetch on miss.
* **Common Issues:** Incorrect cache policy, misconfigured origin, TTL too long.
* **Fixes:** Invalidate cache, adjust TTL, check origin availability.
* **Best Practices:** Cache static assets, selective dynamic caching, enable compression, monitor hit/miss ratio.

### **DevOps vs Developer Responsibilities**

* **DevOps:** Install/configure server, enable caching/compression, configure CDN, monitor performance.
* **Developer:** Provide static content/build artifacts, recommend caching paths/headers/TTL.


---

## 2.2 Web Server – Common Issues & Troubleshooting – Quick Revision Notes

### **Common Errors**

* **4XX Errors:** Client-side issues like wrong URLs, missing files, permissions → 403/404.
* **5XX Errors:** Server-side issues like backend down, reverse proxy misconfig → 500/502/504.
* **TLS/SSL Errors:** Expired/mismatched certificates, unsupported TLS → handshake failures, HTTPS errors.
* **Performance/Latency:** High CPU/memory, database slowness, no caching → slow responses.
* **Config Syntax Errors:** Typos/misconfig → server fails reload.

### **Troubleshooting / Fixes**

* Check server logs (`/var/log/nginx/error.log`, `/var/log/apache2/error.log`).
* Verify backend health and connectivity.
* Validate TLS/SSL certificates (openssl s_client, correct permissions, renew if expired).
* Correct file permissions and root paths (directories 755, files 644).
* Enable caching/compression, adjust workers, use CDN for performance.
* Test configuration syntax before reload (`nginx -t`, `apachectl configtest`).

### **Best Practices**

* Monitor logs and metrics regularly.
* Test configs in staging first.
* Keep servers updated.
* Use caching, compression


---
---
