# Kubernetes SecurityContext & Pod Security Standards (PSS) — Detailed Explanation Notes

---

## **1. Concept / What**

### **What is SecurityContext?**

SecurityContext is a set of rules inside a Pod or Container manifest that controls **how the pod or container should behave from a security point of view**. It decides things like:

* Should the container run as root or not?
* Should it have the ability to escalate privileges?
* Should its filesystem be read-only?
* Which Linux capabilities are allowed or removed?
* Which system calls can the container make?

It is this **SecurityContext** that Kubernetes checks when applying Pod Security Standards (PSS).

### **What is Pod Security Standards (PSS)?**

Pod Security Standards define **three security levels** that describe how safe a pod is allowed to be:

* **Privileged** — Full access, least secure
* **Baseline** — Medium restrictions, blocks highly dangerous settings
* **Restricted** — Very strict, production-grade secure setup

Kubernetes enforces these standards through **namespaces labels**, not through the YAML inside the pod.

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why SecurityContext?**

SecurityContext prevents dangerous workloads from running with:

* Root privileges
* High-risk capabilities
* Host access
* Ability to escape containers

It ensures pods run with **least privilege**, which is required for real-world security, compliance, and preventing attacks.

### **Why PSS?**

PSS ensures the entire namespace follows a consistent security posture by blocking unsafe pods. It:

* Protects production clusters from insecure workloads
* Enforces compliance (PCI-DSS, ISO, SOC2)
* Prevents human error or misconfiguration
* Ensures least privilege for containers

Real-world scenarios:

* Developers accidentally run containers as root → PSS blocks it
* Someone tries to deploy a privileged pod → PSS blocks it
* Third-party Helm charts violate security → DevOps fixes or overrides the manifest

---

## **3. How it Works / Steps / Syntax**

### **How SecurityContext Works**

SecurityContext is defined in two places:

* **Pod level:** Applies to all containers in the pod
* **Container level:** Applies only to that container (overrides pod level)

Example:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: app
    securityContext:
      runAsUser: 1000
```

Pod-level says "no root allowed", container-level says "user ID = 1000".

### **Important Fields Inside SecurityContext (Simple Explanation)**

* **runAsUser** → Run container as specific user ID
* **runAsNonRoot: true** → Kubernetes will block if container tries to run as root
* **allowPrivilegeEscalation: false** → Prevents container from becoming root later
* **privileged: true/false** → If true = full machine access (very risky)
* **capabilities.drop** → Remove dangerous Linux capabilities (recommended: drop ALL)
* **readOnlyRootFilesystem: true** → Container cannot write to its own filesystem
* **seccompProfile: RuntimeDefault** → Restricts system calls (recommended default)

Together, these fields define how safe your container is.

---

### **How PSS Works**

PSS is applied at the **namespace level**, not inside pod YAML.

Namespace labels decide what security level the namespace allows:

```bash
pod-security.kubernetes.io/enforce=restricted
pod-security.kubernetes.io/warn=baseline
pod-security.kubernetes.io/audit=restricted
```

### **Three PSS Modes**

* **enforce** → Block unsafe pods
* **warn** → Allow but show warning
* **audit** → Allow silently but log violations

Example:

```bash
kubectl label namespace prod pod-security.kubernetes.io/enforce=restricted
```

This means: only "restricted-level" safe pods can run in the `prod` namespace.

---

### **How SecurityContext + PSS Work Together**

SecurityContext = the **rules inside the pod**
PSS = the **rules enforced on the namespace**

PSS checks:

* “Does this pod follow the required safety level?”
* If namespace = restricted → pod must follow all restricted rules

If not → Kubernetes blocks the pod creation.

Example violation:

* Pod tries to run with `privileged: true`
* Namespace has `enforce=restricted`
* Kubernetes blocks the pod

---

## **4. Common Issues / Errors**

### **1. Pod blocked due to restricted mode**

Error: Pod violates restricted profile.
Occurs when:

* Running as root
* Privileged container used
* Capabilities added
* No seccomp profile

### **2. Developer workloads break after enabling PSS**

Because developers may not have added correct securityContext fields.

### **3. Third-party Helm charts fail**

Some charts assume root access or privileged mode.

---

## **5. Troubleshooting / Fixes**

* Add correct `securityContext` values (runAsNonRoot, drop capabilities, etc.)
* Change namespace PSS level if workload truly requires high privilege (rare)
* Override chart values with custom securityContext
* Use warn/audit modes to test before strict enforcement
* Help developers update YAML when their pods get blocked

---

## **6. Best Practices / Tips**

* Always use **runAsNonRoot** and **runAsUser** (non-root UID)
* Drop ALL capabilities unless absolutely required
* Enable **readOnlyRootFilesystem** for production workloads
* Use **seccompProfile: RuntimeDefault**
* Use **restricted** PSS level for production namespaces
* Developers write most manifests, DevOps enforces security
* Never allow privileged containers in prod unless fully justified
* Start enforcing with WARN → AUDIT → ENFORCE modes in sequence

---

## Final Summary (Easy to Remember)

* **SecurityContext** controls the container's permissions and behavior.
* **PSS** controls what kinds of pods the namespace is allowed to run.
* Security team sets policies, DevOps ENFORCES them.
* Developers write manifests, DevOps ensures compliance.
* Restricted PSS + correct securityContext = safest production setup.

---
---

# Kubernetes Network Policies — Detailed Explanation Notes

---

## **1. Concept / What**

### **What are Network Policies?**

Network Policies in Kubernetes act like **firewalls for Pods**. They control:

* What traffic can **enter** a pod (Ingress)
* What traffic can **leave** a pod (Egress)
* Which pods can communicate with which pods
* Whether pods can access the internet or other namespaces

**Without Network Policies:**

* All pods can communicate with all pods
* All pods have full internal and external network access
* There is no isolation

Network Policies provide **network-level security and isolation**.

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why Network Policies?**

Network Policies are essential for:

* Securing communication between microservices
* Preventing compromised pods from attacking others
* Achieving Zero-Trust networking
* Restricting database access to specific services only
* Blocking unwanted internal movement inside the cluster
* Restricting outgoing (egress) traffic

### **Real-World Use Cases**

* Only allow Frontend → Backend communication
* Only Backend can access the database
* Block all access to sensitive pods
* Deny internet access for specific pods
* Allow pods to talk only to DNS but nothing else
* Enforce namespace isolation

---

## **3. How it Works / Steps / Syntax**

### **Basic Behavior**

A NetworkPolicy becomes effective when:

1. **It selects pods using podSelector**
2. **It has policyTypes (Ingress/Egress)**

After that, traffic in the specified directions is **blocked by default**, unless explicitly allowed.

### ✔ Ingress policy → Blocks all incoming traffic

### ✔ Egress policy → Blocks all outgoing traffic

### ✔ Both → Blocks everything (full default deny)

### **Components of a NetworkPolicy**

1. **podSelector** – selects which pods the policy applies to
2. **policyTypes** – Ingress / Egress / Both
3. **ingress rules** – who can talk to the selected pods
4. **egress rules** – where selected pods can send traffic
5. **namespaceSelector / podSelector** – selects allowed peers

---

## **4. Examples and Syntax**

### **1️⃣ Block All Ingress Traffic (Default Deny Ingress)**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

**Meaning:** No one can connect to ANY pod in this namespace.

---

### **2️⃣ Allow Only Frontend → Backend**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```

**Meaning:** Only pods with `app=frontend` can access the backend pods.

---

### **3️⃣ Allow Database Traffic Only From Backend**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - protocol: TCP
      port: 5432
```

**Meaning:** Only backend pods can connect to DB on port 5432.

---

### **4️⃣ Block ALL Egress (No Internet Access)**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress: []
```

**Meaning:** Pods cannot reach the internet or other services.

---

## **5. Common Issues / Errors**

### **1. NetworkPolicy does NOTHING**

* CNI plugin does not support NetworkPolicies
* Example: Flannel does NOT support them
* Fix: Use Calico / Cilium / Weave

### **2. DNS stops working**

If egress is blocked, pods may lose DNS access.
Fix: Allow DNS explicitly:

```yaml
egress:
- to:
  - namespaceSelector: {}
  ports:
  - protocol: UDP
    port: 53
```

### **3. Accidentally blocking internal communication**

* Happens when no ingress rules are defined
* Fix: Add required allow rules

### **4. Developers complain services stopped talking**

* NetworkPolicy blocked traffic
* DevOps must add appropriate allow rules

---

## **6. Troubleshooting / Fixes**

* Check podSelector — wrong labels mean policy doesn’t apply
* Verify CNI supports NetworkPolicies
* Add DNS egress rules
* Use `kubectl describe networkpolicy` to debug
* Test in staging before applying in prod
* Add allow rules gradually (don’t block everything suddenly)

---

## **7. Best Practices / Tips**

* Start with **default deny** and add allow rules
* Use clear labels for selecting pods
* Keep policies small and specific
* Avoid mixing unrelated traffic rules in one policy
* Test policies carefully using `kubectl exec` commands
* Prefer Calico or Cilium for production
* Document all communication paths between services

---

## **Final Summary (Easy to Remember)**

* NetworkPolicy = firewall for pods
* Ingress controls who can reach the pod
* Egress controls where the pod can go
* Default deny happens ONLY for selected pods and ONLY for specific directions
* Allow rules open required traffic paths
* Supported only by NetworkPolicy-aware CNIs
* Extremely important for Zero Trust and cluster security

---
---

# Restricting Privileged Containers — Detailed Explanation Notes

---

## **1. Concept / What**

### **What are Privileged Containers?**

A **privileged container** is a container that has almost the same permissions as the **host machine**. It can:

* Access the host filesystem
* Access all host devices
* Modify kernel settings
* Load kernel modules
* Escape the container boundary
* Perform root-level operations on the node

This makes privileged containers extremely risky.

### **What is Restricting Privileged Containers?**

It means preventing pods from running with:

```yaml
securityContext:
  privileged: true
```

so that containers cannot take full control of the Kubernetes node.

Restricting privileged containers is a critical part of Kubernetes security.

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why restrict privileged containers?**

Privileged containers are dangerous because they can:

* Break out of their container
* Modify or delete host-level files
* Disable system security controls
* Attack or spy on other pods
* Move laterally inside a cluster
* Gain full administrator-level control of the worker node

### **Why real companies avoid them?**

In production:

* Almost **no application** should be privileged
* Only system-level components need it
* Security teams forbid privileged mode except in rare system-specific cases

### **Legitimate Use Cases**

Privileged containers are used only for:

* CNI plugins (Calico, Cilium)
* CSI plugins (storage drivers)
* Monitoring agents with host-level access
* Debugging node issues (very rare)

Normal workloads never need privileged mode.

---

## **3. How it Works / Steps / Syntax**

### **How to Restrict Privileged Containers**

There are **three main ways**:

---

### **1️⃣ Enforce PSS (Pod Security Standards)**

Setting the namespace to **restricted** mode automatically blocks privileged pods:

```bash
kubectl label namespace prod pod-security.kubernetes.io/enforce=restricted
```

Restricted PSS blocks:

* privileged: true
* allowPrivilegeEscalation: true
* running as root
* extra Linux capabilities

This is the simplest and most effective method.

---

### **2️⃣ Use a Safe SecurityContext**

Ensure your pod YAML does NOT contain privileged settings:

```yaml
securityContext:
  privileged: false
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

This enforces least-privilege behavior.

---

### **3️⃣ Use Policy Engines (Kyverno or Gatekeeper)**

Large companies use policies to deny privileged containers:

#### Example Kyverno Policy:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-privileged-containers
spec:
  validationFailureAction: enforce
  rules:
  - name: check-privileged
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Privileged containers are not allowed."
      pattern:
        spec:
          containers:
          - securityContext:
              privileged: "false"
```

This ensures that no user can deploy privileged pods.

---

## **4. Common Issues / Errors**

### **1. Pod Blocked by PSS**

Restricted PSS rejects privileged containers.
Fix: Remove privileged or adjust securityContext.

### **2. Third-party Helm Charts Break**

Some charts assume root or privileged access.
Fix: Apply overrides to securityContext.

### **3. System Components Need Privilege**

CNI/CSI plugins may require privileged mode.
Fix: Allow privileged workloads only in system namespaces like:

* kube-system
* calico-system
* cilium-system

---

## **5. Troubleshooting / Fixes**

* Check PSS enforcement level
* Check securityContext configuration
* Review network/host access requirements
* Inspect logs using `kubectl describe pod`
* Apply overrides in Helm values if needed

---

## **6. Best Practices / Tips**

* Never allow privileged containers in production
* Use PSS=restricted on all prod namespaces
* Drop all Linux capabilities unless required
* Always disable privilege escalation
* Limit privileged pods only to system namespaces
* Use policy engines for extra enforcement
* Audit cluster regularly for privileged pods

---

## **Final Summary (Easy to Remember)**

* Privileged containers = full host access → extremely dangerous
* Use PSS restricted + correct securityContext to block them
* Only system components ever need privilege
* Application pods should always run with least privilege

---
---

# Image Security & Scanning (ECR) — Detailed Explanation Notes

---

## **1. Concept / What**

### **What is Image Security?**

Image security refers to validating and securing Docker container images before using them in Kubernetes. This includes:

* Checking for vulnerabilities (CVEs)
* Ensuring no outdated OS packages
* Detecting unsafe libraries or malware
* Ensuring minimal and safe base images

### **What is Image Scanning?**

Image scanning automatically checks each image layer for vulnerabilities. AWS ECR (Elastic Container Registry) provides:

* **Basic Scanning** (on push)
* **Enhanced Scanning** (continuous, real-time via Amazon Inspector)

This prevents unsafe images from getting deployed into Kubernetes clusters.

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why do we need Image Scanning?**

Container images often include:

* Outdated packages
* Vulnerable libraries (e.g., log4j)
* Misconfigured OS layers
* Third-party components that may not be trusted

If these vulnerabilities reach pod containers, attackers can exploit them and gain access to the cluster.

### **Real-World Use Cases**

* Scan every image before pushing it to production
* Avoid vulnerable public base images (ubuntu/latest, alpine, etc.)
* Ensure compliance for audits (PCI, ISO, SOC2)
* Detect risks early in CI/CD pipeline
* Block images with Critical/High vulnerabilities

---

## **3. How it Works / Steps / Syntax**

### **Step-by-Step Workflow**

1. Developer builds an image locally or in CI:

   ```bash
   docker build -t myapp:1.0 .
   ```
2. DevOps pushes the image to ECR:

   ```bash
   docker push <account-id>.dkr.ecr.<region>.amazonaws.com/myapp:1.0
   ```
3. ECR scans the image (Basic or Enhanced scanning).
4. ECR generates a vulnerability report (Critical, High, Medium, Low).
5. DevOps reviews & fixes issues before deploying.

---

## **Types of ECR Scanning**

### **1️⃣ Basic Scanning (Default)**

* Triggered on image push
* Checks for known vulnerabilities
* Uses AWS-managed database
* Good for small teams

### **2️⃣ Enhanced Scanning (Recommended)**

* Uses Amazon Inspector
* Continuous scanning (even after push)
* Real-time alerts when new vulnerabilities appear
* Tracks vulnerabilities across all repos
* Provides detailed package-level insights

---

## **4. How to Enable ECR Scanning**

### **Enable Basic Scanning**

```bash
aws ecr put-image-scanning-configuration \
  --repository-name myrepo \
  --image-scanning-configuration scanOnPush=true
```

### **Enable Enhanced Scanning**

```bash
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED
```

---

## **5. Understanding Scan Results**

ECR scan results show:

* Package name
* CVE ID
* Severity (Critical / High / Medium / Low)
* Whether a fix exists

### **Typical Severity Rules in Companies:**

* **CRITICAL:** Must be fixed immediately
* **HIGH:** Fix before production deployment
* **MEDIUM/LOW:** Acceptable for dev environments

---

## **6. How DevOps Fixes Vulnerabilities**

### **Fix 1 — Use minimal base images**

* Use `alpine`, `distroless`, `ubi-minimal`

### **Fix 2 — Update OS packages**

```bash
apt-get update && apt-get upgrade -y
```

### **Fix 3 — Update application dependencies**

* npm update
* pip upgrade
* go mod tidy

### **Fix 4 — Pin safe versions of libraries**

* Avoid `latest` tag
* Specify exact package versions

### **Fix 5 — Rebuild & push again**

Re-run scans until vulnerabilities reduce.

---

## **7. How ECR Scanning Fits Into Kubernetes Security**

* Even with PSS and Network Policies, a vulnerable image can compromise the cluster.
* Image scanning is the **first line of defense**.
* Most companies use **multiple layers of scanning**:

  * Trivy in CI/CD
  * ECR scan on push
  * Enhanced continuous scanning

---

## **8. Common Issues / Troubleshooting**

### **1. Scanning doesn’t trigger**

* Cause: `scanOnPush` disabled
* Fix: Enable basic/enhanced scanning

### **2. Too many vulnerabilities**

* Cause: Using outdated base images
* Fix: Switch to minimal OS images

### **3. No fixes available for some CVEs**

* Fix: Change base image entirely

### **4. Enhanced Scanning shows extra vulnerabilities**

* Fix: Prioritize Critical/High first

---

## **9. Best Practices / Tips**

* Always enable Enhanced Scanning for ECR
* Treat ECR scan results like a deployment gate
* Block builds with Critical vulnerabilities
* Use Trivy in CI before pushing image
* Avoid unnecessary packages in Dockerfile
* Rebuild images periodically for patch updates
* Use trusted base images from official repos
* Keep images small and minimal

---

## **Final Summary (Easy to Remember)**

* Image scanning ensures container safety before Kubernetes deployment.
* ECR offers Basic & Enhanced scanning.
* CI tools like Trivy add extra security.
* Only deploy images with no Critical vulnerabilities.
* Image scanning is a mandatory part of DevSecOps.

---
---


# Secret Encryption using KMS + External Secrets Operator (ESO) — Detailed Explanation Notes

---

## **1. Concept / What**

### **What is Secret Encryption in Kubernetes?**

Kubernetes stores secrets in **ETCD**, and by default they are only **base64-encoded**, not encrypted. To securely store secrets, Kubernetes must encrypt them at rest using **AWS KMS**.

### **What is AWS Secrets Manager + External Secrets Operator (ESO)?**

Instead of storing secrets in Kubernetes, you can store your secrets in **AWS Secrets Manager**, and use **External Secrets Operator (ESO)** to fetch them securely into the cluster. ESO creates Kubernetes Secrets internally, but the *source of truth* remains outside the cluster.

Both together provide:

* Secure storage of secrets outside K8s (AWS SM)
* Secure encryption of secrets inside K8s (KMS)
* Automatic rotation
* No secrets inside GitHub or manifests

---

## **2. Why / Purpose / Real-World Use Cases**

### **Why use KMS Encryption in Kubernetes?**

* Kubernetes stores Secrets inside ETCD
* Base64 is not encryption
* ETCD compromise exposes secrets
* KMS encryption protects data at rest
* Helps meet compliance (PCI, HIPAA, SOC2)

### **Why use AWS Secrets Manager with ESO?**

* Secrets are stored *outside* the cluster
* No raw secrets inside GitHub or YAML files
* AWS SM encrypts secrets using KMS automatically
* ESO syncs secrets securely via HTTPS
* Automatic rotation without redeploying pods
* IAM-based access control using IRSA

This approach is more secure than using native Kubernetes Secrets alone.

---

## **3. How It Works / Steps / Syntax**

### **A. Kubernetes Secrets + KMS Encryption-at-Rest**

1. You enable encryption for secrets in EKS (during cluster creation or update).
2. EKS configures the API server to use the selected KMS Key.
3. Whenever a secret is created, Kubernetes encrypts it **before writing into ETCD**.
4. When a pod requests a secret, API server decrypts it using KMS.

### **B. AWS Secrets Manager + External Secrets Operator**

This is the modern approach.

#### **Step-by-step flow:**

1. Secret stored in AWS Secrets Manager (encrypted via KMS).
2. External Secrets Operator (ESO) inside Kubernetes authenticates to AWS using **IRSA**.
3. ESO fetches secrets from AWS SM over **HTTPS**.
4. AWS Secrets Manager **decrypts the secret internally** (only AWS has the KMS permissions).
5. AWS SM returns **plaintext secret over TLS** to ESO.
6. ESO creates a **Kubernetes Secret** inside the cluster.
7. Kubernetes stores this Secret in **ETCD**, and if encryption-at-rest is enabled, KMS encrypts it again.

### **ESO CRDs Installed**

ESO installs 3 CRDs:

* **SecretStore** – defines the external provider (AWS Secrets Manager)
* **ClusterSecretStore** – provider configuration for all namespaces
* **ExternalSecret** – defines which external secret to fetch and which K8s Secret to create

Example ExternalSecret:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secret
spec:
  secretStoreRef:
    name: aws-secret-store
  target:
    name: app-secret
  data:
  - secretKey: password
    remoteRef:
      key: prod/db/password
```

---

## **4. Common Issues / Errors**

### **1. KMS not configured for EKS secrets**

* Secrets remain base64-encoded
* Fix: Enable EKS Secrets Encryption

### **2. ESO cannot fetch secrets**

* Incorrect IAM permissions for IRSA
* Required permissions:

  * `secretsmanager:GetSecretValue`

### **3. Secrets stored in ETCD unencrypted**

* KMS encryption disabled
* Fix: Recreate secrets after enabling encryption

### **4. Secret rotation not reflected**

* ESO sync interval too long
* Fix: Update ESO refresh interval

---

## **5. Troubleshooting / Fixes**

* Ensure IRSA role has correct AWS permissions
* Enable EKS encryption-at-rest for Secrets
* Re-sync ExternalSecret to refresh data
* Inspect ESO pod logs for AWS API errors
* Verify HTTPS access to AWS SM endpoints

---

## **6. Best Practices / Tips**

* Prefer storing secrets in **AWS Secrets Manager**, not in Kubernetes
* Use **IRSA** for authenticating ESO to AWS
* Enable **auto rotation** for KMS keys and Secrets Manager secrets
* Avoid storing secrets in Git, YAML, or ConfigMaps
* Re-encrypt secrets after enabling or rotating KMS keys
* Grant least-privilege IAM policies to ESO
* Ensure ETCD encryption is enabled for Kubernetes Secrets

---

## **Final Summary (Easy to Remember)**

* Kubernetes Secrets are base64-encoded unless encrypted using KMS.
* EKS can encrypt Secrets at rest using your KMS keys.
* AWS Secrets Manager stores secrets securely with built-in KMS encryption.
* External Secrets Operator fetches decrypted secrets via HTTPS and creates K8s Secrets.
* Kubernetes then stores them in ETCD, which is again encrypted if enabled.
* This setup offers multiple layers of security and avoids putting secrets into Git.

---
---

# IRSA (IAM Roles for Service Accounts) & OIDC in EKS — Detailed Explanation Notes

---

## **Concept / What**

### **What is OIDC?**

OIDC (OpenID Connect) is an *identity protocol* that defines how identity tokens (JWTs) should be issued and verified. It is *not* an AWS service.

### **OIDC Provider (IDP)**

The OIDC Provider is the component that exposes an **issuer URL** and **public keys** used to verify signed tokens. In EKS, AWS automatically provides the OIDC provider for the cluster.

### **ServiceAccount Identity & Pod Identity**

In Kubernetes, a ServiceAccount acts as the identity for a pod. When a pod starts, Kubernetes generates a **signed JWT token** for that ServiceAccount.

### **IRSA (IAM Roles for Service Accounts)**

IRSA is the mechanism that allows Kubernetes pods to assume IAM roles **securely** without using static AWS access keys. The pod's OIDC-issued JWT token is used as proof of identity.

---

## **Why / Purpose / Real-World Use Case**

* Pods often need access to AWS services (S3, Secrets Manager, DynamoDB, CloudWatch, etc.).
* Storing AWS access keys inside pods is insecure.
* IRSA solves this by:

  * Giving pods a **Kubernetes identity** (ServiceAccount)
  * Using **OIDC-issued signed JWT tokens** to prove pod identity
  * Allowing AWS to **verify** the token and give **temporary credentials** safely
* Used widely in production to prevent key leakage and enforce least privilege.

---

## **How it Works / Steps / Syntax**

### **1. EKS Creates an OIDC Provider Automatically**

* When an EKS cluster is created, AWS automatically provisions an OIDC issuer URL.
* Example:

```
https://oidc.eks.<region>.amazonaws.com/id/<OIDC-ID>
```

* You do **not** create or configure this; it is built-in.

### **2. Kubernetes Generates the JWT Token**

* A ServiceAccount is created (your pod identity inside Kubernetes).
* Pod uses this ServiceAccount:

```yaml
spec:
  serviceAccountName: myapp-sa
```

* When the pod starts:

  * Kubernetes generates a **JWT token**
  * Kubernetes **signs** the token using the **EKS OIDC provider’s signing keys**
  * Token includes:

    * issuer URL
    * namespace
    * ServiceAccount name
    * expiration time
  * Token is stored at:

```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

### **3. IAM Trust Policy (WHO is allowed to assume the role)**

This trust policy must include **two things**:

#### **A. OIDC Provider ARN**

Tells AWS which OIDC provider to trust.

```json
"Federated": "arn:aws:iam::<account-id>:oidc-provider/oidc.eks.<region>.amazonaws.com/id/<OIDC-ID>"
```

#### **B. ServiceAccount Identity Condition**

Tells AWS which ServiceAccount is allowed.

```json
"Condition": {
  "StringEquals": {
    "oidc.eks.<region>.amazonaws.com/id/<OIDC-ID>:sub": "system:serviceaccount:default:myapp-sa"
  }
}
```

### **4. IAM Permission Policy (WHAT the pod can do)**

This policy defines actions the pod is allowed to perform:

```json
"Action": [
  "s3:ListBucket",
  "s3:GetObject",
  "secretsmanager:GetSecretValue"
]
```

### **5. Annotate the ServiceAccount with the IAM Role**

```yaml
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/myapp-role
```

This links the Kubernetes identity → AWS identity.

---

## **Common Issues / Errors**

* **OIDC provider not added to IAM** → AWS rejects JWT.
* **Wrong SA or namespace in trust policy** → AccessDenied.
* **Wrong annotation on ServiceAccount** → Pods cannot assume role.
* **Permission policy missing required actions** → SDK errors.
* **Expired JWT token** → Role assumption failure.
* **Using default SA accidentally** → No IRSA identity mapping.

---

## **Troubleshooting / Fixes**

* Run:

```
aws iam list-open-id-connect-providers
```

to ensure OIDC provider exists.

* Verify ServiceAccount annotation:

```
kubectl get sa myapp-sa -o yaml
```

* Decode pod JWT to confirm claims:

```
cat /var/run/secrets/kubernetes.io/serviceaccount/token | jq -R 'split(".") | .[1] | @base64d'
```

* Check trust policy formatting and correct issuer.

* Ensure correct namespace + SA name in `sub` claim.

---

## **Best Practices / Tips**

* Use **unique ServiceAccounts** per application.
* Use **least privilege** IAM permission policies.
* Never attach credentials via environment variables.
* Avoid using the default ServiceAccount.
* Rotate IAM roles and clean unused ones.
* Review IAM trust policies carefully — they control WHO can assume roles.

---

## **End of Notes**

---
---

# Security Assessment Tools (kube-bench) — Detailed Explanation Notes

---

## **Concept / What**

**kube-bench** is a Kubernetes security auditing tool that checks whether a cluster follows the **CIS Kubernetes Benchmark** — a widely accepted security standard defining best practices for configuring Kubernetes components such as:

* API Server
* Kubelet
* Controller Manager
* Scheduler
* etcd
* Certificate & file permissions

It analyzes your cluster’s security posture and reports **PASS / FAIL / WARN** for each rule.

---

## **Why / Purpose / Use Case in Real-World**

* Ensures the cluster is following industry security standards.
* Detects weak or risky configurations early.
* Helps maintain audits for **SOC2, PCI, HIPAA, ISO 27001** and other compliance frameworks.
* Used mainly by **security teams**, not daily DevOps work.
* DevOps engineers typically need **awareness**, not deep CIS-level expertise.

Common real-world uses:

* Pre-production security review
* Cluster hardening exercises
* Periodic compliance checks
* Detecting misconfigurations after upgrades

---

## **How it Works / Steps / Syntax**

### **How kube-bench works (high-level):**

1. kube-bench inspects Kubernetes component configurations:

   * Flags passed to API server
   * Kubelet arguments
   * Scheduler & Controller Manager settings
   * TLS configuration
   * File permissions on certs/configs
2. Compares them with CIS Benchmark rules.
3. Produces a detailed security report.

### **Run kube-bench directly:**

```
kube-bench --version 1.28
```

### **Run kube-bench inside Kubernetes as a Job:**

```
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
```

### **Output includes:**

* **PASS:** configuration matches CIS Benchmark
* **FAIL:** configuration violates CIS Benchmark
* **WARN:** potentially risky setting

---

## **Common Issues / Errors**

* Anonymous authentication enabled in API Server
* Insecure Kubelet configuration (e.g., read-only port enabled)
* Missing or weak audit logging
* Wrong file permissions on certificates
* Missing secure TLS configurations
* Unsecured controller or scheduler flags

---

## **Troubleshooting / Fixes**

* Review FAIL section in kube-bench output.
* Apply recommended CIS remediation steps.
* Coordinate with **security teams** for cluster-wide changes.
* Some CIS rules do **not** apply fully to **managed clusters like EKS**, so interpret results carefully.
* Re-run kube-bench after fixes to verify compliance.

---

## **Best Practices / Tips**

* kube-bench should be owned by **security teams**, DevOps assists when required.
* Run in **staging + production** clusters periodically.
* Integrate with CI/CD only if your organization has strong security requirements.
* Treat kube-bench as an **audit tool**, not a real-time monitoring tool.
* Do not expect 100% PASS in EKS — some CIS checks are handled by AWS internally.

---

## **End of Notes**

---
---

# Security Assessment Tool — kube-hunter (Detailed Explanation Notes)

---

## **Concept / What**

kube-hunter is a **Kubernetes penetration testing tool** designed to identify real attack vectors in a Kubernetes cluster. Unlike kube-bench—which checks configuration compliance—kube-hunter actively probes the cluster the way an attacker might.

It looks for runtime vulnerabilities such as:

* Open or insecure ports
* Exposed dashboards
* Anonymous API access
* Misconfigured Kubelet
* Privilege escalation paths
* Access to cloud metadata endpoints

---

## **Why / Purpose / Use Case in Real-World**

* Helps discover vulnerabilities **before attackers do**.
* Supports internal security audits and red-team assessments.
* Commonly used during:

  * Pre-production cluster validation
  * Periodic security reviews
  * Compliance audits
* Mostly owned by **security teams**. DevOps engineers only need working awareness.

---

## **How it Works / Steps / Syntax**

kube-hunter can perform two types of scans:

### **1. Remote Scan** (Outside the Cluster)

Scans exposed interfaces:

```
kube-hunter --remote <cluster-ip-or-hostname>
```

This identifies:

* Open Kubernetes Dashboard
* Anonymous API server access
* Exposed services over public endpoints

### **2. In-Cluster Scan** (Inside the Cluster)

Run as a Kubernetes Job:

```
kubectl apply -f kube-hunter-job.yaml
```

This identifies:

* Internal network vulnerabilities
* Access to cloud provider metadata API
* Kubelet unauthenticated ports
* Privilege escalation vectors
* Weak RBAC permissions

### **Output Categories:**

* **INFO** – Informational findings
* **LOW** – Low-risk vulnerabilities
* **MEDIUM** – Moderate security concerns
* **HIGH** – Serious vulnerabilities requiring immediate attention

---

## **Common Issues / Errors kube-hunter Detects**

* Kubernetes Dashboard exposed without authentication
* Kubelet API open or partially secured
* Anonymous API server access enabled
* Metadata API accessible from pods
* Misconfigured RBAC enabling privilege escalation
* Unprotected internal services

---

## **Troubleshooting / Fixes**

* Repair HIGH severity issues immediately.
* Restrict access to open ports and endpoints.
* Secure Kubernetes Dashboard or disable it.
* Harden RBAC policies.
* Block pod access to metadata APIs when unnecessary.
* Work with **security teams** for remediation.

---

## **Best Practices / Tips**

* Do not run kube-hunter without security team approval—it performs active tests.
* Use it in **staging and production** periodically.
* Combine kube-hunter (runtime scanning) with kube-bench (configuration auditing).
* DevOps responsibility is **awareness + supporting fixes**, not owning full security testing.
* Treat kube-hunter as an **audit tool**, not a continuous monitoring system.

---

## **End of Notes**

---
---
---


