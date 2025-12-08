# Helm Master Concept Plan (Checklist Version)

1. **Helm Fundamentals**

* [ ] What is Helm and why it's used
* [ ] Difference between Helm and kubectl
* [ ] Charts, releases, repositories
* [ ] How Helm improves config management
* [ ] Helm in GitOps workflows

2. **Helm Installation & Setup**

* [ ] Installing Helm
* [ ] Adding repositories (bitnami, stable, etc.)
* [ ] helm repo update
* [ ] Chart folder structure (Chart.yaml, values.yaml, templates/, charts/, .helmignore)

3. **Helm Chart Structure Deep Dive**

* [ ] Chart.yaml fields
* [ ] values.yaml usage
* [ ] templates/*.yaml
* [ ] helpers.tpl
* [ ] Naming templates

4. **Values (Core Concept)**

* [ ] Global values
* [ ] Environment-specific values
* [ ] Overriding values via CLI
* [ ] Multiple values files
* [ ] Values precedence
* [ ] How values populate templates

5. **Templating Engine**

* [ ] Template syntax
* [ ] .Values
* [ ] .Release
* [ ] .Chart
* [ ] Conditionals (if/else)
* [ ] Loops (range)
* [ ] Pipeline operators
* [ ] Helper templates (helpers.tpl)
* [ ] Template patterns for Deployment, Service, Ingress, ConfigMaps, Secrets

6. **Rendering & Validating Templates**

* [ ] helm template
* [ ] helm lint
* [ ] Debugging templates
* [ ] Dry runs
* [ ] Validate rendered YAML

7. **Installing & Upgrading Releases**

* [ ] helm install
* [ ] helm upgrade
* [ ] helm upgrade --install
* [ ] helm rollback
* [ ] helm history
* [ ] helm uninstall
* [ ] How Helm stores release history

8. **Helm Release Lifecycle**

* [ ] What is a release
* [ ] Release naming conventions
* [ ] Release versioning

9. **Dependency Management**

* [ ] charts/ directory
* [ ] Declaring dependencies
* [ ] Updating dependencies
* [ ] helm dependency update

10. **Security & Secrets**

* [ ] Why values.yaml should not contain secrets
* [ ] Using AWS Secrets Manager
* [ ] Using SSM Parameter Store
* [ ] Using External Secrets Operator
* [ ] helm-secrets plugin (optional)

11. **Packaging & Sharing Charts**

* [ ] Packaging your chart (helm package)
* [ ] Hosting in S3
* [ ] Hosting in GitHub Pages
* [ ] Hosting in CodeArtifact
* [ ] helm repo index

12. **Helm Best Practices**

* [ ] Avoid hardcoding values
* [ ] Keep defaults minimal
* [ ] Use helpers.tpl
* [ ] Separate dev/stage/prod values
* [ ] CI/CD integration
* [ ] Keep secrets outside Helm

13. **Real-World DevOps Scenarios**

* [ ] Deploy microservices to EKS
* [ ] Configure Ingress with ALB controller
* [ ] Blue-Green/Canary deployments via values
* [ ] Zero-downtime upgrade
* [ ] Manage multiple microservices
* [ ] Helm in Jenkins pipelines

14. **Final Helm Project**

* [ ] Create chart from scratch
* [ ] Write Deployment, Service, Ingress templates
* [ ] Add ConfigMap & Secret templates
* [ ] Create multiple values files
* [ ] Deploy to EKS
* [ ] Upgrade the release
* [ ] Roll back the release
* [ ] Package chart
* [ ] Upload chart to S3
* [ ] Install chart from S3

