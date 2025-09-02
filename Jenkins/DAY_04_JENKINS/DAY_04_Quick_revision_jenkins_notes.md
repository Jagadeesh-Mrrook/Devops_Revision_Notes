## Jenkins Plugins & Integrations – Quick Revision

### Plugin Management

* Install, update, remove plugins via Manage Jenkins → Manage Plugins.
* Keep Jenkins and plugin versions compatible.

### Git, GitHub, GitLab Plugins

* Integrate SCM tools with Jenkins.
* Automate builds via commits, PRs, multi-branch pipelines.
* Requires credentials setup in Jenkins.

### Pipeline Plugin

* Enables Jenkins pipeline configuration (Declarative/Scripted).
* Automate CI/CD stages: build, test, deploy.
* Multi-branch pipeline support.

### Blue Ocean Plugin

* Modern UI for pipeline visualization.
* Faster log rendering and better pipeline monitoring.

### Credentials Binding Plugin

* Securely use credentials in pipelines without hardcoding.
* Supports Git, SonarQube, Kubernetes, AWS, and more.

### Email Extension Plugin

* Configure email notifications for build events.
* Install from Manage Jenkins → Plugins.

### SonarQube Scanner Plugin

* Static code analysis: bugs, code smells, vulnerabilities, coverage.
* Configure SonarQube server in Manage Jenkins → Configure System.
* Optional Global Tool Configuration for automatic installation on agents.

### Nexus Artifact Uploader Plugin

* Upload artifacts (.jar, .war, .zip) to Nexus.
* Configure credentials in Jenkins.
* Automates artifact promotion across repos.

### Slack Notification Plugin

* Send Slack notifications for build status.
* Integrate with channels using Webhook.
* Can be used across Jenkins, Kubernetes, Terraform pipelines.

### Kubernetes Plugin

* Spin up ephemeral Jenkins agents on Kubernetes cluster.
* Pods are created per build, isolated, then terminated.
* Supports scaling and agent image customization.

### Dependency Management & Plugin Compatibility

* Some plugins depend on others.
* Ensure plugin versions compatible with Jenkins core.
* Jenkins UI shows deprecation and compatibility warnings.

