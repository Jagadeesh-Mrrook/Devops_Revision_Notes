# Three-Tier Architecture - Checklist

## 1. Frontend Layer
- [ ] Web app vs Mobile app (React / React Native)
- [ ] Static vs dynamic content
- [ ] UI assets (images, templates, CSS, JS files)
- [ ] Serving static content (S3 + CloudFront vs Web server on EC2/K8s)
- [ ] Frontend pods in Kubernetes
- [ ] Frontend request flow (user → browser/app → frontend → backend)
- [ ] HTTPS and TLS (certificates, handshake, termination at ALB vs end-to-end)
- [ ] Common issues and troubleshooting (CORS, slow load, asset caching)

## 2. Backend / Application Layer
### 2.1 Application Server
- [ ] Application server (Node.js, Java, etc.)
- [ ] REST API design and request flow
- [ ] Dynamic content generation
- [ ] Connection with frontend (API endpoints)
- [ ] Load balancing (ALB, NLB) and session handling
- [ ] Auto Scaling of backend pods or EC2 instances
- [ ] Security: WAF, IAM roles, API throttling
- [ ] Common issues and troubleshooting (500 errors, latency, scaling failures)

### 2.2 Web Server
- [ ] Static content serving (Nginx, Apache)
- [ ] Reverse proxy configuration to backend
- [ ] SSL termination at web server
- [ ] Caching and compression (gzip, CDN integration)
- [ ] Common issues and troubleshooting (misconfigured proxies, TLS handshake failures)

## 3. Database Layer
- [ ] Relational vs NoSQL database selection
- [ ] Connection pooling and database access from app
- [ ] Scaling strategies (vertical, read replicas, sharding)
- [ ] Backup and restore, snapshot strategies
- [ ] Security: encryption at rest, IAM policies
- [ ] High availability and failover
- [ ] Common issues and troubleshooting (connection limits, replication lag, deadlocks)

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

