# Three-Tier Architecture - Checklist

## 1. Frontend Layer
- [x] Web app vs Mobile app (React / React Native)
- [x] Static vs dynamic content
- [x] UI assets (images, templates, CSS, JS files)
- [x] Serving static content (S3 + CloudFront vs Web server on EC2/K8s)
- [x] Frontend pods in Kubernetes
- [x] Frontend request flow (user → browser/app → frontend → backend)
- [x] HTTPS and TLS (certificates, handshake, termination at ALB vs end-to-end)
- [x] Common issues and troubleshooting (CORS, slow load, asset caching)

## 2. Backend / Application Layer
### 2.1 Application Server
- [x] Application server (Node.js, Java, etc.)
- [x] REST API design and request flow
- [x] Dynamic content generation
- [x] Connection with frontend (API endpoints)
- [x] Load balancing (ALB, NLB) and session handling
- [x] Auto Scaling of backend pods or EC2 instances
- [x] Security: WAF, IAM roles, API throttling
- [x] Common issues and troubleshooting (500 errors, latency, scaling failures)

### 2.2 Web Server
- [x] Static content serving (Nginx, Apache)
- [x] Reverse proxy configuration to backend
- [x] SSL termination at web server
- [x] Caching and compression (gzip, CDN integration)
- [x] Common issues and troubleshooting (misconfigured proxies, TLS handshake failures)

## 3. Database Layer
- [x] Relational vs NoSQL database selection
- [x] Connection pooling and database access from app
- [x] Scaling strategies (vertical, read replicas, sharding)
- [x] Backup and restore, snapshot strategies
- [x] Security: encryption at rest, IAM policies
- [x] Database connectivity with the backend server
- [x] High availability and failover
- [x] Common issues and troubleshooting (connection limits, replication lag, deadlocks)

## 4. Request and Data Flow
- [ ] Typical user request flow (Browser/App → Frontend → Backend → Database)
- [ ] Backend returns JSON + static URLs → frontend renders
- [ ] Edge caching and CDN interaction
- [ ] Error handling and retries

## 5. Deployment and Environment Considerations
- [ ] Dev / Staging / Prod separation
- [ ] CI/CD pipelines for frontend and backend
- [ ] Containerized deployment (Docker/K8s)
- [ ] Versioning and rollback strategies

## 6. Monitoring and Logging
- [ ] Frontend logging (errors, asset load times)
- [ ] Backend logs and metrics
- [ ] Database monitoring and alerts
- [ ] End-to-end request tracing

## 7. Common Bottlenecks and Troubleshooting
- [ ] Slow page loads due to static asset misconfiguration
- [ ] API latency due to backend scaling issues
- [ ] Database connection limits or replication lag
- [ ] Load balancer misconfiguration
- [ ] Security misconfigurations (WAF, TLS, IAM)

