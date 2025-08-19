Quick Revision Version
What

Encrypted credential store for passwords, tokens, SSH keys, secret files, AWS keys.

Why

No hardcoding; central control; easier rotation; environment isolation.

How (core patterns)

Add: Manage Jenkins → Credentials → Add (choose kind, scope, ID).

Bind in pipeline:

Secret text → withCredentials([string(...)]){ ... }

User+Pass → withCredentials([usernamePassword(...)]){ ... }

SSH → sshagent(['id']) { git ... }

File → withCredentials([file(...)]){ ... }

AWS → withCredentials([[ $class: 'AmazonWebServicesCredentialsBinding', ... ]]){ ... }

Disable tracing when using secrets: set +x.

Common Issues

Wrong ID/scope → No such credential.

Secrets in logs → tracing on or echoing secrets.

SSH failures (key/known_hosts).

AWS auth/region/permissions.

Fixes

Copy exact ID; use Snippet Generator; correct scope.

Wrap with set +x; avoid echo/inline secrets.

For SSH, use sshagent, set known_hosts.

For AWS, validate with sts get-caller-identity, region & IAM policy.

Best Practices

Least privilege & folder scoping.

Descriptive IDs; rotate regularly.

Don’t print secrets; prefer secret files when tools support them.
