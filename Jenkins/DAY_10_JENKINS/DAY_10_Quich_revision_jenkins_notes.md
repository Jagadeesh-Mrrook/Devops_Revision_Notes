## ⚡ Quick Revision: Jenkins Shared Library (Declarative Pipeline)

### 🔹 Concept / What

A **Jenkins Shared Library** is a centralized repository that stores reusable pipeline code (like stages, functions, or scripts) for multiple Jenkinsfiles.

---

### 🔹 Why / Purpose / Use Case

* Avoids code duplication across pipelines.
* Standardizes CI/CD workflows (build, test, deploy).
* Makes maintenance easier — update once, apply everywhere.
* Allows dynamic logic and reusable utilities.

---

### 🔹 How / Structure

```
jenkins-shared-library/
├── vars/
│   ├── buildPipeline.groovy
│   ├── deployPipeline.groovy
│   └── utilities.groovy
├── resources/
│   ├── build.sh
│   ├── test.sh
│   └── deploy.sh
└── src/ (optional)
```

**vars/** → Pipeline entry points (each file = callable function)

**resources/** → Shell scripts, YAMLs, templates.

**utilities.groovy** → Common helper functions (ex: execute resource scripts).

---

### 🔹 Sample: utilities.groovy

```groovy
def runScript(scriptName) {
  def scriptContent = libraryResource("${scriptName}")
  writeFile file: "${scriptName}", text: scriptContent
  sh "chmod +x ${scriptName} && ./${scriptName}"
}
```

---

### 🔹 Sample: buildPipeline.groovy

```groovy
def call() {
  pipeline {
    agent any
    stages {
      stage('Build') {
        steps {
          script {
            utilities.runScript('build.sh')
          }
        }
      }
    }
  }
}
```

### 🔹 Sample: deployPipeline.groovy

```groovy
def call() {
  pipeline {
    agent any
    stages {
      stage('Deploy') {
        steps {
          script {
            utilities.runScript('deploy.sh')
          }
        }
      }
    }
  }
}
```

---

### 🔹 Common Issues / Fixes

| Issue                         | Cause                                   | Fix                                          |
| ----------------------------- | --------------------------------------- | -------------------------------------------- |
| `No such property: utilities` | File name mismatch or not under `vars/` | Ensure file is `utilities.groovy` in `vars/` |
| `Resource not found`          | Wrong path to resource file             | Use correct relative path under `resources/` |
| Permission denied             | Script not executable                   | Add `chmod +x` before executing              |

---

### 🔹 Best Practices

* Keep **helper logic** in `utilities.groovy` to avoid repetition.
* Separate each pipeline stage logic (`buildPipeline`, `deployPipeline`, etc.).
* Use **resources** for shell scripts or config templates.
* Maintain proper naming (lowerCamelCase for vars, snake_case for scripts).
* Always test libraries via `Jenkinsfile` before pushing changes.

---

### ✅ Real-World Use Case

* `buildPipeline.groovy` → compiles code, runs tests.
* `deployPipeline.groovy` → deploys to staging/prod.
* `utilities.groovy` → executes scripts dynamically.
* Scripts (`build.sh`, `deploy.sh`) stored in `resources/` for reuse.

---

### 🔹 Troubleshooting Quick Tips

* Re-run library reload via **Manage Jenkins → Script Console** if updates don’t reflect.
* Check Jenkins logs for `Library not found` errors.
* Verify correct library name and branch in Jenkins configuration.

---
---

# Jenkinsfile Linting & Validation — Quick Revision

---

## ⚙️ **What**

Linting = syntax + structure validation for Jenkinsfiles before execution.
Checks correctness of pipeline blocks (stages, steps, agents, etc.).

---

## 🎯 **Why (Purpose)**

* Avoids build failures due to syntax errors.
* Saves debugging time.
* Maintains pipeline quality and consistency.
* Enforces standards across teams.

---

## 🧩 **How (Methods)**

### 1️⃣ Manual Validation

**URL:** `<JENKINS_URL>/pipeline-model-converter/validate`
Steps:

* Open Jenkins URL → `/pipeline-model-converter/validate`
* Paste Jenkinsfile → Click Validate → Review result.

✅ Works on any system with Jenkins access (not just agent).

---

### 2️⃣ CI/CD Linting Stage (Automated)

Used for: Continuous validation + Audit tracking.

```groovy
stage('Lint Jenkinsfile') {
  steps {
    script {
      sh 'curl -X POST -F "jenkinsfile=<Jenkinsfile" http://<jenkins-url>/pipeline-model-converter/validate'
    }
  }
}
```

📘 Benefits:

* Auto-checks Jenkinsfile on each commit.
* Prevents invalid code merges.
* Maintains validation logs.

---

## ⚠️ **Common Issues**

| Error             | Reason                      |
| ----------------- | --------------------------- |
| Invalid directive | Wrong stage/block placement |
| Missing braces    | Syntax formatting issue     |
| Wrong parameter   | Typo or unsupported field   |
| File not found    | Jenkinsfile missing in repo |
| HTTP 403/404      | Bad URL or permission issue |

---

## 🔧 **Troubleshooting**

* Verify file path & branch.
* Check user permissions.
* Fix closing `{}` brackets.
* Use VS Code / Jenkins plugins for quick validation.
* Always validate before merging to main.

---

## 💡 **Best Practices**

* Lint Jenkinsfiles pre-commit or via CI stage.
* Maintain shared templates for standardization.
* Add linting to PR validation for safety.
* Keep clear error outputs for faster debugging.

---
---

## ⚡ Quick Revision – Jenkins Dynamic Parameters

### 🧩 Concept

Dynamic parameters allow parameter values to change **based on other selections or runtime logic**, using **Active Choices Plugins**.

---

### ⚙️ Required Plugins

* Active Choices Plugin
* Active Choices Reactive Parameter Plugin

---

### 🏗️ Placement in Jenkinsfile

Define parameters **inside** the `pipeline` block, **before** the `stages` section.

```groovy
pipeline {
  agent any
  parameters {
    // define here
  }
  stages {
    // pipeline logic
  }
}
```

---

### 💡 Example

```groovy
pipeline {
  agent any
  parameters {
    choice(name: 'ENV', choices: ['dev', 'staging', 'prod'])

    activeChoiceReactiveParam('APP_TYPE') {
      groovyScript {
        script('return ["frontend", "backend", "database"]')
        fallbackScript('return ["none"]')
      }
    }
  }
  stages {
    stage('Build') {
      steps {
        echo "Building for ${params.ENV} with ${params.APP_TYPE}"
      }
    }
  }
}
```

---

### ⚙️ Key Points

* `choice` → static dropdown
* `activeChoiceReactiveParam` → dynamic dropdown linked to another parameter
* `groovyScript` → defines logic for dynamic values
* `fallbackScript` → backup values if main script fails

---

### 🧰 Real-World Use Case

* Choose **Environment (dev/prod)**, then load corresponding **App Types** dynamically.
* Reduces Jenkinsfile changes for environment-specific builds.

---

### ⚠️ Troubleshooting

* Ensure both Active Choice plugins are installed.
* Check **Script Approval** in Jenkins for Groovy code.
* Use fallbackScript for safe recovery if dynamic loading fails.

---

### 💡 Tip

Use dynamic parameters for environment-based logic, app versions, or artifact selection to make pipelines more flexible.

---
---


