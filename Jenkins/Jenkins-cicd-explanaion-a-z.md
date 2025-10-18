# Complete CI/CD Flow for Microservices (with Feature Branches, Bugfixes, and Production Images)

## 1. Git Branching Strategy

* **Feature Branches:** Developers work on individual features in separate branches.
* **Dev Branch:** After PR approval, feature branches are merged into the dev branch. Dev contains the latest code of all features.
* **QA Branch:** Created from dev branch. Contains all feature code. Used for QA testing.
* **UAT Branch:** Created from QA branch. Contains all code from QA. Used for UAT testing.
* **Release Branch:** Created from dev branch after UAT approval for production-ready image creation. Contains all features approved for production.
* **Bugfix Branches:** Created from release branch if bugs are found after production. Developers apply fixes here and raise PRs back to release branch.

**Note:** Branches are lightweight pointers in Git; they do not duplicate source code, they just point to commits.

---

## 2. CI/CD Pipeline Flow

### 2.1 CI Pipeline

1. **Trigger:** PR raised from feature branch to dev branch.
2. **Steps:**

   * Code is checked out from the PR branch.
   * Run build and unit tests.
   * Run static code analysis (e.g., SonarQube).
   * Build Docker image for the feature.
   * Tag Docker image (use feature name + build number or semantic versioning).
   * Push Docker image to AWS ECR or Artifactory.

### 2.2 CD Pipeline for Lower Environments (Dev/QA/UAT)

1. **Trigger:** Manual or automatic (parameterized for environment).
2. **Steps:**

   * Select environment (Dev, QA, UAT).
   * Fetch corresponding Docker image for the environment (feature-specific images for lower environments).
   * Pull Kubernetes manifest files from GitHub.
   * Use variables or Helm `values.yaml` to dynamically configure images and environment-specific settings.
   * Deploy to the selected environment.
   * Approval stage is **not needed** for lower environments.

### 2.3 Production Deployment

1. **Trigger:** Manual selection of Production in the same Jenkins job.
2. **Steps:**

   * Create release branch from dev branch after UAT approval.
   * Jenkins pipeline builds **production-ready Docker image** containing all features.
   * Push production image to dedicated repository (ECR/Artifactory).
   * Select production environment in Jenkins pipeline.
   * Approval stage required: submitter and manager approval configured.
   * Deploy production-ready image to production environment.

**Key:** Single Jenkinsfile handles all environments via parameterization and conditional stages (production vs lower environments).

---

## 3. Versioning Strategy

* **Feature Images:** Use feature name + build number (e.g., `feature-login-45`) for uniqueness.
* **Production Images:** Use semantic versioning (e.g., `v1.0.0`).
* **Bugfix Images:** Increment minor version (e.g., `v1.0.1`) and push updated images after bug fixes.

---

## 4. Bugfix Workflow

1. Create bugfix branch from release branch.
2. Developers apply fixes on bugfix branch.
3. Raise PR from bugfix branch to release branch.
4. CI/CD pipeline triggers:

   * Build updated Docker image.
   * Push to ECR.
   * Deploy to QA environment (using QA testing from release branch or QA environment).
5. After successful QA testing:

   * Merge bugfix branch into release branch.
   * Update dev branch to keep code synchronized.
   * No need to merge back into feature branches.

**Note:** UAT stage is **optional** for bugfixes if the code is already in production.

---

## 5. Handling Multiple Jenkinsfiles

* CI Jenkinsfile: Handles feature branch builds.
* CD Jenkinsfile: Single Jenkinsfile for Dev/QA/UAT/Production.
* Conditional logic in Jenkinsfile:

  * If environment = Production → use production-ready image + approval stage.
  * Else → use feature-specific image and direct deployment.
* No need to maintain separate Jenkinsfiles per environment.
* Each Jenkins job specifies:

  * Git repository URL.
  * Branch to monitor (e.g., dev for CI).
  * Script path (Jenkinsfile in repo).
* PR triggers same Jenkins job; multiple PRs run as separate pipelines without creating multiple jobs.

---

## 6. Docker Image Handling

* Lower environments: Feature-specific Docker images.
* Production: Single production-ready image containing all features.
* Tag images uniquely using semantic versioning or feature names + build number.
* QA/UAT/test environments can pull images dynamically based on environment variables or Helm charts.

---

## 7. Helm / Kubernetes Integration

* Kubernetes deployment manifests can reference images using variables.
* Helm charts recommended:

  * Maintain `values.yaml` per environment.
  * Use environment parameter in Jenkinsfile to select correct `values.yaml`.
  * Deploy images automatically using Helm templates.
* No need to pull artifacts; Docker image contains all necessary application code.

---

## 8. Summary of Key Points

1. Developers work on individual feature branches.
2. PR to dev branch triggers CI to build feature image.
3. QA and UAT environments deploy feature images from dev branch.
4. After all features are approved in UAT:

   * Create release branch from dev.
   * Build production-ready image containing all features.
   * Deploy to production using single Jenkinsfile with approval stage.
5. Bugfix workflow:

   * Bugfix branch from release branch.
   * Fix code, raise PR to release branch.
   * Build new Docker image, deploy to QA.
   * Merge back to release and dev branches.
6. Versioning strategy:

   * Feature images: feature name + build number.
   * Production images: semantic versioning.
   * Bugfix images: minor version increment.
7. Jenkinsfile management:

   * Single CD Jenkinsfile for all environments.
   * Conditional stages for production vs lower environments.
   * CI Jenkinsfile for PR builds.
8. Helm charts used to manage environment-specific variables and image references.
9. Git branches are lightweight pointers; no unnecessary duplication of code.
10. Multiple PRs can trigger the same Jenkins job; each runs as a separate pipeline.

---

This flow ensures **automation, maintainability, and clarity** for microservice deployments, CI/CD pipelines, and production image management.

