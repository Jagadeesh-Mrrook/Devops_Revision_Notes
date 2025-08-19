# Jenkins Credentials — Storing & Using Secrets

## Detailed Explanation Version

### Concept / What

Jenkins provides a **Credential Store** to securely save sensitive data (usernames/passwords, secret text/tokens, SSH private keys, secret files, AWS keys). Credentials are **encrypted at rest**, masked in logs when used correctly, and can be scoped to **Global** or **Folder/Project**.

### Why / Purpose / Use Case in Real-World

* **Avoid hardcoding** secrets in Jenkinsfiles or scripts.
* **Centralize & audit** who can use which secret.
* **Environment isolation** (e.g., dev/test/prod credentials kept separate via folders/labels).
* **Safer rotation** without code changes.

### How it Works / Steps / Syntax

1. **Add a credential**

   * Path: **Manage Jenkins → Credentials → (System) → Global credentials (or Folder) → Add Credentials**
   * Choose **Kind** and give a clear **ID** (e.g., `aws-prod-deploy`, `github-ci-ssh`).
   * Recommended kinds:

     * **Username with password**
     * **Secret text** (API tokens, webhook tokens)
     * **SSH Username with private key**
     * **Secret file** (kubeconfig, certs, JSON keys)
     * **AWS credentials** (Access Key ID + Secret)

2. **Bind credentials in Pipeline** (available only inside the block)

   **Secret text**

   ```groovy
   withCredentials([string(credentialsId: 'slack-token', variable: 'SLACK_TOKEN')]) {
     sh '''
       set +x
       curl -H "Authorization: Bearer $SLACK_TOKEN" https://slack.example/api
     '''
   }
   ```

   **Username + Password**

   ```groovy
   withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
     sh '''
       set +x
       mvn deploy -Dnexus.user=$USER -Dnexus.pass=$PASS
     '''
   }
   ```

   **SSH key (for Git/SSH)**

   ```groovy
   sshagent(credentials: ['github-ssh']) {
     sh 'git fetch --all --prune'
   }
   ```

   **Secret file**

   ```groovy
   withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
     sh 'kubectl get ns --kubeconfig "$KUBECONFIG"'
   }
   ```

   **AWS credentials**

   ```groovy
   withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-prod']]) {
     sh 'aws sts get-caller-identity'
   }
   ```

   **Declarative shorthand for username/password**

   ```groovy
   pipeline {
     agent any
     environment { NEXUS = credentials('nexus') } // exposes NEXUS_USR and NEXUS_PSW
     stages {
       stage('Deploy') {
         steps {
           sh 'set +x; mvn deploy -Dnexus.user=$NEXUS_USR -Dnexus.pass=$NEXUS_PSW'
         }
       }
     }
   }
   ```

3. **About shell tracing**

   * `set -x` **enables** command tracing (prints commands & args).
   * `set +x` **disables** tracing. Always disable around secret usage to avoid leaks.

### Common Issues / Errors

* **`No such credential 'ID'`** → Wrong `credentialsId` or wrong **scope** (credential saved under a folder, job is elsewhere).
* **Secrets appear in logs** → Tracing enabled (`set -x`) or commands echo the secret.
* **SSH: Permission denied / host key verification failed** → Missing/incorrect key or `known_hosts` not set.
* **AWS auth failures** → Expired/rotated keys, wrong account/region, missing IAM permissions.
* **Variable not available outside block** → Using bound variables after `withCredentials {}` ends.
* **Masking didn’t work** → Secret embedded in a larger string or produced by a sub-process in a way Jenkins can’t mask.

### Troubleshooting / Fixes

* **Verify ID & scope**: open **Manage Jenkins → Credentials** and copy the exact ID; prefer the Pipeline Snippet Generator to avoid typos.
* **Prevent log leaks**: wrap secret usage with `set +x`; never `echo` or build long command strings that inline secrets.
* **SSH**: use `sshagent(['id'])`; ensure private key matches server; pre-populate `known_hosts` (e.g., `ssh-keyscan -H host >> ~/.ssh/known_hosts`).
* **AWS**: test with `aws sts get-caller-identity`; check region (`AWS_REGION`); confirm policy permissions; re-create keys if compromised/expired.
* **Scope**: move the job into the folder that holds the credential or re-create the credential at the required scope.
* **Short/odd secrets**: if masking fails for very short values, rotate to a longer token where possible.

### Best Practices / Tips

* **Never hardcode** secrets; always use credential bindings.
* **Least privilege**: one credential per purpose/environment; restrict who can **view** vs **use** creds via RBAC.
* **Descriptive IDs** and descriptions; avoid generic names like `token1`.
* **Rotate** keys regularly; automate rotation where possible.
* **Keep Master executors = 0** and run builds on agents; use folder-scoped creds per team/environment.
* **Audit & cleanup**: remove unused credentials; review usage periodically.
* **Avoid printing** commands containing secrets; prefer config files (secret file kind) when tools support it.

---

