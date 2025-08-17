
# Jenkins Pipeline as Code: Quick Revision

## Key Points
- Declarative Pipeline: Predefined blocks (`pipeline`, `agent`, `stages`, `steps`, `post`), Groovy-based, human-readable, standard pipelines.
- Scripted Pipeline: Full Groovy code, handles complex logic, loops, dynamic behavior, used for advanced scenarios.

## Examples
// Declarative Pipeline
pipeline {
    agent any
    stages {
        stage('Build') { steps { echo 'Building...' } }
        stage('Test') { steps { echo 'Testing...' } }
    }
    post {
        success { echo 'Pipeline succeeded' }
        failure { echo 'Pipeline failed' }
    }
}

// Scripted Pipeline
node {
    stage('Build') { echo 'Building...' }
    stage('Test') { echo 'Testing...' }
    if (env.BRANCH_NAME == 'main') {
        stage('Deploy') { echo 'Deploying...' }
    }
}

## Common Issues
- Mixing styles, syntax errors, misused `node` blocks.

## Tips
- Use Declarative by default, Scripted for advanced cases.
- Keep Jenkinsfile version-controlled.

---


---

### **2️⃣ Quick Revision (`jenkinsfile_quick.md`)**



# Jenkinsfile Creation and Storage in SCM - Quick Revision

## Key Points
- Jenkinsfile: text file defining pipeline, stored in SCM.
- Declarative or Scripted.
- Version-controlled and automatically fetched by Jenkins.

## Steps
1. Create `Jenkinsfile` at repository root.
2. Write pipeline (Declarative/Scripted).
3. Commit and push to SCM.
4. Configure Jenkins job to point to repository.
5. Jenkins executes pipeline automatically.

## Example (Declarative)
pipeline {
    agent any
    stages {
        stage('Build') { steps { echo 'Building...' } }
        stage('Test') { steps { echo 'Testing...' } }
    }
}

## Issues / Tips
- Ensure Jenkinsfile is at repo root.
- Verify correct branch in Jenkins job.
- Keep version-controlled with application code.

---

# Pipeline Parameters - Quick Revision

## Key Points
- Parameters: dynamic input values for pipelines, defined in `parameters` block.
- Types: string, booleanParam, choice, password.
- Access via `params.PARAM_NAME`.
- Supports multibranch pipelines using `env.BRANCH_NAME`.

## Example (Declarative)
pipeline {
    agent any
    parameters {
        string(name: 'APP_VERSION', defaultValue: '1.0')
        booleanParam(name: 'RUN_TESTS', defaultValue: true)
        choice(name: 'ENV', choices: ['dev', 'qa', 'prod'])
    }
    stages {
        stage('Deploy') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        echo 'Deploying to Production'
                    } else {
                        echo "Deploying to ${params.ENV} environment"
                    }
                }
            }
        }
    }
}

## Issues / Tips
- Validate parameter names and types.
- Test different combinations.
- Prefer single Jenkinsfile with branch-aware logic for maintainability.


---

# Jenkins Environment Variables - Quick Revision

## Key Points
- Key-value pairs accessible in pipeline stages.
- Types: built-in (provided by Jenkins) and custom (user-defined).
- Custom variables defined in `environment` block, accessed via `env.VAR_NAME`.
- Can dynamically reference parameters: `VERSION = "${params.APP_VERSION}"`.
- Useful for multi-environment and multibranch pipelines.

## Example
pipeline {
    agent any
    environment {
        DEPLOY_ENV = 'dev'
        VERSION = "${params.APP_VERSION}"
    }
    stages {
        stage('Deploy') {
            steps { echo "Deploying version ${env.VERSION} to ${env.DEPLOY_ENV}" }
        }
    }
}

## Common Tips
- Ensure correct scope; avoid overwriting built-in variables.
- Use descriptive names; test parameter references.
- Prefer environment block for consistency.
- Combine with branch-aware logic for multibranch pipelines.


---

# Jenkins `when` Conditions - Quick Revision Notes

## Key Points
- Conditional execution of stages in Declarative pipelines.
- Supported conditions: branch, parameter, environment variable, custom expression.
- Stage is skipped if the condition evaluates to false.
- Useful for selective deployments, skipping tests, or environment-specific logic.

## Example
pipeline {
    agent any
    parameters {
        booleanParam(name: 'RUN_TESTS', defaultValue: true)
    }
    stages {
        stage('Unit Tests') {
            when { expression { params.RUN_TESTS } }
            steps { echo "Running unit tests..." }
        }
        stage('Deploy to Prod') {
            when { branch 'main' }
            steps { echo "Deploying to production..." }
        }
    }
}

## Common Issues / Tips
- Stage skipped due to branch/parameter/env mismatch.
- Incorrect references in expression.
- Generic: mismatched names or wrongly configured parameters/environment variables.
- Fix: correct configuration, validate syntax, test in feature branch.



---------------------------------------------------------------------------------------


A proper .md file for your Jenkins Error Handling Quick Revision should look like this:

# Jenkins Error Handling - Quick Revision

## Concept & Purpose
- Handle errors gracefully in pipelines.
- Main: `try/catch/finally`, `catchError`, `retry`.
- Supporting: `unstable`, `error`, `post` blocks, `skipStagesAfterUnstable`, `failFast`.

**Purpose:**
- Capture errors, run cleanup, avoid pipeline abortion.
- Retry transient failures.
- Mark stages/builds as unstable.
- Trigger notifications using `post` blocks.

## try/catch/finally
- `try`: execute steps.
- `catch`: handle exceptions, mark build `FAILURE` or `UNSTABLE`.
- `finally`: cleanup actions always.

**Example:**

```
try { sh 'make build' }
catch(Exception e) { currentBuild.result = 'FAILURE' }
finally { echo 'Cleanup' }
```



## catchError
- Marks stage/build as `UNSTABLE` or `FAILURE` without aborting.
- Declarative pipelines friendly.

**Example:**


```
catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
sh 'make test'
}
```


## retry
- Retries a stage multiple times if it fails.
- Marks build FAILED if max retries exceeded.

**Example:**

```
retry(3) { sh 'make build' }
```

## Supporting Mechanisms
- `unstable("Some tests failed")` → marks stage/build unstable.
- `error("Build failed")` → fails stage/build immediately.
- `post` blocks: run actions after `success`, `failure`, `unstable`, `always`.
- Pipeline options:
  - `skipStagesAfterUnstable()` → skips stages after unstable.
  - `failFast true` → aborts all parallel stages if any fails.

## Common Issues & Best Practices
**Common Issues:**
- Stage fails due to misconfigured branch, parameters, or env vars.
- Syntax errors in `try/catch`, `catchError`, `retry`.
- Cleanup skipped if `finally` missing.
- Retry exceeds max attempts.

**Best Practices:**
- Wrap critical stages with `try/catch` or `catchError`.
- Use `finally` for cleanup.
- Combine `retry` with error handling.
- Use `post` blocks for notifications.
- Keep retry counts minimal, log all errors clearly.

---



