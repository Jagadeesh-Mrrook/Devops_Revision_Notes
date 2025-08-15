# Jenkins Master Checklist for 5+ Years Experience

A comprehensive list of Jenkins concepts, features, and integrations for interviews and real-world DevOps work.

---

## 1. **Jenkins Fundamentals**
- [ ] What is Jenkins & why it's used
- [ ] Jenkins architecture: Master/Agent (controller/worker) model
- [ ] Jenkins installation (Linux, Windows, Docker)
- [ ] Jenkins UI overview: Dashboard, Views, Build history
- [ ] Freestyle jobs vs Pipeline jobs
- [ ] Running builds manually, on SCM changes, on schedule (CRON)
- [ ] Jenkins home directory & file structure

---

## 2. **Pipeline as Code**
- [ ] Declarative vs Scripted pipelines
- [ ] Jenkinsfile creation and storage in SCM
- [ ] Pipeline stages, steps, and post actions
- [ ] Parameters in pipelines
- [ ] Environment variables (built-in & custom)
- [ ] Using `when` conditions in pipelines
- [ ] Error handling (`try/catch/finally`, `catchError`)
- [ ] Parallel stages execution
- [ ] Agent selection (`agent any`, `agent none`, `agent { label 'node' }`)

---

## 3. **Credentials & Security**
- [ ] Storing and using credentials (username/password, secret text, SSH keys, AWS creds)
- [ ] Credentials binding in pipelines
- [ ] Role-based access control (RBAC)
- [ ] Matrix-based security
- [ ] Integrating with LDAP/AD
- [ ] Securing Jenkins with HTTPS
- [ ] Preventing CSRF attacks and securing API tokens

---

## 4. **Plugins & Integrations**
- [ ] Plugin management (install, update, remove)
- [ ] Commonly used plugins:
  - [ ] Git, GitHub, GitLab
  - [ ] Pipeline
  - [ ] Blue Ocean
  - [ ] Credentials Binding
  - [ ] Email Extension
  - [ ] SonarQube Scanner
  - [ ] Nexus Artifact Uploader
  - [ ] Slack Notification
  - [ ] Kubernetes
- [ ] Dependency management & plugin compatibility

---

## 5. **Source Code Management**
- [ ] Git integration (HTTPS & SSH)
- [ ] GitHub/GitLab/Bitbucket webhooks
- [ ] Multi-branch pipelines
- [ ] SCM polling
- [ ] Pull request builds (GitHub PR builder)

---

## 6. **Build, Test, and Deploy**
- [ ] Build triggers (manual, SCM changes, periodic)
- [ ] Unit testing integration (JUnit, PyTest, etc.)
- [ ] Code coverage tools
- [ ] Artifact storage (Nexus, Artifactory, S3)
- [ ] Deployments to:
  - [ ] On-prem servers (SSH, Ansible)
  - [ ] AWS (CodeDeploy, Lambda)
  - [ ] Kubernetes (kubectl, Helm)

---

## 7. **Quality & Security Scans**
- [ ] SonarQube integration
  - [ ] Quality Profiles & Quality Gates
  - [ ] PR decoration
- [ ] Static code analysis
- [ ] Dependency scanning (OWASP Dependency Check, Snyk)

---

## 8. **Distributed Build Setup**
- [ ] Adding and managing Jenkins agents
- [ ] Static vs dynamic agents
- [ ] Connecting agents via SSH, JNLP, or Kubernetes
- [ ] Scaling Jenkins builds with Kubernetes plugin
- [ ] Node labels & node selection in pipelines

---

## 9. **Monitoring & Maintenance**
- [ ] Jenkins logs & troubleshooting (`/var/log/jenkins/jenkins.log`)
- [ ] Enabling system logs for debugging
- [ ] Build history retention & cleanup
- [ ] Backup & restore (Jenkins home, job configs)
- [ ] Monitoring Jenkins health (via CloudWatch, Prometheus, Grafana)

---

## 10. **Advanced Jenkins Features**
- [ ] Shared Libraries for reusable pipeline code
- [ ] Jenkinsfile linting & validation
- [ ] Dynamic parameters in pipelines
- [ ] Pipeline input steps for approvals
- [ ] Matrix builds
- [ ] Parallel job execution across agents

---

## 11. **UI & Visualization**
- [ ] Blue Ocean UI for visual pipelines
- [ ] Build trend graphs
- [ ] Test results visualization
- [ ] Console output & log filtering

---

## 12. **Notifications & ChatOps**
- [ ] Email notifications (HTML templates)
- [ ] Slack/MS Teams integration
- [ ] Build status notifications
- [ ] Custom message formatting in notifications

---

## 13. **Best Practices for Enterprise Jenkins**
- [ ] Folder-based job organization
- [ ] Naming conventions for jobs & pipelines
- [ ] Minimal plugin usage for stability
- [ ] Using `lock` and `throttle` plugins for resource control
- [ ] Store Jenkinsfiles in SCM (not in Jenkins UI)
- [ ] Regular backups & disaster recovery plan
- [ ] Periodic plugin & Jenkins core upgrades

---

## 14. **Hands-On Scenarios**
- [ ] CI/CD pipeline for Java/Maven application
- [ ] CI/CD pipeline for Dockerized microservices
- [ ] CI/CD pipeline deploying to Kubernetes
- [ ] Integrating security scans in the build process
- [ ] Multi-environment deployments (dev, qa, prod)

