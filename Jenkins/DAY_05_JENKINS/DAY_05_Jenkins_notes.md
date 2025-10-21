### Jenkins Source Code Management (SCM) - Detailed Notes

**• Concept / What:**
SCM in Jenkins is the integration of version control systems (like Git) so Jenkins can pull source code for building, testing, and deploying. It allows Jenkins to track changes, automate builds, and manage code versions.

**• Why / Purpose / Use Case in Real-World:**

* Ensures Jenkins always builds the latest version of code.
* Automates CI/CD workflows by triggering builds on repository events.
* Supports multi-branch development, PR validation, and release management.
* Example: A Jenkins job builds a Java application automatically whenever a developer pushes code to GitHub.

**• How it Works / Steps / Syntax:**

1. In a Jenkins job, select **Source Code Management**.
2. Choose the SCM system (e.g., Git, SVN).
3. Provide repository URL:

   * HTTPS: `https://github.com/user/repo.git`
   * SSH: `git@github.com:user/repo.git` (preferred for security)
4. Add credentials if the repository is private (stored securely in Jenkins Credential Store).
5. Jenkins fetches the code and prepares it for the job or pipeline.

**Triggers / Build Initiation:**

* **Webhooks:** Repository notifies Jenkins immediately on commits or PRs.
* **SCM Polling:** Jenkins checks the repository periodically and triggers a build only if there are changes.
* **Cron Polling:** Builds scheduled at fixed intervals.
* **Build Trigger on Other Jobs:** Start a pipeline after another job completes successfully.

**Freestyle vs Pipeline:**

* Freestyle jobs: SCM configured directly in the job.
* Pipeline jobs: SCM integration defined in a `Jenkinsfile`, which can be stored in the repository along with the code.

**• Common Issues / Errors:**

* Incorrect repository URL.
* Missing or incorrect credentials.
* Network/firewall issues blocking access.
* Jenkinsfile missing in branch for pipeline jobs.

**• Troubleshooting / Fixes:**

* Test credentials manually (HTTPS login or SSH key test).
* Verify Jenkins server can access the repository over the network.
* Check branch access and permissions.
* Validate Jenkinsfile syntax in each branch.

**• Best Practices / Tips:**

* Use SSH keys for secure authentication.
* Version Jenkinsfile alongside application code.
* Monitor logs for SCM-related errors.
* Keep repository URLs, credentials, and branch configurations correct.
* Combine webhooks with polling as a fallback for reliability.

---

### Jenkins Git Integration (HTTPS & SSH) - Detailed Notes

**• Concept / What:**
Git integration in Jenkins allows Jenkins to pull code from Git repositories for building, testing, and deployment. The connection can be done via HTTPS or SSH.

**• Why / Purpose / Use Case in Real-World:**

* Enables secure access to Git repositories.
* Supports both public and private repositories.
* Real-world example: A Jenkins pipeline pulls code from a private GitHub repository and automatically builds and tests a Java application.
* SSH is preferred in enterprise setups for security and easier credential management.

**• How it Works / Steps / Syntax:**

1. **Configure Jenkins Job:**

   * In SCM section, select Git.
   * Enter repository URL:

     * HTTPS: `https://github.com/user/repo.git`
     * SSH: `git@github.com:user/repo.git`
2. **Authentication:**

   * HTTPS: Use username/password or personal access token stored in Jenkins Credentials.
   * SSH: Generate SSH key pair, add public key to the Git repo, store private key in Jenkins Credentials.
3. **Test Connection:**

   * For freestyle jobs: Click **Test Connection** in Jenkins UI.
   * For pipeline jobs: Run a temporary checkout in Jenkinsfile to verify access.
4. **Integration with Pipelines:**

   * Use `git url:` and `credentialsId:` in Jenkinsfile.
   * Example:

     ```groovy
     git branch: 'main', url: 'git@github.com:user/repo.git', credentialsId: 'my-ssh-cred'
     ```

**• Common Issues / Errors:**

* Authentication failure (wrong credentials, missing SSH key).
* Network/firewall blocking Git access.
* Incorrect repository URL format.
* Permission denied on private repos.

**• Troubleshooting / Fixes:**

* Verify credentials in Jenkins Credential Manager.
* Test SSH connection: `ssh -T git@github.com`.
* Confirm HTTPS credentials or personal access token.
* Ensure Jenkins server has network access to the Git repository.

**• Best Practices / Tips:**

* Prefer SSH for private repositories in enterprise environments.
* Store credentials securely in Jenkins.
* Test repository connection before running builds.
* Keep repository URLs consistent and versioned with Jenkinsfile.
* Combine HTTPS/SSH with proper credential management to avoid failures.
----
# 🌩️ Jenkins + AWS Secrets Manager + GitHub SSH Integration Notes

## Overview

These notes explain how to use AWS Secrets Manager to store SSH private keys and use them securely in Jenkins pipelines for GitHub checkout.

---

## 🔹 Plugins Required

* **AWS Secrets Manager Credentials Plugin**

  * Stores AWS Secrets Manager secrets as Jenkins credentials.
* **AWS Secrets Manager Credentials Provider Plugin**

  * Fetches secrets dynamically and provides them to Jenkins plugins/pipeline steps.

## 🔹 Prerequisites

1. **AWS Secrets Manager**

   * Secret Name: `github-ssh-key`
   * Value: Private SSH key (`id_rsa`) contents
2. **Jenkins Plugins Installed** (listed above)
3. **EC2 Jenkins Agent IAM Role**

   * Permissions: `secretsmanager:GetSecretValue`
4. **Jenkins Credential**

   * Type: “Secret from AWS Secrets Manager”
   * Secret ID: `github-ssh-key`
   * Credential ID in Jenkins: `github-ssh-key`

## 🔹 Pipeline Usage

### Example: Checkout Private GitHub Repo

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout Private GitHub Repo') {
            steps {
                // Use AWS Secrets Manager credential
                withAWS(credentials: 'aws-credentials-id') {

                    // Map secret to SSH key credential
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'github-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh '''
                        # Load key into ssh-agent
                        eval "$(ssh-agent -s)"
                        ssh-add $SSH_KEY

                        # Clone private GitHub repo
                        git clone git@github.com:myorg/myrepo.git
                        '''
                    }
                }
            }
        }
    }
}
```

### 🔹 Key Points

* `withAWS(credentials: 'aws-credentials-id')` → Authenticates Jenkins to fetch secrets using IAM role or credentials.
* `withCredentials([sshUserPrivateKey(...)])` → Exposes private key temporarily as `$SSH_KEY`.
* `ssh-agent` + `ssh-add` → Loads private key into memory for Git SSH authentication.
* Secret exists **only during the pipeline stage** and is removed automatically afterward.

### 🔹 Security Advantages

* No permanent storage of private keys on Jenkins nodes.
* Secrets are **never exposed in logs**.
* Supports **automatic IAM role-based access**, avoiding static AWS credentials.
* Reduces risk of human error and accidental leakage.

## 🔹 Snippet Generator

* Use Jenkins **Pipeline Syntax / Snippet Generator** for `withAWS` and `withCredentials` steps.
* Helps avoid syntax errors and remember variable mappings.
* Generates ready-to-use Groovy snippets for pipelines.

---

### 🔹 Workflow Summary

```
AWS Secrets Manager (private key stored)
        ↓ withAWS
Jenkins fetches secret securely
        ↓ withCredentials
Private key exposed as environment variable ($SSH_KEY)
        ↓ ssh-agent
Used for GitHub checkout in pipeline
        ↓ end of stage
Key removed automatically
```

> **Bottom Line:** Plugins handle secret fetching, temporary exposure, and cleanup securely. AWS CLI approach is flexible but requires careful handling of environment variables and temporary files.

---
---


### Jenkins GitHub/GitLab/Bitbucket Webhooks - Detailed Notes

**• Concept / What:**
Webhooks are HTTP callbacks that GitHub, GitLab, or Bitbucket sends to Jenkins whenever an event occurs in the repository (e.g., push, pull request). Jenkins listens for these webhooks to trigger jobs automatically.

**• Why / Purpose / Use Case in Real-World:**

* Automates CI/CD by triggering Jenkins builds instantly on repository events.
* Faster feedback compared to polling mechanisms.
* Example: Jenkins pipeline builds and tests code automatically when a developer opens a PR or pushes a commit.

**• How it Works / Steps / Setup:**

1. **In GitHub/GitLab/Bitbucket:**

   * Go to repository settings → Webhooks → Add webhook.
   * Payload URL: `http://<JENKINS_URL>/github-webhook/` (must be reachable from Git server).
   * Content type: `application/json`.
   * Secret: Optional security string (explained below).
   * Select events to trigger on: push, pull request, etc.

2. **In Jenkins:**

   * Install GitHub (or GitLab/Bitbucket) Integration Plugin if not installed.
   * In the job:

     * Select **GitHub project** and provide repo URL.
     * Enable **Build when a change is pushed to GitHub**.
   * For pipeline jobs, ensure multibranch pipeline or job listens to webhooks.

3. **Test:**

   * Make a commit or open a PR; Jenkins should automatically start the build.

**• Webhook Secret:**

* A secret is a shared key between GitHub and Jenkins.
* Used to sign the payload to verify authenticity.
* GitHub calculates an HMAC hash of the payload using the secret and sends it in the request header.
* Jenkins recalculates the hash and compares it; if they match, the request is valid.
* Ensures only authorized requests trigger builds.

**• Common Issues / Errors:**

* Jenkins not reachable from Git server (network/firewall).
* Incorrect payload URL.
* Secret mismatch between Git and Jenkins.
* Missing plugin or improper job configuration.
* Permissions issues on the repository preventing webhook creation.

**• Troubleshooting / Fixes:**

* Verify Jenkins URL is accessible from the repository.
* Check webhook delivery logs in Git server (response codes, errors).
* Ensure plugin is installed and job is correctly configured.
* Match webhook secret between Git server and Jenkins.
* Validate repository permissions.

**• Best Practices / Tips:**

* Always use HTTPS for Jenkins URL.
* Use webhook secret for security.
* Prefer webhooks over polling for efficiency.
* Monitor webhook delivery logs for debugging.
* Combine webhooks with polling as a fallback if needed.


---


# Multi-Branch Pipeline in Jenkins

## Detailed Explanation Version

### Concept / What

A **Multi-Branch Pipeline** in Jenkins automatically discovers, manages, and executes pipelines for multiple branches in a source code repository. Each branch can have its own Jenkinsfile, allowing different build, test, and deploy stages for different environments (e.g., dev, QA, UAT, prod).

---

### Why / Purpose / Use Case in Real-World

* **Branching Strategies**: Works well with Gitflow or feature-branch workflows where multiple branches exist (dev, qa, uat, main).
* **CI/CD Separation**: CI (build/test/artifact creation) usually runs on `dev`. CD (deployments) can run on `qa`, `uat`, or `prod` branches.
* **Automation**: No need to create multiple jobs manually for each branch.
* **Environment-Specific Pipelines**: Example – `dev` branch pipeline builds and pushes artifacts, while `qa` branch pipeline only deploys.
* **Real-World Example**: In projects where artifacts must be tested in QA/UAT before production, multi-branch ensures consistency and automation.

---

### How it Works / Steps / Syntax

1. **Job Setup**:

   * Create a Jenkins *Multi-Branch Pipeline* job.
   * Point it to a GitHub/GitLab/Bitbucket repository.
   * Jenkins scans repository branches.

2. **Jenkinsfile Per Branch**:

   * Each branch contains its own `Jenkinsfile`.
   * `dev` Jenkinsfile → CI (build, unit tests, artifact creation, push to Nexus/Artifactory).
   * `qa` Jenkinsfile → CD (deploy artifacts from Nexus to QA environment).
   * `uat` Jenkinsfile → CD (deploy artifacts to UAT).
   * `main` Jenkinsfile → Final production deployment.

3. **Webhooks**:

   * GitHub webhooks notify Jenkins when PRs or commits are made.
   * Jenkins automatically detects changes and triggers the appropriate branch pipeline.

4. **Artifact Versioning**:

   * Two methods:

     * **Developer-driven**: Version defined in `pom.xml` by developers. Jenkins uses this version to publish to Nexus.
     * **Pipeline-driven**: Jenkins dynamically sets version using build numbers, timestamps, or branch names.

5. **Build Numbers**:

   * Each pipeline run increases Jenkins build number (`#1`, `#2`, etc.).
   * Build numbers are unique per branch/job.
   * Stored under `JENKINS_HOME/jobs/<job-name>/builds`.

---

### Common Issues / Errors

* **Duplicate Artifacts**: If build numbers alone are used for versioning, accidental deletions or resets can cause conflicts.
* **Webhook Failures**: If webhook secret mismatches or Jenkins is unreachable, pipelines won’t trigger.
* **Wrong Branch Build**: Misconfigured Jenkinsfile may run unnecessary stages (e.g., running CI stages in `qa` branch).
* **Artifact Drift**: Using different artifacts for dev/qa/uat instead of reusing the same tested one.

---

### Troubleshooting / Fixes

* **Artifact Conflicts**: Always prefer developer-specified versioning (`pom.xml`) for consistency.
* **Webhook Issues**: Check GitHub → Webhooks → Delivery logs; ensure Jenkins endpoint is reachable and secret matches.
* **Pipeline Debugging**: Use `echo` statements and `when { branch }` conditions in Jenkinsfile to control stage execution.
* **Build Number Reliability**: Never depend solely on Jenkins build number for artifact versioning. Combine with Git commit hash or `pom.xml` version.

---

### Best Practices / Tips

* Use **developer-set versions** in `pom.xml` for artifacts. Treat Jenkins build numbers only as pipeline run identifiers.
* Always **reuse artifacts** built in `dev` branch for QA/UAT/Prod to ensure consistency.
* **Separate CI/CD**: CI runs on `dev`, CD on `qa/uat/main`.
* Keep Jenkinsfiles DRY (don’t repeat logic) by using shared libraries.
* Use `parameters` in CD pipelines for selecting environment.
* Secure webhooks with **secrets** to avoid unauthorized triggers.
* Clearly document which branch is responsible for build vs deployment.

---

# SCM Polling in Jenkins

## Detailed Explanation Version

### • Concept / What
**SCM Polling** is a mechanism in Jenkins where the server periodically checks a source code repository (GitHub, GitLab, Bitbucket) for new commits or changes. If any changes are detected, the pipeline/job is triggered.

---

### • Why / Purpose / Use Case in Real-World
- **Fallback to webhooks** if push-based events fail.
- **Firewalled/on-premise repos** where webhooks aren’t feasible.
- **Controlled build schedule** instead of triggering on every commit.
- Suitable for teams with **legacy setups** or needing predictable polling intervals.

---

### • How it Works / Steps / Syntax

#### Freestyle Job
1. Configure **Source Code Management → Git** (repo URL, branch, credentials).
2. Under **Build Triggers**, select **Poll SCM**.
3. Enter a cron schedule (e.g., `H/15 * * * *` for every 15 minutes).
4. Jenkins will check for changes and trigger builds only if changes exist.

#### Declarative Pipeline
```groovy
pipeline {
  agent any
  triggers {
    pollSCM('H/10 * * * *')
  }
  stages {
    stage('Build') {
      steps { sh 'mvn clean install' }
    }
  }
}
```

#### Multi-branch Pipeline
- **Branch indexing scan**: Periodically discovers new/removed branches/PRs.
- **Per-branch polling**: `triggers { pollSCM(...) }` in Jenkinsfile for fallback builds.
- **Recommended**: Use webhooks for real-time + polling for fallback.

### Cron Reference
- `H/15 * * * *` → every ~15 minutes.
- `H * * * *` → every hour.
- `H 4 * * 1-5` → 4 AM weekdays.

---

### • Common Issues / Errors
- Incorrect cron syntax → polling not triggered.
- Invalid credentials/network → cannot check repo.
- Branch specifier mismatch → builds not triggered.
- Polling too frequent → load on Jenkins/SCM.
- Multibranch without Jenkinsfile → branch ignored.

---

### • Troubleshooting / Fixes
- Check **Polling Log** in Jenkins job to see if changes were detected.
- Validate cron syntax and SCM credentials.
- Ensure Jenkins can reach the repo (network, proxy, firewall).
- In multibranch, confirm each branch has a valid Jenkinsfile.
- Combine polling with webhooks for reliability.

---

### • Best Practices / Tips
- Prefer **webhooks** for real-time builds; polling only as fallback.
- Avoid frequent polling (e.g., every minute) in production.
- Use `H` in cron expressions to distribute load.
- Keep multibranch Jenkinsfile logic branch-aware (build only where necessary).
- Monitor polling logs to confirm expected behavior.


---

# Pull Request Builds (GitHub PR Builder) in Jenkins

## Quick Revision Version

### • Concept / What
PR Builds: Jenkins jobs triggered automatically when a developer raises or updates a pull request in a Git repo.

### • Why / Purpose / Use Case
- Automate CI validation before merging.
- Catch code quality issues early using tools like SonarQube.
- Reduce manual testing and prevent breaking main/dev branches.
- Supports team feature branch workflows.
- Enforce quality gates before merge.

### • How it Works / Steps / Syntax
- **Install Plugins:** GitHub, GitHub Branch Source, PR Builder.
- **Create Jenkins Job:** Freestyle or Multibranch Pipeline.
- **Connect to GitHub:** Provide repo URL and credentials (token/SSH).
- **Enable PR Builds:**
  - Freestyle: use GitHub PR Builder plugin.
  - Multibranch: enable “Build when PR is opened or updated.”
- **Configure Triggers:** Webhooks on GitHub push PR events; optional SCM polling fallback.
- **Jenkinsfile:** Define stages (checkout, build, test, lint, code analysis). Skip deploy in PR builds.

### • Example (Declarative Pipeline)
```groovy
pipeline {
  agent any
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Build') { steps { sh 'mvn clean install' } }
    stage('Test') { steps { sh 'mvn test' } }
  }
  post {
    success { echo 'PR Build Passed' }
    failure { echo 'PR Build Failed' }
  }
}
```

### • Common Issues / Errors
- Webhook not configured → PR events not triggering builds.
- Invalid credentials → cannot fetch PR branch.
- Jenkinsfile missing on feature branch → build fails.
- Duplicate builds if multiple PR events → deduplication needed.
- PR comment/trigger events not recognized.

### • Troubleshooting / Fixes
- Verify GitHub webhook URL.
- Check plugin versions (GitHub, PR Builder, Multibranch).
- Ensure Jenkins has access via token/SSH.
- Jenkinsfile must exist on PR branch.
- Use throttling/deduplication to avoid multiple builds for same PR.

### • Best Practices / Tips
- Run full CI (build + test) on PRs before merge.
- Use multibranch pipelines to auto-detect PRs and branches.
- Skip deployment stages in PR builds.
- Enable GitHub status checks to block merge if PR fails.
- Keep PR builds lightweight for faster feedback.

