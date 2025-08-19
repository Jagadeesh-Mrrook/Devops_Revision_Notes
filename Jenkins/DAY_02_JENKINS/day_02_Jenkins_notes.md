
# Jenkins Pipeline as Code: Declarative vs Scripted Pipelines

## Concept
- Declarative Pipeline: Structured pipeline using predefined blocks (`pipeline`, `agent`, `stages`, `steps`, `post`). Groovy-based, human-readable, ideal for standard pipelines.
- Scripted Pipeline: Full Groovy code, allows complex logic, loops, dynamic behavior; used for advanced scenarios.

## Purpose / Use Case
- Declarative: Build CI/CD pipelines quickly, readable by team members, version-controlled in SCM.
- Scripted: Advanced workflows like dynamic node selection, multiple branch handling, conditional execution, or custom logic.

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

## Common Issues / Errors
- Mixing declarative and scripted syntax incorrectly.
- Syntax errors: missing braces, indentation mistakes.
- Scripted pipelines may fail silently if `node` blocks are misused.

## Troubleshooting / Fixes
- Use Jenkins “Pipeline Syntax” snippet generator to validate code.
- Test complex scripted logic in sandbox environment.
- Review logs carefully for stage-specific failures.

## Best Practices / Tips
- Prefer Declarative for standard pipelines.
- Use Scripted only for advanced scenarios.
- Keep Jenkinsfile version-controlled.
- Avoid mixing styles in the same pipeline.

---

# Jenkinsfile Creation and Storage in SCM - Detailed Notes

## Concept
- Jenkinsfile is a text file defining a Jenkins pipeline.
- Can be Declarative or Scripted.
- Stored in source code repository (SCM) alongside application code.
- Enables version-controlled, reproducible pipelines.

## Purpose / Use Case
- Track pipeline changes with application code.
- Supports branch-specific pipelines (dev, QA, prod).
- Collaborate safely across multiple developers.
- Jenkins fetches Jenkinsfile automatically during job execution.

## How it Works / Steps
1. Create `Jenkinsfile` at repository root.
2. Write Declarative or Scripted pipeline inside it.
3. Commit and push to SCM (GitHub, GitLab, Bitbucket, etc.).
4. Configure Jenkins pipeline job to point to repository URL.
5. Jenkins fetches and executes the pipeline.

## Example (Declarative)
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

## Common Issues / Errors
- Jenkinsfile not at repository root → Jenkins cannot detect.
- Incorrect branch configuration → wrong Jenkinsfile executed.
- SCM credentials missing → pipeline fails to fetch Jenkinsfile.

## Troubleshooting / Fixes
- Ensure Jenkinsfile is in repository root.
- Verify branch name matches Jenkins job configuration.
- Use proper Jenkins credentials for SCM access.

## Best Practices / Tips
- Keep Jenkinsfile versioned with application code.
- Prefer Declarative pipelines unless advanced logic is required.
- Use parameterized Jenkinsfiles for multi-environment setups.

---

# Pipeline Parameters - Detailed Notes

## Concept
- Parameters allow pipelines to accept **dynamic input values** at runtime.
- Defined in the `parameters` block in a Jenkinsfile.
- Enable pipelines to be **reusable and configurable** without modifying the Jenkinsfile.

## Purpose / Use Case
- Trigger pipelines with different configurations: build environment, version numbers, deployment targets.
- Reduce duplication by using the same pipeline for multiple scenarios.
- Useful in **multibranch pipelines** or **promoted builds**.
- Supports dynamic pipeline execution across environments (dev, QA, prod) using the same Jenkinsfile.

## How it Works / Steps
1. Define a `parameters` block at the top of the Declarative pipeline.
2. Specify parameter types:
   - `string`: free text input
   - `booleanParam`: true/false
   - `choice`: select from predefined options
   - `password`: secret value
3. Access parameters inside stages using `params.PARAM_NAME`.
4. Combine with `env.BRANCH_NAME` to handle multibranch pipelines dynamically.

## Example (Declarative)
pipeline {
    agent any
    parameters {
        string(name: 'APP_VERSION', defaultValue: '1.0', description: 'Application version to build')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run unit tests')
        choice(name: 'ENV', choices: ['dev', 'qa', 'prod'], description: 'Deployment environment')
    }
    stages {
        stage('Build') {
            steps { echo "Building version ${params.APP_VERSION}" }
        }
        stage('Test') {
            when { expression { params.RUN_TESTS } }
            steps { echo 'Running tests...' }
        }
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

## Common Issues / Errors
- Parameter names mismatched in pipeline code.
- Default values unsuitable, causing unexpected behavior.
- Boolean or choice parameters misused in expressions.
- Confusion between branch-specific Jenkinsfiles vs single Jenkinsfile behavior.

## Troubleshooting / Fixes
- Validate parameter names exactly match usage in pipeline.
- Test different parameter combinations.
- Use descriptive names and comments.
- For multibranch pipelines, verify correct branch logic using `env.BRANCH_NAME`.

## Best Practices / Tips
- Use parameters to make pipelines reusable across branches/environments.
- Keep parameter types simple and clear.
- Avoid overcomplicating pipelines with too many parameters.
- **Industry standard**: use **single Jenkinsfile with branch-aware logic** and parameters rather than separate Jenkinsfiles per branch, unless workflows differ drastically.

---

# Jenkins Environment Variables - Detailed Notes

## Concept
- Environment variables are **key-value pairs** accessible across Jenkins pipeline stages.
- They can be **built-in** (provided by Jenkins) or **custom-defined** by the user.
- Used to **pass configurations, dynamic values, or credentials** across the pipeline.

## Purpose / Use Case
- Share information between stages without hardcoding, e.g., build version, workspace path, or server URLs.
- Control pipeline behavior dynamically using environment variables like deployment environment.
- Make pipelines reusable across branches and environments.
- Combine with **pipeline parameters** to supply dynamic values at runtime.
- Keep sensitive values secure by using **Jenkins credentials** instead of plain text.

## How it Works / Steps / Syntax
1. **Built-in variables**: Automatically available, e.g., `env.BRANCH_NAME`, `env.BUILD_NUMBER`, `env.WORKSPACE`.
2. **Custom environment variables**:
   - Declarative pipeline: define in `environment` block.
   - Scripted pipeline: define using `env.VAR_NAME = 'value'`.
3. **Dynamic parameter usage**:
   - You can assign a parameter value to a custom environment variable:
     ```groovy
     environment {
         VERSION = "${params.APP_VERSION}"
     }
     ```
4. Access variables in stages using `env.VAR_NAME`.

## Example (Declarative)
pipeline {
    agent any
    parameters {
        string(name: 'APP_VERSION', defaultValue: '1.0', description: 'Application version')
    }
    environment {
        DEPLOY_ENV = 'dev'
        APP_NAME = 'MyApp'
        VERSION = "${params.APP_VERSION}"
    }
    stages {
        stage('Build') {
            steps { echo "Building ${env.APP_NAME} version ${env.VERSION}" }
        }
        stage('Deploy') {
            steps { echo "Deploying to environment: ${env.DEPLOY_ENV}" }
        }
    }
}

## Common Issues / Errors
- Variables not scoped correctly → unavailable in some stages.
- Accidentally overwriting built-in variables.
- Incorrect reference to parameters in environment block.
- Confusion between parameter values and environment variables in multibranch pipelines.

## Troubleshooting / Fixes
- Always access variables with `env.VAR_NAME`.
- Use unique names for custom variables to avoid conflicts.
- Test variable resolution in each stage.
- Ensure correct syntax when referencing parameters.

## Best Practices / Tips
- Use environment variables for **dynamic or reusable values**.
- Keep sensitive values in **Jenkins credentials**, not in plain text.
- Use descriptive names for clarity.
- Prefer `environment` block in Declarative pipelines for consistency.
- Combine with parameters and `env.BRANCH_NAME` for multibranch flexibility.
- Avoid hardcoding environment-specific values directly in Jenkinsfile.

---

# Quick Revision Notes - Environment Variables

## Key Points
- Key-value pairs available to pipeline stages.
- Types: built-in (Jenkins-provided) and custom (user-defined).
- Custom variables defined in `environment` block; accessed via `env.VAR_NAME`.
- Can dynamically reference parameters: `VERSION = "${params.APP_VERSION}"`.
- Useful for multi-environment and multibranch pipelines.

## Example (Declarative)
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

## Issues / Tips
- Ensure correct scope; don’t overwrite built-in variables.
- Use descriptive names; test parameter references.
- Prefer environment block for consistency.
- Combine with branch-aware logic for multibranch pipelines.

---

# Jenkins `when` Conditions - Detailed Notes

## Concept
- `when` conditions allow **conditional execution of stages** in Declarative pipelines.
- Stage executes only if the condition evaluates to **true**, otherwise it is **skipped**.
- Supported conditions: branch, parameters, environment variables, or custom expressions.

## Purpose / Use Case
- Run deployment stage only on specific branches (e.g., `main` or `production`).
- Skip test stages when a parameter indicates “skip tests.”
- Execute stages only for specific environments or pull requests.
- Reduce unnecessary builds and improve pipeline efficiency.

## How it Works / Steps / Syntax
1. Add a `when` block inside a stage.
2. Common conditions:
   - `branch 'branch_name'` → stage runs only on this branch.
   - `expression { ... }` → custom Groovy boolean expression.
   - `environment name: 'VAR', value: 'VALUE'` → based on environment variable.
   - `beforeAgent true` → evaluate before allocating an agent.
   - `anyOf { ... } / allOf { ... }` → combine multiple conditions.
3. Stage is skipped automatically if the `when` condition evaluates to false.

## Example
pipeline {
    agent any
    parameters {
        booleanParam(name: 'RUN_TESTS', defaultValue: true)
    }
    stages {
        stage('Build') {
            steps { echo "Building..." }
        }
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

## Common Issues / Errors
- Stage silently skipped → misunderstanding the condition logic.
- Branch name mismatch in `branch` condition.
- Parameter or environment variable referenced incorrectly in `expression`.
- **Generic common issues across pipeline concepts:** mismatched names or wrongly configured parameters/environment variables.

## Troubleshooting / Fixes
- Verify branch names, parameter names, and environment variables.
- Correct any syntax errors in `when` blocks.
- Use `echo` statements in `expression` to debug logic.
- **Generic fixes:** correct mismatches or misconfigurations, ensure proper references.

## Best Practices / Tips
- Use `when` to **avoid unnecessary stage execution**.
- Prefer `expression` for custom conditions.
- Keep conditions simple and readable.
- Combine with parameters and environment variables for flexible pipelines.
- Test in a feature branch before merging into main pipeline.

---

# Jenkins Error Handling - Concept & Purpose

## Concept
- Jenkins pipelines handle errors to **manage failures gracefully** and avoid abrupt pipeline stops.
- Main mechanisms: `try/catch/finally`, `catchError`, `retry`.
- Supporting mechanisms: `unstable`, `error`, `post` blocks, pipeline options like `skipStagesAfterUnstable` or `failFast`.

## Purpose / Use Case
- Capture errors in build, test, or deploy stages.
- Run cleanup or rollback steps even if a stage fails (`finally` block).
- Avoid pipeline abortion for non-critical failures.
- Retry transient failures automatically (`retry` block).
- Mark stages/builds as unstable without stopping the pipeline (`catchError` or `unstable`).
- Trigger notifications or other actions after success/failure/always using `post` blocks.


# Jenkins Error Handling - try/catch/finally

## How it Works
- `try`: execute steps/commands.
- `catch`: handle exceptions; inspect logs, mark build status, or take custom actions (ignore minor warnings, mark unstable, fail build).
- `finally`: run cleanup actions regardless of success or failure.

## Example

stage('Build') {
    steps {
        script {
            try {
                sh 'make build'
            } catch (Exception e) {
                if (e.toString().contains("minor warning")) {
                    echo "Minor warning, marking build as UNSTABLE"
                    currentBuild.result = 'UNSTABLE'
                } else {
                    echo "Critical error: ${e}"
                    currentBuild.result = 'FAILURE'
                }
            } finally {
                echo "Cleanup actions executed"
            }
        }
    }
}




---

### **Block 3: catchError**

# Jenkins Error Handling - catchError

## How it Works
- Marks a stage or build as `UNSTABLE` or `FAILURE` without aborting the pipeline.
- Useful for simple declarative error handling without scripting.

## Example

stage('Test') {
    steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
            sh 'make test'
        }
    }
}



---

### **Block 4: retry**

# Jenkins Error Handling - retry

## How it Works
- Retries a step/stage multiple times if it fails.
- Marks stage/build as **FAILED** if maximum retries are exceeded.
- Useful for transient failures like network issues or flaky tests.

## Example

stage('Build') {
    steps {
        retry(3) {
            sh 'make build'
        }
    }
}



---

### **Block 5: Supporting Mechanisms**

# Jenkins Error Handling - Supporting Mechanisms

- `unstable("Some tests failed")` → marks stage/build as unstable without stopping the pipeline.
- `error("Build failed due to XYZ")` → immediately fails stage/build.
- `post` blocks → execute actions based on build status:

post {
    success { echo "Build succeeded" }
    failure { echo "Build failed" }
    unstable { echo "Build unstable" }
    always { echo "Cleanup actions executed" }
}
- Pipeline options:
  - `skipStagesAfterUnstable()` → skips stages after an unstable stage.
  - `failFast true` → aborts all parallel stages if any stage fails.



---

### **Block 6: Common Issues, Troubleshooting, Best Practices**

# Jenkins Error Handling - Common Issues, Troubleshooting, Best Practices

## Common Issues / Errors
- Stage fails due to mismatched branch, parameter, or environment variable names.
- Syntax errors in `try/catch/finally`, `catchError`, or `retry` blocks.
- Misusing `catchError` parameters (`buildResult`, `stageResult`).
- Cleanup steps skipped if `finally` is missing.
- Retry exceeding max attempts → stage/build fails.
- Generic: wrong configuration or misreference of variables/parameters.

## Troubleshooting / Fixes
- Always wrap critical stages with `try/catch` or `catchError`.
- Verify correct syntax and proper references (`env.VAR_NAME`, `params.PARAM_NAME`).
- Use `finally` for cleanup tasks.
- Use logging (`echo`) to debug errors and retry behavior.
- Correct mismatches or misconfigurations for variables, parameters, or branch names.
- Ensure retry counts are reasonable for transient failures only.

## Best Practices / Tips
- Use `catchError` for simple Declarative error handling.
- Use `try/catch/finally` for advanced control or cleanup.
- Combine `retry` with error handling for transient failures.
- Use `post` blocks for notifications, cleanup, or reporting.
- Avoid hiding critical failures; always log errors clearly.
- Test error handling logic in feature branches before production.
- Keep retry counts minimal to prevent unnecessarily long pipelines.


---

# Parallel Stages Execution (Declarative) — with failFast

1. Concept:
   - Run multiple, independent stages at the same time using the `parallel` directive in Declarative Pipelines.
   - Goal: reduce total pipeline time by executing work concurrently.
   - `failFast` behavior: stop other parallel branches early when one fails (to save time/resources).

2. Purpose / Use Case in Real-World:
   - Speed up CI: run unit tests, integration tests, lint, and security scans in parallel.
   - Matrix-like checks: run on multiple OS/JDK/Node versions in parallel.
   - Build/test multiple services or modules concurrently.
   - Fail fast when one failure makes remaining work irrelevant (e.g., compile or unit tests failing).

3. How it Works / Steps / Syntax:
   - Basic Declarative parallel (each inner stage runs concurrently):

       pipeline {
         agent any
         stages {
           stage('Quality Gates') {
             parallel {
               stage('Unit Tests') {
                 steps { sh 'make test-unit' }
               }
               stage('Lint') {
                 steps { sh 'npm run lint' }
               }
               stage('Security Scan') {
                 steps { sh 'make scan' }
               }
             }
           }
         }
       }

   - Fail-fast patterns commonly used in real projects:

     a) Pipeline-wide fail-fast for all parallels (Declarative):

       pipeline {
         agent any
         options { parallelsAlwaysFailFast() }   // abort sibling branches when one fails
         stages {
           stage('Checks') {
             parallel {
               stage('UT') { steps { sh 'make test' } }
               stage('Lint') { steps { sh 'make lint' } }
             }
           }
         }
       }

     b) Hybrid (scripted-style map) to enable `failFast: true` inside a Declarative stage:

       pipeline {
         agent any
         stages {
           stage('Parallel (failFast)') {
             steps {
               script {
                 parallel(
                   'Unit':        { sh 'make test-unit' },
                   'Integration': { sh 'make test-integration' },
                   'Lint':        { sh 'npm run lint' },
                   failFast: true
                 )
               }
             }
           }
         }
       }

     c) Matrix with fail-fast (Declarative):

       pipeline {
         agent any
         stages {
           stage('Matrix Tests') {
             matrix {
               axes {
                 axis { name 'OS';  values 'linux', 'windows' }
                 axis { name 'JDK'; values '11', '17' }
               }
               options { failFast true }  // stop siblings on failure
               stages {
                 stage('Run') {
                   steps { echo "Testing on ${OS} JDK ${JDK}" }
                 }
               }
             }
           }
         }
       }

   - Key notes:
     - Use `parallelsAlwaysFailFast()` when you want a blanket fail-fast across all parallel branches.
     - Use the `script { parallel(..., failFast: true) }` map style when you need per-stage fail-fast in a Declarative pipeline.
     - `matrix { options { failFast true } }` is supported for matrix executions.

4. Common Issues / Errors:
   - Not enough executors/agents → branches queue or starve (slow or stuck builds).
   - `failFast` placed incorrectly (e.g., inside `steps` for Declarative `parallel { }`) → has no effect.
   - Shared workspace collisions (parallel branches writing to same files/dirs) → flaky or corrupted outputs.
   - Label mismatch (e.g., stage requires `windows` agent but node unavailable) → branch stuck or fails to schedule.
   - Mixed logs from parallel branches → hard to read failures.
   - Cross-branch dependencies (stage B needs output from stage A) → parallelization causes failures/races.

5. Troubleshooting / Fixes:
   - Capacity: ensure sufficient executors/agents; right-size the number of parallel branches.
   - Fail-fast placement:
     • Use `options { parallelsAlwaysFailFast() }` pipeline-wide, or
     • Use `script { parallel(..., failFast: true) }` for a single parallel block,
     • Use `matrix { options { failFast true } }` for matrix builds.
   - Isolation: use separate dirs (`dir('path')`) or share via `stash/unstash` instead of writing to the same workspace.
   - Labeling: set correct `agent { label 'linux' }`, `agent { label 'windows' }` per branch if needed.
   - Logging: enable `timestamps()`, prefix logs with branch names (`echo "[UT] ..."`) for clarity.
   - Validate syntax with Pipeline Syntax/Replay; upgrade Pipeline plugins if known parallel/fail-fast bugs affect behavior.

6. Best Practices / Tips:
   - Only parallelize truly independent work; avoid shared mutable state.
   - Prefer `parallelsAlwaysFailFast()` for fast feedback pipelines where partial results aren’t needed.
   - If you need a full report (e.g., all OS/versions), avoid fail-fast so every branch completes.
   - Use descriptive stage names for better visibility in the UI.
   - Build once, then `stash` artifacts and `unstash` in parallel branches to avoid rebuilding.
   - Add reasonable `timeout` to each branch to prevent hung tasks from blocking the pipeline.


---

# Jenkins Declarative Pipeline — Agent Directive (Part 1)

## What is `agent`?
- `agent` is a directive in Declarative Pipelines that defines **where/how** the pipeline or a stage runs (which worker node or container).

## Where can you define it?
- Pipeline level (global): applies to all stages unless overridden.
- Stage level: overrides global for that stage.

---

## Agent Types

### 1) agent any
- Runs on any available Jenkins worker node.

Example (groovy):
pipeline {
  agent any
  stages {
    stage('Build') {
      steps { echo 'Running on any available agent' }
    }
  }
}

---

### 2) agent none
- No global agent; each stage must define its own agent.
- Use when different stages need different environments.

Example (groovy):
pipeline {
  agent none
  stages {
    stage('Build') {
      agent any
      steps { echo 'Build on any agent' }
    }
    stage('Test') {
      agent { label 'test-node' }
      steps { echo 'Test on test-node' }
    }
  }
}

# Jenkins Declarative Pipeline — Agent Directive (Part 2)

### 3) agent { label '...' }
- Run only on nodes that have a matching label (labels are set on nodes in Jenkins).

Example (groovy):
pipeline {
  agent { label 'linux' }
  stages {
    stage('Build') {
      steps { echo 'Running on linux-labeled node' }
    }
  }
}

---

### 4) agent { docker { image '...' } }
- Run inside a temporary Docker container (tools come from the image).
- Requires Docker installed on the Jenkins agent host.

Example (groovy):
pipeline {
  agent {
    docker {
      image 'maven:3.9.6-eclipse-temurin-17'
      // optional: args '-v /tmp/.m2:/root/.m2'
    }
  }
  stages {
    stage('Build') {
      steps {
        sh 'mvn -v'
        sh 'mvn clean verify'
      }
    }
  }
}

---

### 5) agent { dockerfile { ... } }
- Build a custom image from a Dockerfile in your repo, then run inside it.

Example (groovy):
pipeline {
  agent {
    dockerfile {
      filename 'Dockerfile'
      dir 'ci-image'
      // optional: additionalBuildArgs '--no-cache'
    }
  }
  stages {
    stage('CI') {
      steps { sh 'make test' }
[O    }
  }
}
# Jenkins Declarative Pipeline — Agent Directive (Part 3)

## Special Cases (for awareness)
- Docker-outside-of-Docker (DooD): mount host docker socket into the job/container to run docker builds.
- Docker-in-Docker (DinD): run a docker daemon inside a container (less common in Jenkins).
- If Jenkins/agents are on EC2/VM (not containerized), you typically don’t need DinD/DooD—just install Docker on the node.

---

## Real-World Usage
- agent any → simple jobs, no special environment.
- agent none + per-stage agents → heterogeneous pipelines (build on linux, test on windows, deploy on deploy-node).
- agent { label '...' } → ensure specific tool/OS/hardware (gpu, windows, docker-enabled).
- agent { docker image } → isolated, repeatable toolchains without installing on nodes.
- agent { dockerfile } → fully customized CI environment.

---

## Best Practices / Tips
- Prefer labels to route workloads (e.g., linux, windows, docker, gpu).
- Use docker agents to avoid dependency conflicts and speed setup.
- Keep images lean (cache deps, pin versions).
- With agent none, ensure every stage has an agent.
- Document node labels and keep them up to date.

---

## Common Issues / Errors
- Label not found or node offline → builds stuck in queue.
- Using docker agent but Docker not installed on node → container start fails.
- agent none without stage-level agent → “No node assigned”.
- Overweight images → slow pull/start times.

## Troubleshooting / Fixes
- Verify labels in Manage Nodes; check executors/online status.
- Install/verify Docker service on agents for docker agents.
- Add/adjust stage-level agents when using agent none.
- Optimize images (multi-stage builds, cache dependencies).
---
