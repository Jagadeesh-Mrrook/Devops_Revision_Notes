# Kubernetes Authentication & Authorization (Detailed Explanation)

## **Concept / What**

### **Authentication**

Authentication verifies **who** is trying to access the Kubernetes API (user or service).
Kubernetes does not maintain its own user database; instead, it uses external identity sources such as ServiceAccounts (for pods), IAM identities (cloud providers), or OIDC providers.

### **Authorization**

Authorization controls **what actions** the authenticated identity can perform inside the cluster (e.g., list pods, create deployments, read secrets). Kubernetes uses **RBAC** (Role-Based Access Control) for authorization.

---

## **Why / Purpose / Real-World Use Cases**

### **Why Authentication?**

* To ensure only trusted users and workloads communicate with the API server.
* To prevent unauthorized access.
* To identify actions in audit logs.

### **Why Authorization?**

* To strictly control what authenticated identities can do.
* To enforce **least privilege** across environments.
* To avoid accidental or malicious changes inside the cluster.

### **Real-World Scenarios**

* Developers can access only the **dev namespace**, not prod.
* An application pod can read **ConfigMaps** but must not read **Secrets**.
* CI/CD pipelines use ServiceAccounts to deploy workloads with limited permissions.

---

## **How it Works / Steps / Syntax**

### **Authentication Flow**

1. User/Pod sends request to Kubernetes API.
2. API server verifies identity using one of the supported authentication methods:

   * **ServiceAccount tokens** (for pods)
   * **IAM-based authentication** (AWS, Azure, GCP)
   * **OIDC tokens** (Okta, Google, Cognito, AAD)
3. If identity is valid → proceed to authorization.
4. If invalid → request fails with **401 Unauthorized**.

### **Authorization (RBAC) Flow**

1. Once authenticated, API server evaluates:

   * Roles / ClusterRoles
   * RoleBindings / ClusterRoleBindings
2. Checks permissions for the requested action:

   * **Verb** (get, list, create, delete)
   * **Resource** (pods, deployments, secrets)
   * **Namespace** (if applicable)
3. If allowed → action proceeds.
4. If not → **403 Forbidden**.

### **Testing Authorization**

```
kubectl auth can-i create pods --namespace dev
kubectl auth can-i get secrets --as system:serviceaccount:dev:app-sa
```

Output:

* "yes" → allowed
* "no" → forbidden

---

## **Common Issues / Errors**

| Issue                                  | Description                                            |
| -------------------------------------- | ------------------------------------------------------ |
| 401 Unauthorized                       | Invalid token, expired token, wrong kubeconfig context |
| 403 Forbidden                          | RBAC permissions missing or incorrect                  |
| User cannot access cluster             | IAM not mapped in aws-auth (AWS)                       |
| Pod has wrong permissions              | Incorrect ServiceAccount / RoleBinding                 |
| Pod has more permissions than expected | ClusterRoleBinding used instead of RoleBinding         |

---

## **Troubleshooting / Fixes**

### **Authentication Issues (401)**

* Verify kubeconfig context.
* Check token validity.
* Regenerate or reconfigure credentials.

### **Authorization Issues (403)**

* Validate Role/ClusterRole rules.
* Validate RoleBinding/ClusterRoleBinding.
* Ensure correct namespace usage.
* Test with `kubectl auth can-i`.

### **Cloud/IAM Issues**

* Validate IAM-to-Kubernetes mapping (AWS `aws-auth` ConfigMap).
* Ensure OIDC provider is configured.
* Verify ServiceAccount annotation (for IRSA).

---

## **Best Practices / Tips**

* Always follow **least privilege**.
* Use **namespaced Roles** unless you truly need cluster-wide access.
* Avoid ClusterRoleBindings for application workloads.
* Use ServiceAccounts for all in-cluster workloads.
* Use IAM/OIDC for human user authentication in cloud environments.
* Test all permissions using `kubectl auth can-i`.
* Rotate tokens and credentials regularly.
* Avoid using `system:masters` group in production.

---
---

# RBAC: Roles, ClusterRoles, RoleBindings, ClusterRoleBindings (Detailed Explanation)

## **Concept / What**

### **Role**

A **Role** is a namespaced RBAC object that defines permissions (verbs) on resources within a specific namespace. It controls what an identity can do inside that namespace, such as reading pods or creating configmaps.

### **ClusterRole**

A **ClusterRole** defines cluster-wide permissions. It is used for non-namespaced resources (nodes, PVs, namespaces) or when the same permissions are needed across multiple namespaces.

### **RoleBinding**

A **RoleBinding** attaches a **Role** to a user, group, or ServiceAccount within a single namespace.

### **ClusterRoleBinding**

A **ClusterRoleBinding** connects a **ClusterRole** to users, groups, or ServiceAccounts across the entire cluster.

### **Users & Groups in RBAC**

* Kubernetes **does not create users**.
* Users and groups come from **external identity providers** such as IAM (AWS), Active Directory, OIDC (Okta, Google, Azure AD).
* Kubernetes only receives usernames/group names and matches them in RBAC.
* ServiceAccounts are the **only Kubernetes-native identity**.

---

## **Why / Purpose / Real-World Use Cases**

### **Why Roles & ClusterRoles?**

* To define fine-grained permissions.
* To support both namespaced and cluster-wide operations.
* To enforce least privilege for developers, admins, and applications.

### **Why RoleBindings & ClusterRoleBindings?**

* To attach permissions to authenticated identities.
* To safely segregate access:

  * Devs access only `dev` namespace.
  * SRE/DevOps get cluster-wide visibility.
  * Applications get only the permissions they require.

### **Real-World Scenarios**

* A pod should read only ConfigMaps (not Secrets).
* Developers must not modify resources in production.
* Monitoring tools like Prometheus require cluster-wide read access.
* CI/CD pipelines use ServiceAccounts for automation in specific namespaces.

---

## **How It Works / Steps / Syntax**

### **1. Role (Namespaced)**

```yaml\apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
```

**Explanation:**

* Defines permissions for specific resources.
* Works only in the `dev` namespace.

---

### **2. RoleBinding (Namespaced)**

```yaml\apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-reader-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: dev
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

**Explanation:**

* Binds the Role to a ServiceAccount.
* Grants the app only ConfigMap read permissions in `dev`.

---

### **3. ClusterRole (Cluster-Wide)**

```yaml\apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list"]
```

**Explanation:**

* Used for non-namespaced resources.
* Often used by SREs or monitoring tools.

---

### **4. ClusterRoleBinding (Cluster-Wide)**

```yaml\apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
  - kind: User
    name: devops-engineer
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

**Explanation:**

* Grants cluster-wide node read permissions to a human user.

---

## **Common Issues / Errors**

| Issue                                         | Description                                    |
| --------------------------------------------- | ---------------------------------------------- |
| 403 Forbidden                                 | Incorrect Role/Binding or namespace mismatch   |
| User can't see cluster-wide resources         | Using Role instead of ClusterRole              |
| Pod has more permissions than expected        | ClusterRoleBinding used instead of RoleBinding |
| Access works in one namespace but not another | RoleBinding does not apply cluster-wide        |
| User not recognized                           | IAM/OIDC mapping missing                       |

---

## **Troubleshooting / Fixes**

### **RBAC Debugging**

* Test permissions:

```
kubectl auth can-i list pods --as alice -n dev
```

* Check Role or ClusterRole:

```
kubectl describe role <name> -n <ns>
kubectl describe clusterrole <name>
```

* Check RoleBinding/ClusterRoleBinding:

```
kubectl get rolebinding -n <ns>
kubectl get clusterrolebinding
```

### **User/Group Issues**

* Verify IAM/OIDC mapping (e.g., AWS `aws-auth` ConfigMap).
* Ensure correct username/group name.

---

## **Best Practices / Tips**

* Always follow **least privilege**.
* Use **Role + RoleBinding** for workload permissions.
* Use **ClusterRole + ClusterRoleBinding** only when required.
* Avoid ClusterRoleBinding for application service accounts.
* Use ServiceAccounts for all automation and pod identities.
* Separate RBAC per environment (dev, staging, prod).
* Always test using `kubectl auth can-i`.
* Document RBAC mappings clearly.

---
---

# ServiceAccounts (Detailed Explanation)

## **Concept / What**

A **ServiceAccount (SA)** is a Kubernetes-native identity used by **Pods** and internal automation to authenticate to the Kubernetes API server. Unlike human users (IAM/OIDC), ServiceAccounts are created and managed **inside Kubernetes**.

Each ServiceAccount provides:

* A unique identity for pods.
* A token (JWT or projected token).
* Optional annotations (e.g., AWS IRSA mapping).
* RBAC bindings using Roles or ClusterRoles.

ServiceAccounts are the **only built‑in authentication mechanism for workloads** running inside the cluster.

---

## **Why / Purpose / Real-World Use Cases**

### **Why ServiceAccounts exist**

* Pods need secure, isolated identities.
* Operators/Controllers must modify Kubernetes objects with limited permissions.
* CI/CD workloads need access to specific namespaces.
* Cloud provider access (e.g., AWS IRSA) must be pod‑level and secure.

### **Real‑World Examples**

* An application pod should only read ConfigMaps, not Secrets.
* A Jenkins agent running inside K8s should only create pods in `dev`.
* Prometheus requires read-only access across namespaces.
* A pod in EKS must access S3 using IRSA instead of storing AWS keys.

---

## **How It Works / Steps / Syntax**

### **1. Create a ServiceAccount**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: dev
```

**Explanation:**

* Defines a ServiceAccount scoped to a namespace.

---

### **2. Use ServiceAccount in Deployment/Pod**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      serviceAccountName: app-sa
      containers:
        - name: demo
          image: nginx
```

**Explanation:**

* `serviceAccountName` attaches the SA to pod.
* Pod automatically receives the SA token at:
  `/var/run/secrets/kubernetes.io/serviceaccount/token`

---

### **3. Create Role for ServiceAccount**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
```

**Explanation:**

* Grants read-only access to ConfigMaps.

---

### **4. Bind ServiceAccount using RoleBinding**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-reader-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: dev
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

**Explanation:**

* Connects ServiceAccount → permissions.
* Least privilege enforced.

---

## **Including AWS Authentication (IRSA + aws-auth relevance)**

ServiceAccounts participate in AWS authentication only in the context of **IRSA**.

### **How ServiceAccounts relate to aws-auth**

* `aws-auth` ConfigMap is used for **human access** (IAM → Kubernetes RBAC).
* ServiceAccounts are used for **pod access**.
* For AWS permissions (like S3 access), SA is mapped to an IAM role using IRSA.

### **IRSA Example (SA with IAM Role annotation)**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-app-sa
  namespace: dev
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/S3AccessRole
```

**Explanation:**

* SA is linked to an IAM Role.
* Pod receives AWS credentials automatically via OIDC.
* No need to use `aws-auth` for ServiceAccounts.

---

## **Common Issues / Errors**

| Issue                        | Reason                                         |
| ---------------------------- | ---------------------------------------------- |
| Pod gets Forbidden (RBAC)    | SA lacks Role/RoleBinding                      |
| Pod uses default SA          | serviceAccountName missing                     |
| Token not mounted            | automount disabled                             |
| IRSA not working             | Wrong annotation or IAM trust relationship     |
| Pod has too many permissions | ClusterRoleBinding used instead of RoleBinding |

---

## **Troubleshooting / Fixes**

### Check pod's ServiceAccount:

```
kubectl get pod <pod> -o jsonpath='{.spec.serviceAccountName}'
```

### Check SA details:

```
kubectl describe sa app-sa -n dev
```

### Test SA permissions:

```
kubectl auth can-i list configmaps --as system:serviceaccount:dev:app-sa -n dev
```

### Verify IRSA role assumption:

```
aws sts get-caller-identity
```

(Execute inside pod if IRSA is configured)

---

## **Best Practices / Tips**

* Never use the **default** ServiceAccount for workloads.
* Create separate SA per application or per workload type.
* Follow **strict least privilege**.
* Use Role + RoleBinding for namespace-scoped permissions.
* Avoid ClusterRoleBinding unless necessary.
* Use IRSA instead of AWS credentials in pods.
* Keep all SA + RBAC configs in Git.
* Validate permissions using `kubectl auth can-i`.

---
---

# IAM Role for ServiceAccount (IRSA) – Detailed Explanation

## **Concept / What**

**IRSA (IAM Role for ServiceAccount)** is an EKS feature that lets a Kubernetes **ServiceAccount** assume an **AWS IAM Role** securely.
This allows pods to access AWS services (S3, DynamoDB, SNS, SQS, Secrets Manager, etc.) **without storing AWS access keys** inside containers.

IRSA uses:

* Kubernetes **ServiceAccount identity**
* EKS **OIDC provider**
* IAM **trust policy** referencing the ServiceAccount
* STS **AssumeRoleWithWebIdentity**

---

## **Why / Purpose / Real-World Use Cases**

### **Why IRSA Exists**

* Avoid hard‑coded AWS credentials inside pods
* Enforce pod‑level AWS least privilege
* Provide temporary, auto‑rotating AWS credentials
* Remove dependency on EC2 instance IAM roles

### **Real-World Scenarios**

* Application pod accessing S3
* External DNS controller accessing Route53
* ALB Controller using ELB IAM permissions
* CI/CD pipelines pulling artifacts from S3 or Secrets Manager
* Prometheus reading metrics from AWS APIs

---

## **How It Works / Steps / Syntax**

IRSA setup has **4 main steps**:

---

### **1. Enable / Verify OIDC Provider in IAM**

Every EKS cluster exposes its own OIDC endpoint.
Get the OIDC URL:

```
aws eks describe-cluster --name <cluster-name> --query "cluster.identity.oidc.issuer" --output text
```

Check if the IAM OIDC provider exists:

```
aws iam list-open-id-connect-providers
```

If missing, create it manually (most setups already have it):

```
aws iam create-open-id-connect-provider \
  --url <OIDC-URL> \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list <thumbprint>
```

**Purpose:**
AWS must trust EKS's OIDC provider to verify ServiceAccount tokens.

---

### **2. Create IAM Role with Trust Policy Referencing the ServiceAccount**

Example trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.region.amazonaws.com/id/EXAMPLEOIDC"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.region.amazonaws.com/id/EXAMPLEOIDC:sub": "system:serviceaccount:dev:s3-app-sa"
        }
      }
    }
  ]
}
```

**Explanation:**

* **Federated** → EKS OIDC provider
* **AssumeRoleWithWebIdentity** → Required for IRSA
* **Condition.sub** → Exact ServiceAccount + namespace allowed to assume this IAM role

### **3. Attach IAM Policy (AWS Permissions)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-app-bucket",
        "arn:aws:s3:::my-app-bucket/*"
      ]
    }
  ]
}
```

---

### **4. Create ServiceAccount with IAM Role Annotation**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-app-sa
  namespace: dev
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/S3AccessRole
```

**Explanation:**

* Annotation tells EKS + AWS that this SA is linked to this IAM role.
* EKS injects a token for this SA that AWS STS can validate.

---

### **5. Use ServiceAccount in Pod/Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3-app
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: s3demo
  template:
    metadata:
      labels:
        app: s3demo
    spec:
      serviceAccountName: s3-app-sa
      containers:
        - name: app
          image: amazon/aws-cli
          command: ["sh", "-c"]
          args:
            - aws s3 ls s3://my-app-bucket
```

---

## **Common Issues / Errors**

| Issue                                   | Reason                                      |
| --------------------------------------- | ------------------------------------------- |
| `AccessDenied`                          | Wrong IAM policy or wrong S3 ARN            |
| Pod using instance role instead of IRSA | Metadata not restricted or IRSA not enabled |
| Pod sees "Unable to locate credentials" | SA annotation missing or wrong              |
| IRSA not working                        | Wrong trust policy / wrong OIDC URL         |
| Pod using wrong SA                      | `serviceAccountName` missing                |
| Wrong namespace in trust condition      | SA cannot assume IAM role                   |

---

## **Troubleshooting / Fixes**

### Check ServiceAccount used by pod:

```
kubectl get pod <pod> -o jsonpath='{.spec.serviceAccountName}'
```

### Check annotation:

```
kubectl describe sa s3-app-sa -n dev
```

### Check AWS identity inside pod:

```
kubectl exec -it <pod> -- aws sts get-caller-identity
```

### Verify OIDC provider:

```
aws iam list-open-id-connect-providers
```

### Validate trust policy matches:

* OIDC provider URL
* Namespace
* ServiceAccount name

---

## **Best Practices / Tips**

* Always use IRSA instead of EC2 instance roles for pods.
* Follow **least privilege** when creating IAM policies.
* One IAM role per application/group.
* Never give wildcard (`*`) AWS permissions.
* Validate IRSA using `aws sts get-caller-identity`.
* Keep trust policy restricted to specific ServiceAccounts.
* Store IRSA config (IAM + YAML) in Git.

---
---

# Least Privilege Model – Detailed Explanation

## **Concept / What**

The **Least Privilege Model** means granting only the **minimum required permissions** for a user, pod, application, or process to perform its job—nothing more.

In Kubernetes and AWS (via IRSA), least privilege is applied to:

* ServiceAccounts (pod identity)
* Roles and RoleBindings
* ClusterRoles and ClusterRoleBindings
* IAM roles and IAM policies
* CI/CD automation
* Application access to Kubernetes and AWS

This is the core principle behind RBAC and IRSA.

---

## **Why / Purpose / Real-World Use Cases**

### **1. Limit the blast radius**

If a pod or user is compromised, restricted permissions prevent large-scale damage.

### **2. Prevent accidental changes**

Devs or apps should not accidentally update/delete deployments, secrets, or cluster resources.

### **3. Comply with security standards**

Most organizations follow:

* CIS Benchmarks
* SOC2
* PCI-DSS
* ISO27001
  All require least privilege enforcement.

### **4. Reduce impact of human error**

Most production outages occur because someone has permissions they should not have.

---

## **How It Works / Steps / Syntax**

### **Kubernetes Side (RBAC)**

1. Prefer **Role + RoleBinding** for namespace-specific access.
2. Avoid using **ClusterRoleBinding** for applications.
3. Grant only required verbs:

   * Use mostly: `get`, `list`, `watch`
   * Avoid: `create`, `update`, `patch`, `delete` unless required
4. Create separate ServiceAccounts per application.
5. Never use the **default** ServiceAccount.

---

### **AWS Side (IRSA)**

1. Create **one IAM Role per microservice** or per workload type.
2. Restrict IAM policies:

   * Do not use `"Action": "*"`
   * Do not use `"Resource": "*"`
3. Specify exact AWS resources (e.g., specific S3 bucket ARN).
4. Restrict trust policy to a **specific ServiceAccount + namespace**.
5. Avoid giving pods EC2 instance role permissions.

---

## **Real-World Scenarios**

### **Scenario 1: Application pod in dev namespace**

Allowed:

* Read ConfigMaps
* List pods

Not allowed:

* Read secrets
* Delete deployments
* Modify services
* Access other namespaces

### **Scenario 2: Pod accessing S3 via IRSA**

IAM policy should only include:

* `s3:GetObject`
* `s3:ListBucket`
* For **specific** bucket ARN

Should NOT include:

* `s3:*`
* Wildcard bucket access

### **Scenario 3: Jenkins in Kubernetes**

Allowed:

* Create pods in dev namespace for agents

Not allowed:

* Delete deployments
* Read secrets across namespaces
* Access entire cluster

---

## **Common Issues / Errors**

| Issue                               | Reason                                         |
| ----------------------------------- | ---------------------------------------------- |
| Pod reads secrets it shouldn't      | ClusterRoleBinding used instead of RoleBinding |
| App deletes deployments             | Granted delete permissions unintentionally     |
| Developer can access prod namespace | Assigned cluster-wide permissions              |
| Pod has full S3 access              | IAM policy used wildcards (`*`)                |

---

## **Troubleshooting / Fixes**

### Check permissions for a ServiceAccount:

```
kubectl auth can-i --list --as system:serviceaccount:<ns>:<sa>
```

### Check RoleBindings:

```
kubectl get rolebinding -n <namespace>
```

### Check ClusterRoleBindings:

```
kubectl get clusterrolebinding
```

### Inspect IAM policy:

```
aws iam get-role-policy --role-name <RoleName> --policy-name <PolicyName>
```

### Verify AWS identity inside pod:

```
kubectl exec -it <pod> -- aws sts get-caller-identity
```

---

## **Best Practices / Tips**

* Use **Role + RoleBinding** for app-specific permissions.
* Avoid **ClusterRoleBinding** for application ServiceAccounts.
* Give pods only read permissions unless write permissions are absolutely needed.
* Avoid wildcard IAM actions/resources.
* Maintain separate ServiceAccounts and IAM roles per app.
* Keep RBAC and IRSA policies version-controlled.
* Validate permissions using `kubectl auth can-i`.
* Review IAM policies periodically to tighten access.
* Never give `system:masters` access to anyone except cluster admins.

---
---

# Access Testing (`kubectl auth can-i`) – Detailed Explanation

## **Concept / What**

`kubectl auth can-i` is a Kubernetes command used to check whether a **user**, **group**, or **ServiceAccount** has permission to perform specific actions on specific resources.

It returns either:

* **"yes"** → permission allowed
* **"no"** → permission denied

It is the primary tool for validating and troubleshooting RBAC and IAM integrations (via `aws-auth` or IRSA).

---

## **Why / Purpose / Real-World Use Cases**

### **1. Validate RBAC changes**

Before deploying workloads, you can test whether a Role/RoleBinding is correctly configured.

### **2. Debug 403 Forbidden errors**

When pods or users get:

```
Error: forbidden: User "xyz" cannot list pods
```

`can-i` confirms the missing permission.

### **3. Security checks**

Ensure ServiceAccounts and users do not have unnecessary access.

### **4. Verify IAM Role mappings (EKS)**

Test if IAM roles mapped via `aws-auth` or IRSA are correctly recognized by Kubernetes.

---

## **How It Works / Steps / Syntax**

### **Basic Syntax**

```
kubectl auth can-i <verb> <resource>
```

Example:

```
kubectl auth can-i create pods
```

---

### **Check Permissions in a Namespace**

```
kubectl auth can-i list pods -n dev
```

---

### **Check Permissions for a User**

```
kubectl auth can-i delete deployments --as dev-user
```

Useful for IAM + `aws-auth` debugging.

---

### **Check Permissions for a ServiceAccount**

```
kubectl auth can-i get configmaps \
  --as system:serviceaccount:dev:app-sa \
  -n dev
```

---

### **List All Permissions for a User**

```
kubectl auth can-i --list --as dev-user
```

### **List All Permissions for a ServiceAccount**

```
kubectl auth can-i --list \
  --as system:serviceaccount:dev:app-sa \
  -n dev
```

---

## **Real-World Scenarios**

### **Scenario 1: App forbidden to read ConfigMaps**

Test:

```
kubectl auth can-i get configmaps \
  --as system:serviceaccount:dev:app-sa \
  -n dev
```

If "no" → fix Role/RoleBinding.

---

### **Scenario 2: CI/CD pod cannot create pods**

Test:

```
kubectl auth can-i create pods \
  --as system:serviceaccount:dev:jenkins-sa \
  -n dev
```

---

### **Scenario 3: IAM Role not mapped correctly in EKS**

Test:

```
kubectl auth can-i get pods --as dev-user
```

---

### **Scenario 4: Validate Least Privilege**

List all permissions:

```
kubectl auth can-i --list --as system:serviceaccount:prod:payment-sa -n prod
```

---

## **Common Issues / Errors**

| Issue                                 | Reason                                                     |
| ------------------------------------- | ---------------------------------------------------------- |
| Wrong namespace                       | Forgot `-n <namespace>` and used default namespace         |
| IAM role not recognized               | Incorrect mapping in `aws-auth`                            |
| Wrong ServiceAccount                  | SA name or namespace mismatch                              |
| Using ClusterRoleBinding accidentally | Resulting in extra permissions                             |
| Missing specific verbs                | Only `get/list` allowed, but `update/patch/delete` missing |

---

## **Troubleshooting / Fixes**

### Check Role/RoleBinding

```
kubectl describe role <role> -n <ns>
kubectl describe rolebinding <rb> -n <ns>
```

### Check ClusterRoleBindings

```
kubectl get clusterrolebinding
```

### Check Current Identity

```
kubectl auth whoami
```

### Verify IAM Role Mapping via aws-auth

```
kubectl get cm aws-auth -n kube-system -o yaml
```

### Verify AWS Identity Inside Pod (IRSA)

```
kubectl exec -it <pod> -- aws sts get-caller-identity
```

---

## **Best Practices / Tips**

* Always test new RBAC changes using `kubectl auth can-i`.
* When testing ServiceAccounts, use the **full identity string**:

  ```
  ```

system:serviceaccount:<namespace>:<serviceaccount>

```
- Use `--list` to view complete permissions for auditing.
- Validate IAM Role mappings before debugging RBAC.
- Keep ServiceAccounts tightly scoped to their namespace.
- Combine `kubectl auth can-i` with `kubectl describe` for full visibility.
- Never assume RoleBinding applies outside its namespace.

---

```
---
---
---


