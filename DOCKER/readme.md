# 🐳 Docker Concepts Checklist (DevOps – 5 Years)

Use this as your GitHub README.md for revision.

---

## ## 1. Docker Core Fundamentals

* [ ] What is Docker & why it’s used
* [ ] Containers vs Virtual Machines
* [ ] Docker Engine architecture (dockerd, containerd, runc)
* [ ] Images, layers, caching
* [ ] Container lifecycle (create/start/stop/remove)
* [ ] Basic Docker commands

---

## 2. Docker Images (Build → Tag → Push → Pull)

* [ ] Building images
* [ ] Tagging convention & versioning
* [ ] Pushing/pulling images (Docker Hub / ECR)
* [ ] Inspecting images
* [ ] Image history & layers
* [ ] Understanding registries

---

## 3. Dockerfile Mastery

* [ ] FROM, RUN, COPY, WORKDIR, ENV
* [ ] CMD vs ENTRYPOINT
* [ ] HEALTHCHECK instruction
* [ ] EXPOSE & USER
* [ ] .dockerignore usage
* [ ] Production-ready Dockerfile patterns
* [ ] Multi-stage builds
* [ ] Using minimal base images (Alpine, Distroless)

---

## 4. Containers

* [ ] Running containers (detached/interactive)
* [ ] Restart policies
* [ ] Environment variables
* [ ] Inspecting & debugging containers
* [ ] Resource limits (CPU/Memory)
* [ ] Logs and shell access
* [ ] Copy files in/out of containers

---

## 5. Docker Networking

* [ ] Bridge / Host / None
* [ ] Port mapping (-p)
* [ ] DNS inside containers
* [ ] User-defined networks
* [ ] Container-to-container communication

---

## 6. Storage & Volumes

* [ ] Named volumes
* [ ] Bind mounts
* [ ] Anonymous volumes
* [ ] Data persistence patterns
* [ ] Volume backup/restore
* [ ] Permissions issues in volumes

---

## 7. Docker Compose

* [ ] Multi-container application setup
* [ ] Services, networks, and volumes
* [ ] Environment variables in Compose
* [ ] Scaling services
* [ ] Override files
* [ ] Local development with Compose

---

## 8. Docker Security

* [ ] Running non-root containers
* [ ] Image scanning (Trivy, ECR)
* [ ] Avoiding secrets in Dockerfile
* [ ] Docker capabilities basics
* [ ] Minimal base images (Alpine/Distroless)
* [ ] SBOM / supply chain basics

---

## 9. Docker in CI/CD

* [ ] Building images in CI (Jenkins)
* [ ] Layer caching
* [ ] Docker login using credentials
* [ ] Tagging strategy (commit/branch/semantic)
* [ ] Pushing to registry (ECR/Hub)
* [ ] Scanning images during CI
* [ ] Deploying built images to EKS via kubectl/Helm

---

## 10. Docker Best Practices for Kubernetes

* [ ] Logs to stdout/stderr
* [ ] Liveness/readiness probe compatibility
* [ ] Resource limits guidance
* [ ] Config injection patterns (env/files)
* [ ] Distroless images usage

---

## 11. Troubleshooting & Maintenance

* [ ] CrashLoopBackOff investigation (Docker-level)
* [ ] OOMKilled debugging
* [ ] Storage driver issues (overlay2)
* [ ] Docker daemon logs
* [ ] Disk cleanup (prune, dangling)
* [ ] Permission issues in mounts

---

## 12. Optional: Docker Swarm (Basic Awareness Only)

* [ ] Manager vs Worker nodes
* [ ] Services & tasks
* [ ] Why Kubernetes replaced Swarm

---

### ✅ This checklist is complete and production-focused. You can directly push this to GitHub as your Docker learning + revision tracker.

