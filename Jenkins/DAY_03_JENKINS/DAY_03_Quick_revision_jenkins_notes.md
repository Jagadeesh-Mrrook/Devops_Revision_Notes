## Quick Revision Version — Jenkins Credentials

### Concept / What
Encrypted credential store for passwords, tokens, SSH keys, secret files, AWS keys.

### Why / Purpose
No hardcoding; central control; easier rotation; environment isolation.

### How (Core Patterns)
* **Add:** Manage Jenkins → Credentials → Add (choose kind, scope, ID)
* **Bind in Pipeline:**
  * **Secret text:** `withCredentials([string(...)]) { ... }`
  * **Username + Password:** `withCredentials([usernamePassword(...)]) { ... }`
  * **SSH:** `sshagent(['id']) { git ... }`
  * **File:** `withCredentials([file(...)]) { ... }`
  * **AWS:** `withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', ...]]) { ... }`
* **Disable tracing** when using secrets: `set +x`

### Common Issues
* Wrong ID/scope → No such credential  
* Secrets appear in logs → tracing on or echoing secrets  
* SSH failures → key/known_hosts issues  
* AWS authentication/region/permissions issues

### Troubleshooting / Fixes
* Copy exact ID; use Pipeline Snippet Generator; correct scope  
* Wrap commands with `set +x`; avoid echo or inline secrets  
* SSH → use `sshagent`, ensure known_hosts populated  
* AWS → validate with `aws sts get-caller-identity`; check region & IAM policy

### Best Practices / Tips
* Follow least privilege & folder scoping  
* Use descriptive IDs; rotate credentials regularly  
* Avoid printing secrets; prefer secret files when supported by tools


---

## Quick Revision Version

### Concept / What
RBAC defines roles with permissions and assigns them to users/groups. Roles can be global or folder/project-specific. Implemented via **Role-Based Authorization Strategy plugin**.

### Why / Purpose
* Security: restrict access.
* Team management: share Jenkins safely.
* Environment separation: dev/test/prod access.
* Auditing: sensitive operations only for authorized users.

### How / Steps
1. Install Role-Based Authorization Strategy plugin.
2. Enable RBAC in Configure Global Security → Role-Based Strategy.
3. Define Global Roles (admin, developer, viewer).
4. Define Project/Folder Roles for specific jobs/folders.
5. Assign roles to users/groups.
6. Test user permissions by login.

### Common Issues
* Job/folder not visible → missing role assignment.
* Permission denied → role missing required permissions.
* Admin lockout → misconfigured global roles.
* Plugin conflicts.

### Troubleshooting
* Verify role assignments.
* Keep a super-admin account.
* Test permissions with user login.
* Check plugin compatibility.

### Best Practices
* Least privilege principle.
* Use folders and assign folder-specific roles.
* Group users via LDAP/AD.
* Keep super-admins minimal.
* Audit and clean up roles periodically.

---

## Quick Revision Version

### What
Role-Based Access Control to assign specific permissions to users/groups in Jenkins.

### Why
Prevent unauthorized access; least privilege; scoped control per folder/job.

### How
- Install Role-Based Authorization Plugin.
- Enable Role-Based Strategy in Global Security.
- Define roles (Admin, Developer, Viewer) and assign permissions.
- Assign roles to users/groups (global/folder/job scope).
- LDAP/AD integration optional.

### Common Issues
- Access denied due to missing role.
- Role scope incorrect.
- LDAP mapping conflicts.

### Fixes
- Review roles and scope.
- Check LDAP/AD group mapping.
- Test user access; audit logs.

### Best Practices
- Least privilege.
- Clear role names.
- Separate global vs folder roles.
- Centralized admin/security team.
- Periodic audits.


---

## Quick Revision Version

### What
Directly assign permissions to individual users or groups via checkboxes.

### Why
Fine-grained control; prevent unauthorized actions; ideal for custom user requirements.

### How
* Enable: Manage Jenkins → Configure Global Security → Matrix-based security.
* Add users → assign global/job permissions via checkboxes → Save.
* Can apply per folder/job.

### Common Issues
* User can’t login → username missing/mistyped.
* Permissions not applied → config not saved or conflicting authorization.
* Too many users → prone to manual errors.

### Fixes
* Verify username & LDAP integration.
* Check and save permissions correctly.
* Consider RBAC if managing many users.

### Best Practices
* Matrix = individual users; RBAC = groups/roles.
* Regular permission audit.
* Document assigned permissions.

---

## Quick Revision Version

### What
External authentication system storing Jenkins users/groups and credentials.

### Why
Centralized, secure, consistent login; integrates with RBAC/matrix security.

### How
* Install LDAP plugin.
* Configure: Manage Jenkins → Configure Global Security → LDAP.
* Provide server URL, root DN, manager DN/password.
* Map users/groups to roles or permissions.

### Common Issues
* Login fails → wrong server, DN, password.
* SSL/LDAPS errors.
* Group mapping mismatch.
* Permissions not applied correctly.

### Fixes
* Test connectivity (`ldapsearch`).
* Import valid SSL certs.
* Check user/group DN and filters.
* Adjust RBAC/matrix permissions.

### Best Practices
* Use LDAPS for security.
* Map groups to roles, not individual users.
* Audit regularly.
* Limit admin access.


---


## Quick Revision Version

### What
Jenkins runs HTTP by default; HTTPS secures transmissions via SSL/TLS.

### Why
Protect data, ensure secure communication, comply with security policies.

### How
- Security team installs SSL/TLS certificates.
- Jenkins configured to use HTTPS.
- Access via `https://<jenkins-server>`.
- DevOps monitors HTTPS enforcement.

### Common Issues
- Browser warnings (self-signed certs)
- Misconfiguration → inaccessible UI
- Expired certificates

### Fixes
- Use trusted certs
- Monitor and renew certs
- Test HTTPS setup
- Escalate config errors to security team

### Best Practices
- Always enforce HTTPS
- Security team handles certs
- DevOps aware, escalate if needed
- Avoid HTTP exposure

---

# Jenkins CSRF Protection & API Tokens — Quick Revision

## What

- **CSRF**: Attack via authenticated sessions → unauthorized actions.
- **API Tokens**: User-specific credentials for scripts/external tools.
- **Crumb**: Unique token per session; required for POST requests.

## Why

- Prevent unauthorized actions.
- Secure automated pipelines.
- Protect Jenkins resources and project data.

## How / Core Patterns

1. **Enable CSRF Protection**: `Manage Jenkins → Configure Global Security → Prevent CSRF`
2. **Create API Token**: `Manage Jenkins → User → Configure → Add new API token`
3. **Use Crumb in Pipeline**:

```groovy
withCredentials([string(credentialsId: 'jenkins-api-token', variable: 'TOKEN')]) {
  sh '''
    CRUMB=$(curl -u user:$TOKEN -s 'https://jenkins.example.com/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,":",//crumb)')
    curl -X POST -u user:$TOKEN -H "$CRUMB" https://jenkins.example.com/job/my-job/build
  '''
}
```

## Common Issues

- CSRF protection disabled.
- API token missing or incorrect.
- Crumb mismatch → POST requests rejected.

## Best Practices

- Always enable CSRF protection in production Jenkins.
- Use API tokens for automation instead of user passwords.
- Never hardcode
