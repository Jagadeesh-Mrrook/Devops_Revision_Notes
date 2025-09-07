## Jenkins Build Triggers Notes

### Detailed Explanation Version

• **Concept / What:**
Build triggers define **when and how a Jenkins job should start**, either manually or automatically.

• **Why / Purpose / Use Case in Real-World:**

* Automates CI/CD workflows.
* Ensures that code changes are continuously built and tested without manual intervention.
* Common in teams practicing continuous integration.

• **How it Works / Steps / Syntax:**

1. **Freestyle Job:**

   * Open Jenkins job → Configure → Build Triggers.
   * Options:

     * **Build periodically:** Cron-like schedule (e.g., `H/15 * * * *`).
     * **Poll SCM:** Check for repository changes at intervals.
     * **GitHub hook trigger:** Triggered by webhook events.
     * **Manual trigger:** Clicking **Build Now**.
2. **Pipeline Job (Jenkinsfile):**

   ```groovy
   pipeline {
       agent any
       triggers {
           pollSCM('H/15 * * * *')
           cron('H 2 * * 1-5')
       }
       stages {
           stage('Build') {
               steps { echo 'Building...' }
           }
       }
   }
   ```

   * Webhooks are configured on SCM, Jenkins automatically responds.

• **Common Issues / Errors:**

* Polling SCM fails: incorrect repo URL or credentials.
* Cron syntax errors.
* Webhook not firing: network/firewall issues or misconfiguration.

• **Troubleshooting / Fixes:**

* Verify SCM URL and credentials.
* Test cron syntax with online testers.
* Ensure webhook points to correct Jenkins endpoint and Jenkins can access it.

• **Best Practices / Tips:**

* Prefer webhooks over SCM polling for efficiency.
* Use `H/` in cron for load distribution.
* Manual triggers for ad-hoc builds.
* Keep pipeline triggers in Jenkinsfile; freestyle triggers in job config.

---


## Jenkins Unit Testing Integration Notes

### Detailed Explanation Version

• **Concept / What:**
Unit Testing Integration in Jenkins is the process of **automatically running unit tests** for the source code during the CI/CD pipeline, using frameworks like **JUnit** for Java or **PyTest** for Python.

• **Why / Purpose / Use Case in Real-World:**

* Ensures code quality and stability during continuous integration.
* Detects bugs early before deployment.
* Provides automated feedback to developers.
* Example: In a microservices project, testing each service individually before integration ensures reliability.

• **How it Works / Steps / Syntax:**

**Freestyle Job:**

1. Configure job → Build → Add build step.
2. Use **Invoke Maven**, **Invoke Ant**, or **Execute Shell** to run tests.
3. Test commands:

   * Maven: `mvn test` (or `mvn clean install` to include test automatically in build)
   * PyTest: `pytest tests/ --junitxml=report.xml`
4. Post-build action: Publish JUnit test result report.

**Pipeline Job (Jenkinsfile):**

```groovy
pipeline {
    agent any
    stages {
        stage('Build & Test') {
            steps {
                sh 'mvn clean install' // runs compile + test automatically
                // or for Python
                // sh 'pytest tests/ --junitxml=report.xml'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml' // parses test results
                }
            }
        }
    }
}
```

• **Common Issues / Errors:**

* Test reports not generated or not found.
* Dependencies missing causing tests to fail.
* Incorrect paths in Jenkins job or workspace.

• **Troubleshooting / Fixes:**

* Verify test framework installation.
* Check paths to tests and reports.
* Ensure Jenkins workspace has proper environment/dependencies.

• **Best Practices / Tips:**

* Fail the build if tests fail.
* Use JUnit/xUnit plugins to parse and visualize test results.
* Keep test stages fast and isolated.
* Maintain consistent test environments between local dev and Jenkins.
* In Maven projects, use the lifecycle (e.g., `mvn clean install`) to include unit tests automatically during build.

---

````markdown
# Jenkins – Code Coverage Tools (Detailed Explanation)

## Concept / What:
Code coverage tools measure how much of the application code is exercised by unit tests. In Jenkins pipelines, tools like **JaCoCo** generate reports showing which lines, methods, and branches were tested. These reports help ensure effective testing and can be integrated into SonarQube for dashboards and quality gates.

## Why / Purpose / Use Case in Real-World:
- Ensures **unit tests cover critical code paths**, reducing production bugs.
- Provides visibility into **untested or risky areas of code**.
- In pipelines, reports are generated **during the build stage** (Maven/Gradle).
- Optional: integrate with **SonarQube** to enforce **coverage thresholds** (e.g., fail if <40%).
- Real-world scenario: Developers push code → Jenkins runs Maven build → unit tests executed → JaCoCo generates coverage report → team reviews coverage before deployment.

## How it Works / Steps / Syntax:

### 1. Maven Setup (JaCoCo Plugin)
Add JaCoCo plugin in `pom.xml`:
```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.10</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
````

### 2. Jenkins Build Stage

Run Maven build & tests in Jenkins:

```groovy
stage('Build & Test') {
    steps {
        sh 'mvn clean install'
    }
}
```

* During `test` phase, JaCoCo instruments the code and generates coverage reports in Jenkins workspace (`target/site/jacoco/index.html`).

### 3. Optional SonarQube Integration

Feed coverage reports into SonarQube for dashboards or quality gates:

```groovy
stage('Coverage Analysis') {
    steps {
        withSonarQubeEnv('SonarQubeServer') {
            sh 'sonar-scanner -Dsonar.projectKey=project -Dsonar.java.coveragePlugin=jacoco'
        }
    }
}
```

* Quality gates can enforce coverage thresholds (e.g., fail pipeline if <40%).

## Common Issues / Errors:

* Coverage reports missing → incorrect Maven plugin configuration or workspace path.
* Unit test failures → coverage report incomplete.
* SonarQube not picking up JaCoCo reports → wrong path or plugin settings.

## Troubleshooting / Fixes:

* Verify JaCoCo plugin is correctly configured in Maven.
* Ensure Jenkins workspace paths match SonarQube report paths.
* Check Maven logs for test failures or compilation issues.
* Open `target/site/jacoco/index.html` to confirm report generation.

## Best Practices / Tips:

* Always generate coverage reports **during build stage** using Maven/Gradle.
* Feed reports to SonarQube for visualization and quality gate enforcement.
* Do not attempt to fail build in Maven based solely on coverage; use **SonarQube quality gates**.
* Developers should validate coverage locally for feature branches; pipeline coverage enforcement applies post-build.
* Keep coverage enforcement thresholds configurable based on project requirements.

```
```

---

# Jenkins – Artifact Storage (Nexus / JFrog) & Secure Credentials (Detailed Explanation)

## Concept / What:
Artifact storage refers to storing build outputs (e.g., JARs, WARs, Docker images) in centralized repositories like **Nexus** or **JFrog Artifactory**. Secure credentials handling ensures repository passwords are encrypted and not exposed in plain text.

---

## Why / Purpose / Use Case in Real-World:
- Provides a **centralized place** to store and manage build artifacts.
- Facilitates **versioning**, traceability, and rollback.
- Enables **CI/CD pipelines** to automatically deploy artifacts to multiple environments (dev/QA/prod).
- Securing passwords ensures automated deployments do not expose sensitive credentials.
- Example: Jenkins builds Maven artifacts and pushes them to Nexus/JFrog; downstream pipelines pull them for staging or production deployment.

---

## How it Works / Steps / Syntax:

### 1. Artifact Deployment

#### Nexus
1. Configure Nexus server with hosted Maven repositories (snapshots/releases).
2. In `pom.xml`, set `distributionManagement` with repository URLs.
3. Configure credentials in `settings.xml` (see secure credentials below).
4. Jenkins deploy stage example:
```groovy
stage('Deploy to Nexus') {
  steps {
    sh 'mvn clean deploy -DaltDeploymentRepository=nexus-repo::default::http://nexus-server/repository/maven-releases/'
  }
}
```

#### JFrog Artifactory
1. Configure Artifactory server with Maven repositories.
2. Use Artifactory Jenkins plugin for integration.
3. Pipeline example:
```groovy
stage('Deploy to Artifactory') {
  steps {
    rtMavenDeployer(
      id: 'deployer-1',
      serverId: 'artifactory-server',
      releaseRepo: 'libs-release-local',
      snapshotRepo: 'libs-snapshot-local'
    )
    rtMavenRun(
      pom: 'pom.xml',
      goals: 'clean install deploy'
    )
  }
}
```

### 2. Secure Credentials (Master & Server Passwords)

#### a) Master Password
1. Generate master password:
```bash
mvn --encrypt-master-password
```
2. Save encrypted master password in `~/.m2/settings-security.xml`:
```xml
<settingsSecurity>
  <master>{ENCRYPTED_MASTER_PASSWORD}</master>
</settingsSecurity>
```
- This password is **not the Nexus/JFrog password**, just a key for encryption/decryption.

#### b) Encrypt Server Password
1. Encrypt actual server password:
```bash
mvn --encrypt-password
```
2. Add to `settings.xml`:
```xml
<servers>
  <server>
    <id>nexus-repo</id>
    <username>jenkins-user</username>
    <password>{ENCRYPTED_SERVER_PASSWORD}</password>
  </server>
</servers>
```
- Maven automatically uses the master password to decrypt it during builds.

#### c) Using in Jenkins Pipeline
- Store `settings.xml` as a **Secret File** in Jenkins Credentials or in **AWS Secrets Manager**.
- Example pipeline usage:
```groovy
withMaven(mavenSettingsConfig: 'Maven_Settings_Credential_ID') {
    sh 'mvn clean deploy'
}
```
- Maven will read the encrypted server password and deploy artifacts securely.

---

## Common Issues / Errors
- Wrong repository URL → deployment fails.
- Credentials mismatch → unauthorized error.
- Network/firewall issues → cannot reach repository.
- `settings.xml` or `settings-security.xml` misconfigured → decryption fails.

---

## Troubleshooting / Fixes
- Verify repository URL and access.
- Ensure correct username and encrypted password.
- Validate `settings.xml` and `settings-security.xml` paths.
- Test connectivity from Jenkins agent to repository.
- Use Maven debug mode: `mvn -X deploy`.

---

## Best Practices / Tips
- Use **snapshot** and **release** repositories separately.
- Version artifacts properly (semantic versioning).
- Always encrypt passwords; never store in plain text.
- Use Jenkins Secret Files or AWS Secrets Manager.
- Keep `settings-security.xml` secure; do not commit to Git.
- Prefer Nexus/JFrog plugins in Jenkins for smoother integration.



---

# Jenkins – AWS and Kubernetes Deployments (Detailed Explanation)

## Concept / What:
Automated deployment of applications from Jenkins to cloud environments: AWS (CodeDeploy, Lambda) and Kubernetes (kubectl, Helm). Direct Java commands can be used but are less practical.

---

## Why / Purpose / Use Case:
- Automates deployment of applications from CI/CD pipelines.
- Ensures consistency, traceability, and rollback capabilities.
- Supports various deployment strategies (rolling, blue/green, canary).
- Example: Jenkins builds a Docker image or Java artifact and deploys to AWS EC2, Lambda, or K8s clusters.

---

## How it Works / Steps / Syntax:

### 1. AWS CodeDeploy
- Pre-requisites: Create S3 bucket, CodeDeploy application, deployment group, and IAM permissions.
- Jenkins pipeline stage:
```groovy
stage('Deploy to AWS CodeDeploy') {
  steps {
    withAWS(credentials: 'aws-credentials-id', region: 'ap-south-1') {
      sh '''
        aws s3 cp target/my-app.war s3://my-app-bucket/
        aws deploy create-deployment \
          --application-name MyApp \
          --deployment-group-name MyApp-Dev \
          --s3-location bucket=my-app-bucket,key=my-app.war,bundleType=zip
      '''
    }
  }
}
```

### 2. AWS Lambda
- Pre-requisites: Create Lambda function, IAM permissions, and package code.
- Jenkins pipeline stage:
```groovy
stage('Deploy to Lambda') {
  steps {
    withAWS(credentials: 'aws-credentials-id', region: 'ap-south-1') {
      sh '''
        zip -r function.zip .
        aws lambda update-function-code \
          --function-name MyLambdaFunction \
          --zip-file fileb://function.zip
      '''
    }
  }
}
```
- Direct Java SDK deployment is possible but less practical due to manual orchestration.

### 3. Kubernetes Deployments
#### a) kubectl
```groovy
stage('Deploy to K8s (kubectl)') {
  steps {
    sh '''
      kubectl apply -f k8s/deployment.yaml
      kubectl apply -f k8s/service.yaml
    '''
  }
}
```
#### b) Helm
```groovy
stage('Deploy to K8s (Helm)') {
  steps {
    sh '''
      helm upgrade --install myapp ./helm-chart --namespace dev --set image.tag=${BUILD_NUMBER}
    '''
  }
}
```

---

## Common Issues / Errors:
- AWS: IAM permissions missing, wrong S3 bucket, agent not running, function name errors.
- K8s: kubeconfig misconfigured, wrong image tag, Helm chart errors, cluster resource limits.

---

## Troubleshooting / Fixes:
- AWS: Verify IAM roles, S3 bucket paths, deployment group targets.
- K8s: Validate kubeconfig, Helm charts (`helm lint`), check pods status (`kubectl get pods`).
- Direct Java SDK: Ensure correct API usage, credentials, and error handling.

---

## Best Practices / Tips:
- AWS: Use blue/green deployments, automate rollback, version artifacts.
- Lambda: Use versions/aliases, test events post-deployment.
- K8s: Use namespaces, separate stages for build/push/deploy, Helm for complex apps.
- Direct SDK deployments are possible but prefer managed services (CodeDeploy/Lambda) for enterprise pipelines.

