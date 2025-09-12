### Quick Revision Version

**Quality & Security Scans:**

* Automated code analysis in Jenkins pipelines.
* Detects bugs, code smells, vulnerabilities, and enforces coding standards.
* Integrate tools: SonarQube, OWASP Dependency Check, Snyk.
* Use Quality Gates to fail builds if thresholds are not met.
* Run scans per commit/pull request; review reports.
* Best practice: scan PRs, automate notifications, keep tools updated.

**SonarQube Integration:**

* Static code analysis tool for code quality and security.
* Steps:

  1. Install SonarQube server.
  2. Install SonarScanner plugin in Jenkins.
  3. Configure server URL & token.
  4. Add scanner in Jenkins Global Tool Configuration.
  5. Run in pipeline (Freestyle: build step, Pipeline: `withSonarQubeEnv { sh 'mvn sonar:sonar' }`).
  6. Review dashboard results.
* Best practice: integrate in PR pipelines, use Quality Gates, exclude generated code, keep versions updated.

---

### Quick Revision Version

**Quality Profiles:**

* Sets of coding rules in SonarQube (no unused variables, empty catch blocks, naming conventions, security rules).
* Can use predefined or custom profiles per project/language.

**Quality Gates:**

* Conditions projects must meet (code coverage ≥ 80%, duplicated code < 3%, no critical bugs, no vulnerabilities).
* Builds fail in Jenkins if thresholds not met.

**Usage:**

* Link Quality Gate to project branch.
* Run SonarQube scan via Jenkins pipeline.
* Ensure rules are realistic and maintain consistent code quality.

**Best Practices:**

* Separate profiles for different languages/projects.
* Review/update rules regularly.
* Enforce gates in CI/CD pipelines for early feedback.

---

### Quick Revision Version

**PR Decoration:**

* Integrates SonarQube results into SCM pull requests (GitHub/GitLab/Bitbucket).
* Inline comments, Quality Gate status, coverage, and vulnerabilities visible in PR.

**Usage:**

* Enable PR Decoration in SonarQube.
* Provide SCM authentication token.
* Configure Jenkins pipeline to checkout PR branch.
* Pass PR parameters to scanner: `sonar.pullrequest.key`, `sonar.pullrequest.branch`, `sonar.pullrequest.base`.
* Run SonarQube scan; results appear in PR.

**Best Practices:**

* Decorate all PRs for early feedback.
* Enforce Quality Gates to block merges with critical issues.
* Limit decoration to active PRs to save CI/CD resources.
* Keep scans lightweight to reduce pipeline runtime.

---

### Quick Revision – Jenkins Static Code Analysis

#### 1. Static Code Analysis

* **What:** Automated check of code for quality, bugs, vulnerabilities without execution.
* **Why:** Detect issues early, maintain coding standards, ensure secure and maintainable code.
* **How:**

  * Integrate SonarQube / OWASP / Snyk in Jenkins.
  * Trigger scan post code checkout.
  * Configure Quality Profiles & Quality Gates.
  * Use PR Decoration for reviewer visibility.
  * Enable incremental scans for large projects.
* **Common Issues:** Long scan time, authentication/plugin issues, false positives.
* **Fixes:** Exclude generated code, incremental/parallel scans, adjust thresholds, update plugins.
* **Tips:** Run scans early, optimize for large codebase, maintain balance between

---

## Quick Revision Version

### Concept / What

* OWASP Dependency Check: SCA tool in Jenkins for detecting vulnerable libraries.

### Why / Purpose

* Stops insecure dependencies from reaching production.
* Mandatory in compliance-driven environments.

### How / Steps

1. Install Jenkins plugin.
2. Configure in Global Tool Configuration.
3. Add pipeline stage:

```groovy
stage('Dependency-Check') {
  steps {
    dependencyCheck scanpath: '.', outdir: 'dependency-check-report'
    dependencyCheckPublisher pattern: 'dependency-check-report/dependency-check-report.xml'
  }
}
```

4. View report in Jenkins build UI.

### Common Issues

* DB update failure (network issue).
* Long scan times.
* Report missing (wrong config).

### Troubleshooting

* Allow Jenkins internet access.
* Run DB update separately.
* Limit scan path.
* Adjust thresholds with `--failOnCVSS`.

### Best Practices

* Update DB nightly.
* Fail build only on **high/critical CVEs**.
* Archive reports for audits.
* Combine with SonarQube/Trivy for end-to-end security.


---

