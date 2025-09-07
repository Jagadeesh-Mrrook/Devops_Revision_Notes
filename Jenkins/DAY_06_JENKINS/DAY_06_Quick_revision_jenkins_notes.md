## Jenkins Build Triggers Notes - Quick Revision Version

• **Concept / What:** Build triggers = define when/how Jenkins job starts (manual/automatic).
• **Why / Purpose:** Automates CI/CD; ensures continuous build/test.
• **How / Steps:**

* **Freestyle Job:** Configure → Build Triggers → Manual / Poll SCM / Build periodically / GitHub webhook.
* **Pipeline Job:** `triggers` block in Jenkinsfile (pollSCM, cron); webhooks handled externally in SCM.
  • **Common Issues:** Polling fails, cron syntax errors, webhook not firing.
  • **Troubleshooting:** Check repo URL/credentials, cron syntax, webhook URL/network.
  • **Best Practices:** Prefer webhooks, use `H/` in cron, manual triggers for ad-hoc, define pipeline triggers in Jenkinsfile.

---

## Jenkins Unit Testing Integration Notes - Quick Revision Version

• **Concept / What:** Automatically run unit tests in Jenkins CI/CD pipeline (JUnit, PyTest).
• **Why / Purpose:** Ensure code quality, detect bugs early, provide automated developer feedback.
• **How / Steps:**

* **Freestyle Job:** Configure → Build → Invoke Maven/Ant/Shell → Run tests → Publish JUnit report.
* **Pipeline Job:** Use Jenkinsfile stage with `sh 'mvn clean install'` or `pytest` → `junit 'target/surefire-reports/*.xml'`.
  • **Common Issues:** Test reports missing, dependencies missing, incorrect paths.
  • **Troubleshooting:** Verify framework installation, check paths, ensure Jenkins workspace environment correct.
  • **Best Practices:** Fail build on test failure, use plugins to visualize results, keep tests fast and isolated, use Maven lifecycle to include unit tests automatically in build.
---

```markdown
# Jenkins – Code Coverage Tools (Quick Revision)

## Concept / What:
Tools like **JaCoCo** measure code executed by unit tests, generating reports on lines, branches, and methods covered.

## Why / Purpose / Use Case in Real-World:
- Ensures critical code paths are tested.
- Highlights untested code for risk reduction.
- Generates reports during Jenkins build.
- Optional integration with SonarQube for thresholds.

## How it Works / Steps / Syntax:
1. **Maven Setup:** Add JaCoCo plugin in `pom.xml`.
2. **Build Stage:** `mvn clean install` runs unit tests and generates coverage report (`target/site/jacoco/index.html`).
3. **SonarQube Integration (Optional):** Feed JaCoCo report for dashboards and quality gates.

## Common Issues / Errors:
- Missing reports → plugin misconfiguration or wrong paths.
- Failed tests → incomplete coverage.
- SonarQube ignores report → incorrect path or plugin setup.

## Troubleshooting / Fixes:
- Check Maven JaCoCo plugin setup.
- Confirm workspace paths in Jenkins and SonarQube.
- Verify test execution.
- Inspect `index.html` report for coverage.

## Best Practices / Tips:
- Always generate reports during build stage.
- Use SonarQube for coverage thresholds, not Maven alone.
- Developers validate locally; pipeline enforces post-build.
- Keep quality gate thresholds configurable.
```

---

```markdown
## Jenkins – Artifact Storage & Secure Credentials (Quick Revision Version)

## Concept / What:
Centralized storage for build artifacts in Nexus or JFrog; passwords encrypted for security.

## Why / Purpose / Use Case:
- Central repo for artifacts, versioning, traceability.
- CI/CD pipelines can deploy artifacts securely.
- Keeps credentials safe from plain text exposure.

## How it Works / Steps:
1. Configure Nexus/Artifactory repositories.
2. Add encrypted server credentials in `settings.xml`.
3. Master password stored in `settings-security.xml` decrypts passwords.
4. Jenkins pipeline uses credentials to deploy artifacts:
```groovy
withMaven(mavenSettingsConfig: 'Maven_Settings_Credential_ID') {
  sh 'mvn clean deploy'
}
```

## Common Issues / Errors:
- Wrong repo URL → deployment fails.
- Incorrect credentials → unauthorized.
- Network issues → cannot reach repo.
- Misconfigured `settings.xml`/`settings-security.xml`.

## Troubleshooting / Fixes:
- Verify URLs and credentials.
- Check Maven settings files.
- Test connectivity from Jenkins agent.
- Use `mvn -X deploy` for debug.

## Best Practices / Tips:
- Separate snapshot and release repositories.
- Use semantic versioning.
- Encrypt passwords; do not commit to Git.
- Use Jenkins Secret Files or AWS Secrets Manager.
- Prefer Nexus/JFrog plugins for Jenkins integration.
```

---

```markdown
# Jenkins – AWS & Kubernetes Deployments (Quick Revision Version)

## Concept / What:
Automated deployment from Jenkins to AWS (CodeDeploy, Lambda) and Kubernetes (kubectl, Helm). Direct SDK/Java deployment is possible but less practical.

## Why / Purpose / Use Case:
- Automates CI/CD deployments.
- Ensures consistency, rollback, and traceability.
- Supports rolling, blue/green, or canary deployments.
- Example: Jenkins builds artifact/Docker image and deploys to AWS/K8s.

## How it Works / Steps / Syntax:
### AWS CodeDeploy
1. Pre-requisites: S3 bucket, CodeDeploy app & deployment group, IAM roles.
2. Jenkins uploads artifact to S3 and triggers deployment:
```groovy
aws deploy create-deployment --application-name MyApp --deployment-group-name MyApp-Dev --s3-location bucket=my-app-bucket,key=my-app.war,bundleType=zip
```

### AWS Lambda
1. Package function as ZIP.
2. Jenkins updates Lambda function:
```groovy
aws lambda update-function-code --function-name MyLambdaFunction --zip-file fileb://function.zip
```

### Kubernetes kubectl
- Apply deployment YAMLs:
```groovy
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

### Kubernetes Helm
- Deploy/upgrade Helm chart:
```groovy
helm upgrade --install myapp ./helm-chart --namespace dev --set image.tag=${BUILD_NUMBER}
```

## Common Issues / Errors:
- IAM/permissions errors, wrong S3 bucket, function or image tag errors.
- kubeconfig misconfigured, Helm chart errors, resource limits exceeded.

## Troubleshooting / Fixes:
- Verify credentials, repo paths, deployment targets.
- Helm lint and kubectl get pods.
- Ensure correct artifact/image in AWS/K8s.

## Best Practices / Tips:
- AWS: Blue/green deployments, rollback, version artifacts.
- Lambda: Use versions/aliases, test events.
- K8s: Namespaces for environments, separate stages, Helm for complex apps.
- Direct SDK deployment possible but managed services are recommended.
```

