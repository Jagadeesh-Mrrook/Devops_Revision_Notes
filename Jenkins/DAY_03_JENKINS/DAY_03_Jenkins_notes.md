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


# Jenkins RBAC — Role-Based Access Control

## Detailed Explanation Version

### Concept / What
Role-Based Access Control (RBAC) in Jenkins allows you to **define roles with specific permissions** and **assign them to users or groups**. This controls **who can access what** (jobs, folders, credentials, configurations). Roles can be **global** (Jenkins-wide) or **folder/project-specific**. RBAC is usually implemented using the **Role-Based Authorization Strategy plugin**.

### Why / Purpose / Use Case in Real-World
* **Security:** prevent unauthorized access to pipelines, credentials, or configurations.
* **Team management:** safely share Jenkins among large teams without giving full admin rights.
* **Environment separation:** dev/test/prod jobs can have different access.
* **Auditing & compliance:** restrict sensitive operations to authorized users only.

**Practical scenarios:**
- Developers can run builds and view logs but cannot modify jobs or credentials.
- QA team can trigger test jobs in a folder but cannot access prod deployment jobs.
- Admins can manage plugins, nodes, and credentials globally.

### How it Works / Steps / Syntax
1. **Install the plugin**
   - Manage Jenkins → Manage Plugins → Available → Role-Based Authorization Strategy → Install.

2. **Enable RBAC**
   - Manage Jenkins → Configure Global Security → Authorization → Role-Based Strategy → Save.

3. **Define Global Roles**
   - Manage Jenkins → Manage and Assign Roles → Manage Roles → Global Roles
   - Example permissions:
     - `admin` → all permissions
     - `developer` → job read/build, SCM read
     - `viewer` → read-only

4. **Define Project/Folder Roles**
   - Assign permissions specific to a folder/job.
   - Example: `qa-folder-role` → can build jobs in `QA` folder, cannot delete jobs.

5. **Assign Roles to Users/Groups**
   - Manage Jenkins → Assign Roles → Global/Project Roles → map users or LDAP groups to roles.

6. **Testing Permissions**
   - Log in as a user to confirm visible jobs, buttons, and configuration access match the role.

### Common Issues / Errors
* User cannot see a job/folder → missing project/folder role assignment.
* Permission denied on a build → missing specific build/SCM permission in role.
* Admin user locked out → misconfigured global roles (always keep at least one super-admin).
* Plugin conflicts → old authorization plugins may interfere with RBAC.

### Troubleshooting / Fixes
* Verify both Global and Project role assignments for the user.
* Keep a dedicated super-admin account.
* Test permissions by logging in as the user.
* Check plugin compatibility if RBAC is not working after updates.

### Best Practices / Tips
* Follow **least privilege principle**.
* Use folders to separate jobs and assign folder-specific roles.
* Group users (via LDAP/AD) instead of individual accounts.
* Keep super-admin accounts separate and minimal.
* Periodically audit roles and assignments; remove obsolete users or permissions.

---

# Jenkins RBAC — Role-Based Access Control

## Detailed Explanation Version

### Concept / What
RBAC (Role-Based Access Control) in Jenkins is used to assign **specific permissions** to users or groups. It ensures that users only have the privileges necessary for their roles, preventing unauthorized access to pipelines, jobs, credentials, and configuration.

### Why / Purpose / Use Case in Real-World
* Prevent unauthorized access to sensitive pipelines, credentials, or configurations.
* Assign **least privilege**: users get only the permissions they need.
* Scope permissions at global, folder, or job level.
* Centralized control via roles simplifies management in multi-project environments.

### How it Works / Steps / Syntax
1. **Install Plugin**
   - Install **Role-Based Authorization Strategy Plugin** via `Manage Jenkins → Manage Plugins`.

2. **Enable Role-Based Strategy**
   - `Manage Jenkins → Configure Global Security → Authorization → Role-Based Strategy`.

3. **Define Roles**
   - Go to `Manage Jenkins → Manage and Assign Roles → Manage Roles`.
   - Define **Global Roles** (e.g., Admin, Developer, Viewer).
   - Assign granular permissions like:
     - **Global:** Admin (all), Developer (read jobs, SCM), Viewer (read-only).
     - **Job/Folder:** create, configure, read, build, delete, cancel, etc.

4. **Assign Roles to Users/Groups**
   - `Manage Jenkins → Manage and Assign Roles → Assign Roles`.
   - Assign global or project-specific roles to users or LDAP groups.

5. **LDAP/AD Integration (optional)**
   - Users and groups can be authenticated via external LDAP/AD servers.
   - Jenkins maps LDAP groups to RBAC roles.

### Common Issues / Errors
* Users getting **“access denied”** → Incorrect role assignment or missing permissions.
* Roles not applied to specific jobs/folders → Check scope (global vs folder vs job).
* Conflicts between internal users and LDAP users → Verify mapping.

### Troubleshooting / Fixes
* Review role definitions and permissions.
* Verify scope (global vs folder vs job).
* Check LDAP/AD mapping for external users.
* Use **Audit Logs** or **Test User** to validate access.

### Best Practices / Tips
* Apply **least privilege** principle.
* Use descriptive role names and permissions.
* Separate global roles vs project/folder roles.
* Use centralized teams (admins/security) for RBAC management.
* Regularly audit role assignments and access.

---

# Jenkins Matrix-Based Security

## Detailed Explanation Version

### Concept / What
Matrix-Based Security in Jenkins is a **fine-grained permission system** that allows assigning **specific permissions directly to individual users or groups**. Unlike RBAC, where roles are created first and then assigned, matrix-based security assigns permissions per user.

### Why / Purpose / Use Case in Real-World
* Useful when different users require **custom, individual permissions**.
* Prevents unauthorized access while giving users only the permissions they need.
* Easier to control specific actions like **read, build, configure, delete** per job or global resource.
* Ideal for small teams or users with highly individualized access needs.

### How it Works / Steps / Syntax
1. **Enable Matrix-Based Security**
   * Go to **Manage Jenkins → Configure Global Security → Authorization → Matrix-based security**.
2. **Add Users**
   * Enter the **username** of the user you want to assign permissions.
   * Press **Add**.
3. **Assign Permissions**
   * Check the boxes for the permissions that user requires.
   * Categories:
     * **Global permissions**: Administer, Read, Run scripts, Manage credentials, etc.
     * **Job permissions**: Create, Configure, Read, Build, Delete, Cancel, etc.
4. **Save Settings**
   * Apply changes to enforce the matrix for selected users.
5. **Per-Job/Folders**
   * Permissions can also be configured per folder or job if folder-based security is enabled.

### Common Issues / Errors
* User cannot access Jenkins → Username not added or mis-typed.
* Permissions not taking effect → Changes not saved, or conflicts with another authorization strategy.
* Too many manual checkboxes → Can lead to mistakes if managing many users.

### Troubleshooting / Fixes
* Verify the user exists in Jenkins or LDAP/AD if integrated.
* Double-check all required permissions are selected.
* Save configuration properly.
* For large teams, consider RBAC for easier management to avoid errors.

### Best Practices / Tips
* Use RBAC for multiple users with the same permissions; Matrix for individual users.
* Review matrix periodically to remove unused or unnecessary permissions.
* Use descriptive usernames and consistent permission patterns.
* Document the permissions assigned per user for audit and clarity.

---

# Jenkins LDAP — External Authentication

## Detailed Explanation Version

### Concept / What
LDAP (Lightweight Directory Access Protocol) is an external authentication system used by Jenkins to centrally manage users and groups. It stores usernames, passwords, and group information. Jenkins queries the LDAP server for authentication instead of using its internal database.

### Why / Purpose / Use Case in Real-World
* Centralized user and group management.
* Consistent authentication across multiple tools and Jenkins instances.
* High security by avoiding storing credentials in Jenkins.
* Simplifies access control when combined with RBAC or matrix-based security.
* Common in enterprises using Active Directory (AD) or other LDAP-compatible servers.

### How it Works / Steps / Syntax
1. **Install LDAP plugin**
   * Manage Jenkins → Manage Plugins → Available → Search “LDAP Plugin” → Install.
2. **Configure LDAP in Jenkins**
   * Manage Jenkins → Configure Global Security → Security Realm → LDAP
   * Provide:
     * LDAP server URL (`ldap://host` or `ldaps://host` for secure)
     * Root DN / domain names
     * Manager DN and password (for querying LDAP)
3. **Map permissions**
   * Combine with RBAC or matrix-based security:
     * Users/groups in LDAP → mapped to Jenkins roles or individual permissions.
4. **Test login**
   * Try a Jenkins login using LDAP user credentials to verify authentication.

### Common Issues / Errors
* **Login failures** → Wrong URL, port, or credentials for LDAP server.
* **SSL errors (LDAPS)** → Certificates not trusted or missing.
* **Group mapping errors** → Jenkins cannot find LDAP groups or roles.
* **Authorization mismatch** → LDAP user exists but RBAC/matrix permissions not properly mapped.

### Troubleshooting / Fixes
* Validate LDAP URL and connectivity (`ldapsearch` can help test).
* Check LDAPS certificate validity; import trusted CA if necessary.
* Verify user DN, group DN, and filters in Jenkins configuration.
* Confirm permissions in Jenkins RBAC or matrix settings.

### Best Practices / Tips
* Prefer `ldaps://` for secure communication.
* Centralize user management for multiple Jenkins instances.
* Combine with RBAC/matrix-based security for fine-grained access control.
* Limit admin privileges; assign roles based on group membership.
* Regularly audit LDAP mappings and access logs.

---

# Jenkins HTTPS & Security

## Detailed Explanation Version

### Concept / What
Jenkins by default runs on HTTP. To secure user access and protect sensitive data during transmission, HTTPS is implemented using SSL/TLS certificates.

### Why / Purpose / Use Case in Real-World
- Protects sensitive project data from being intercepted.
- Ensures secure communication between Jenkins server and users.
- Complies with organizational security policies and standards.
- Especially important in production environments with multiple users.

### How it Works / Steps / Syntax
1. **SSL/TLS Certificates:** Security team installs certificates on the Jenkins server.
2. **Configure Jenkins:** Update Jenkins settings to use HTTPS instead of HTTP.
3. **Access Jenkins via HTTPS:** Users connect securely using `https://<jenkins-server>`.
4. **Role Awareness:** DevOps engineers monitor and inform the security team if HTTPS is not enforced, ensuring secure CI/CD pipelines.

### Common Issues / Errors
- Browser warnings if self-signed certificates are used.
- HTTPS misconfiguration leading to inaccessible Jenkins UI.
- Certificate expiration causing failed secure connections.

### Troubleshooting / Fixes
- Use certificates from trusted authorities to avoid browser warnings.
- Regularly monitor and renew SSL/TLS certificates.
- Verify Jenkins HTTPS configuration using test connections and `curl`.
- Coordinate with the security team for any configuration errors.

### Best Practices / Tips
- Always enforce HTTPS in production Jenkins.
- Security team should handle certificate issuance and installation.
- DevOps engineers should be aware of HTTPS usage and escalate issues if not enforced.
- Avoid exposing Jenkins over plain HTTP.

---

# Jenkins CSRF Protection & API Tokens

## Detailed Explanation Version

### Concept / What
CSRF (Cross-Site Request Forgery) is an attack where unauthorized commands are transmitted from a user that the web application trusts. Jenkins implements CSRF protection to prevent malicious users from exploiting authenticated sessions. API tokens are unique credentials used to authenticate scripts or external systems without exposing the main user password.

### Why / Purpose / Use Case in Real-World
- Prevent unauthorized actions using valid user sessions.
- Ensure automated pipelines or scripts authenticate securely.
- Protect sensitive Jenkins jobs and resources from malicious triggers.

### How it Works / Steps / Syntax

1. **CSRF Protection**:
   - Enabled in Jenkins under **Manage Jenkins → Configure Global Security → Prevent Cross Site Request Forgery**.
   - Jenkins issues a **Crumb**, which is a unique token for each session. This crumb must be included in any POST request to validate that the request comes from a legitimate source.

2. **API Tokens**:
   - Generated per user: **People → Username → Configure → Show API Token**.
   - Can be used in Jenkinsfile or external scripts to authenticate without exposing the main password.

3. **Using Crumbs in Scripts / Pipelines**:
   - Fetch crumb via Jenkins API:
     ```bash
     CRUMB=$(curl -u user:API_TOKEN -s 'http://jenkins-server/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,":",//crumb)')
     ```
   - Include crumb in POST requests:
     ```bash
     curl -u user:API_TOKEN -H "$CRUMB" -X POST http://jenkins-server/job/myjob/build
     ```
   - In Jenkins Pipeline (Groovy):
     ```groovy
     withCredentials([string(credentialsId: 'jenkins-token', variable: 'API_TOKEN')]) {
       sh '''
         CRUMB=$(curl -u user:$API_TOKEN -s 'http://jenkins-server/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,":",//crumb)')
         curl -u user:$API_TOKEN -H "$CRUMB" -X POST http://jenkins-server/job/myjob/build
       '''
     }
     ```

### Common Issues / Errors
- API requests fail without the correct crumb.
- Tokens leaked if included in public scripts or logs.
- CSRF protection disabled → security vulnerability.

### Troubleshooting / Fixes
- Always include crumb in API POST requests.
- Keep API tokens secret; rotate if compromised.
- Verify CSRF protection is enabled in global security settings.

### Best Practices / Tips
- DevOps engineers should ensure pipelines use API tokens safely.
- Avoid disabling CSRF protection for convenience.
- Use role-based access and credentials together for secure pipelines.

## Quick Revision Version

- CSRF: Attack via authenticated sessions → enable protection in Jenkins.
- **Crumb**: Unique token per session → include in POST requests.
- API tokens: per-user credentials for scripts.
- Include tokens and crumbs in pipelines; keep them secret.
- DevOps monitors and ensures secure usage; security team implements protection.

