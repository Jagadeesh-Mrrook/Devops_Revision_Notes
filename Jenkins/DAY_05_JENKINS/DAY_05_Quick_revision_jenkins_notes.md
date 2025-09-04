### Jenkins Source Code Management (SCM) - Quick Revision Notes

**• Concept / What:**
SCM in Jenkins integrates version control systems like Git to pull code for automated builds.

**• Why / Purpose:**

* Builds latest code automatically.
* Supports CI/CD, PR validation, multi-branch workflows.
* Example: Builds Java app on GitHub commit.

**• How / Steps:**

1. Select SCM in Jenkins job.
2. Choose Git, provide URL (HTTPS/SSH), add credentials if private.
3. Triggers:

   * Webhooks: instant build on commit/PR.
   * SCM Polling: periodic check, triggers only on changes.
   * Cron Polling: scheduled builds.
   * Trigger on other jobs: start pipeline after job success.
4. Freestyle vs Pipeline:

   * Freestyle: configure SCM in job.
   * Pipeline: define SCM in Jenkinsfile stored in repo.

**• Common Issues:**

* Wrong URL, missing credentials, network/firewall issues, missing Jenkinsfile.

**• Fixes:**

* Test credentials, check network access, validate Jenkinsfile.

**• Best Practices:**

* Prefer SSH keys, monitor logs, version Jenkinsfile with code, combine webhooks with polling.

---

### Jenkins Git Integration (HTTPS & SSH) - Quick Revision Notes

**• Concept / What:**
Git integration allows Jenkins to pull code from repositories using HTTPS or SSH.

**• Why / Purpose:**

* Secure access to Git repositories.
* Supports public and private repos.
* Example: Pipeline pulls private GitHub repo and builds Java app.
* SSH preferred for enterprise security.

**• How / Steps:**

1. Configure Jenkins job:

   * Select Git, enter repo URL (HTTPS/SSH), add credentials if private.
2. Authentication:

   * HTTPS: username/password or token.
   * SSH: SSH key pair stored in Jenkins Credentials.
3. Test connection:

   * Freestyle: click Test Connection.
   * Pipeline: run checkout step in Jenkinsfile.
4. Pipeline example:

```groovy
git branch: 'main', url: 'git@github.com:user/repo.git', credentialsId: 'my-ssh-cred'
```

**• Common Issues:**

* Authentication failure, network/firewall issues, wrong repo URL, permission denied.

**• Fixes:**

* Verify credentials, test SSH/HTTPS connection, ensure Jenkins server network access.

**• Best Practices:**

* Prefer SSH for private repos.
* Store credentials securely.
* Test repo connection before builds.
* Keep URLs consistent and versioned with Jenkinsfile.

---

### Jenkins GitHub/GitLab/Bitbucket Webhooks - Quick Revision Notes

**• Concept / What:**
Webhooks are HTTP callbacks from GitHub/GitLab/Bitbucket to Jenkins to trigger jobs on repository events.

**• Why / Purpose:**

* Automates CI/CD, triggers builds instantly on push or PR.
* Faster than polling.
* Example: Pipeline builds code automatically on commit or PR.

**• How / Steps:**

1. In Git repo settings, add webhook:

   * Payload URL: `http://<JENKINS_URL>/github-webhook/`
   * Content type: `application/json`
   * Secret: optional security key.
   * Choose events (push, PR, etc.).
2. In Jenkins:

   * Install GitHub/GitLab/Bitbucket plugin.
   * Enable **Build when a change is pushed**.
   * Pipeline jobs: multibranch pipeline listens to webhooks.
3. Test by pushing code or creating PR.

**• Webhook Secret:**

* Shared key to sign payload.
* Verifies authenticity; prevents unauthorized triggers.

**• Common Issues:**

* Jenkins not reachable, wrong URL, secret mismatch, plugin/config issues, permission errors.

**• Fixes:**

* Ensure Jenkins URL is accessible.
* Check delivery logs, plugin, job config, and secret.
* Verify repo permissions.

**• Best Practices:**

* Use HTTPS for Jenkins URL.
* Always use webhook secret.
* Prefer webhooks over polling.
* Monitor webhook delivery logs.
* Combine webhooks with polling as fallback.


---


# Multi-Branch Pipeline in Jenkins

## Quick Revision Version

### • Concept / What

Multi-branch pipeline auto-creates jobs for each branch in repo with a `Jenkinsfile`.

### • Why / Purpose / Use Case

* Supports branching strategies (dev, QA, UAT, main).
* Automates CI/CD per branch.
* Reduces manual job setup.
* Ensures consistency and PR-based builds.

### • How it Works / Steps / Syntax

1. Create Multi-Branch Pipeline job.
2. Connect to SCM repo.
3. Jenkins scans repo → creates jobs for branches with `Jenkinsfile`.
4. Webhook/SCM polling triggers builds.
5. Example: Dev = CI, QA = deploy using Dev artifacts, Main = prod deploy.

### • Common Issues / Errors

* Missing `Jenkinsfile` → branch ignored.
* Bad SCM credentials.
* Webhooks not firing.
* Duplicate artifact versions.

### • Troubleshooting / Fixes

* Always commit valid `Jenkinsfile`.
* Check webhooks, network, credentials.
* Use unique artifact versioning.
* Align branching with CI/CD flow.

### • Best Practices / Tips

* Use multi-branch for CD, single branch for CI.
* Prefer developer-set versions in pom.xml.
* Keep artifacts same across envs.
* Parameterize deployments for CD.
* Don’t rely only on build numbers.

---

# SCM Polling in Jenkins

## Quick Revision Version

### • Concept / What
SCM Polling: Jenkins periodically checks Git repo for changes; triggers build if changes exist.

### • Why / Purpose / Use Case
- Fallback when webhooks fail.
- Firewalled/on-prem repos.
- Controlled, scheduled builds.
- Legacy or predictable CI setups.

### • How it Works / Steps / Syntax
- **Freestyle Job**: Build Triggers → Poll SCM → set cron → triggers if changes.
- **Declarative Pipeline**: `triggers { pollSCM('H/10 * * * *') }` inside Jenkinsfile.
- **Multi-branch Pipeline**: branch indexing scan + per-branch `pollSCM` for fallback.
- Prefer webhooks + polling as backup.

### Cron Examples
- `H/15 * * * *` → every 15 min.
- `H * * * *` → every hour.
- `H 4 * * 1-5` → 4 AM weekdays.

### Common Issues / Errors
- Wrong cron → no polling.
- Invalid credentials/network → polling fails.
- Branch specifier mismatch.
- Polling too frequent → load.
- No Jenkinsfile → branch ignored.

### Troubleshooting / Fixes
- Check **Polling Log**.
- Validate cron & credentials.
- Ensure network access.
- For multibranch, ensure Jenkinsfile exists.
- Combine polling + webhooks for reliability.

### Best Practices / Tips
- Use webhooks primarily; polling as fallback.
- Avoid frequent polling.
- Use `H` to spread load.
- Branch-aware Jenkinsfile logic.
- Monitor polling logs.

---


# Pull Request Builds (GitHub PR Builder) in Jenkins

## Quick Revision Version

### • Concept / What
PR Builds: Jenkins jobs triggered automatically when a developer raises or updates a pull request in a Git repo.

### • Why / Purpose / Use Case
- Automates CI validation before merge.
- Early code quality checks (e.g., SonarQube).
- Reduces manual testing and prevents breaking main/dev branches.
- Supports team feature branch workflows.
- Enforces quality gates before merging.

### • How it Works / Steps / Syntax
- Install plugins: GitHub, PR Builder, Multibranch.
- Create Freestyle or Multibranch Pipeline job.
- Connect repo via token/SSH.
- Enable PR builds:
  - Freestyle: GitHub PR Builder plugin.
  - Multibranch: “Build when PR is opened/updated.”
- Configure webhooks (push-based) or optional SCM polling.
- Jenkinsfile: checkout, build, test, lint, code analysis. Skip deploy in PR builds.

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
- Verify webhook URL.
- Check plugin versions (GitHub, PR Builder, Multibranch).
- Ensure Jenkins has access via token/SSH.
- Jenkinsfile must exist on PR branch.
- Use throttling/deduplication to avoid multiple builds for same PR.

### • Best Practices / Tips
- Run full CI on PRs before merge.
- Use multibranch pipelines to auto-detect PRs and branches.
- Skip deployment stages in PR builds.
- Enable GitHub status checks to block merge if PR fails.
- Keep PR builds lightweight for faster feedback.

