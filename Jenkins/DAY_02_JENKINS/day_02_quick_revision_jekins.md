
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


# Quick Revision — Parallel Stages (Declarative) + failFast

1. Concept:
   - Run independent stages at the same time using `parallel`.
   - Fail fast = stop sibling branches when one fails.

2. Purpose / Real-World Use:
   - Cut CI time (UT + IT + Lint + Security together).
   - Run on multiple OS/JDK in parallel.
   - Save resources by aborting other branches after a critical failure.

3. How it Works / Syntax (pick one):
   - Basic parallel:

       stage('Quality Gates') {
         parallel {
           stage('UT'){ steps { sh 'make test' } }
           stage('Lint'){ steps { sh 'make lint' } }
         }
       }

   - Pipeline-wide fail-fast (Declarative):

       options { parallelsAlwaysFailFast() }

   - Hybrid (inside Declarative using scripted map):

       steps {
         script {
           parallel(
             'Unit': { sh 'make test-unit' },
             'IT':   { sh 'make test-int' },
             failFast: true
           )
         }
       }

   - Matrix fail-fast:

       matrix {
         options { failFast true }
         /* axes + stages ... */
       }

4. Common Issues:
   - Insufficient executors/agents → branches queue or hang.
   - Misplaced `failFast` → no effect.
   - Workspace/file collisions across branches.
   - Wrong agent labels → branch can’t schedule.
   - Mixed logs → hard to debug.

5. Troubleshooting / Fixes:
   - Ensure capacity; limit parallel width if needed.
   - Use correct fail-fast form: `parallelsAlwaysFailFast()` or `parallel(..., failFast: true)` or `matrix { options { failFast true } }`.
   - Isolate work (`dir()`, `stash/unstash`); avoid shared writes.
   - Set correct `agent { label '...' }`.
   - Add `timestamps()` and clear log prefixes.

6. Best Practices:
   - Parallelize only independent work.
   - Use fail-fast when partial results are not needed; avoid it when you need full matrix results.
   - Name branches clearly; add per-branch `timeout`.


---

# Jenkins Agent Directive - Quick Revision Notes

## 1. Purpose of Agent
- Defines **where** (which node/worker) the pipeline or stage will run.
- Can be applied at:
  - **Pipeline level** → applies to the whole pipeline.
  - **Stage level** → overrides global agent for that stage.

---

## 2. Types of Agents
### a) `any`
- Runs pipeline/stage on **any available Jenkins node**.
- Most commonly used default.

### b) `none`
- Means **no global agent** is assigned.
- You **must specify agent** at the stage level.
- Useful when different stages need **different nodes/tools**.

### c) `label`
- Assigns jobs to nodes with a specific **label**.
- Example: `agent { label 'linux' }`
- Real-time use case → run on a node that has required dependencies (like Maven, Docker).

### d) `docker`
- Runs the stage inside a **Docker container**.
- Useful if tools should not be installed on Jenkins nodes.
- Jenkins pulls the Docker image, runs commands, and removes container after completion.

### e) `dockerfile`
- Similar to `docker`, but Jenkins builds the Docker image using a `Dockerfile` before running.
- Good for **custom environments**.

---

## 3. Key Points
- **Global Agent** (`pipeline { agent ... }`) → applies to all stages unless overridden.
- **Stage-specific Agent** (`stage { agent ... }`) → useful when:
  - Different tools needed per stage.
  - Workload separation across nodes.

---

## 4. Real-Time Examples
- `agent any` → General pipeline where no node restriction is needed.
- `agent none` + stage-level agents → Build on Linux node, test on Windows node.
- `agent { label 'docker' }` → Run pipeline only on a node with Docker installed.
- `agent { docker 'maven:3.8.1' }` → Run build inside Maven container.

---

## 5. Common Mistakes
- Using `none` at pipeline level but forgetting stage-level agent → pipeline fails.
- Not configuring correct node labels → jobs stuck in queue.
- Docker agent without Docker installed on node → job fails.

---

