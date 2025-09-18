# Distributed Build Setup in Jenkins (Quick Revision)

## Concept / What

* Jobs run on multiple agents instead of controller.
* Controller schedules and manages jobs; agents execute builds.

---

## Why / Purpose

* Avoid controller overload.
* Run builds in different environments (Windows/Linux/Docker).
* Parallel builds for faster CI/CD.
* Dynamic agents reduce cost in cloud/Kubernetes setups.

---

## How

* Configure Jenkins controller.
* Add agents (static/dynamic).
* Assign executors (parallel builds per agent).
* Jobs distributed based on labels and availability.
* Monitor agents in `Manage Jenkins → Nodes and Clouds`.

---

## Real-Time Use Case

* Windows agent → .NET builds.
* Linux agent → Java builds.
* Kubernetes pods → containerized pipelines.
* Faster builds, correct environment execution.

---

## Common Issues

* Jobs stuck (no free agents/wrong labels).
* Agent connection failures (firewall, credentials, missing Java).
* Agent offline often (network/resources).
* Load imbalance among agents.

---

## Troubleshooting

* Check agent logs.
* Verify SSH/JNLP connectivity.
* Ensure Java installed on agents.
* Use correct node labels.
* Kubernetes: check RBAC, IAM, pod templates.

---

## Best Practices

* Avoid heavy builds on controller.
* Use dynamic agents for scalability.
* Assign clear labels.
* Monitor CPU/memory/disk usage.
* Secure credentials (Vault/Ansible Vault).
* Keep agents lightweight.


---

## Quick Revision Notes – Jenkins Agents

### Static Agents

* **Definition**: Pre-configured machines (physical/VM) permanently attached to Jenkins.
* **When to Use**: Predictable workloads, long-running jobs, fixed infra.
* **Setup**:

  * Manage Jenkins → Nodes → New Node.
  * Enter name, select Permanent Agent.
  * Configure Remote root dir, Labels, Launch method (SSH/JNLP).

### Dynamic Agents

* **Definition**: On-demand agents provisioned via cloud/plugins.
* **When to Use**: Variable workloads, scale up/down automatically.
* **Setup**:

  * Install plugin (e.g., EC2, Docker).
  * Configure Cloud in Jenkins.
  * Define agent template (AMI/image, labels, launch method).
  * Jenkins provisions/destroys agents per job demand.

### Key Difference

* **Static** = Always available, manual scaling.
* **Dynamic** = Elastic, cost-efficient, automated scaling.

### Logs

* Agent connection issues → Manage Jenkins → Nodes → Select Node → Logs.
* Shows connection attempts, SSH/JNLP errors, heartbeats.


---

# Jenkins Agents Connection Methods – Quick Revision

* **SSH Connectivity**

  * Direction: Controller → Agent
  * Port: 22
  * Best for: Agents in same network/VPC as controller
  * Steps: Manage Jenkins → Nodes → New Node → Permanent Agent → SSH → Configure Host, Credentials, Root Dir, Labels → Controller connects
  * Issues: Wrong credentials, firewall, Java missing
  * Fixes: Manual SSH test, check Java, check agent logs

* **JNLP Connectivity**

  * Direction: Agent → Controller
  * Ports: 50000 + 8080/443
  * Best for: Agents outside VPC / behind NAT/firewall
  * Steps: Manage Jenkins → Nodes → New Node → Connect agent to controller → Copy command → Run on agent CLI
  * Issues: Secret mismatch, blocked ports, firewall/proxy
  * Fixes: Test URL, verify secret, check outbound firewall, monitor agent process

* **Practical Guideline**

  * Same network/VPC → SSH
  * Agent outside network → JNLP
  * Controller usually in private subnet
  * Always run builds on agents, not controller
  * Use labels to assign builds
  * Monitor agent connectivity
  * Secure credentials in Jenkins Credentials plugin


---

# Jenkins Agents via Kubernetes & Scaling – Quick Revision

* **Kubernetes Agents**:

  * Agents are launched as ephemeral **pods** using Jenkins Kubernetes plugin.
  * Pods exist only for the duration of the build.
  * Configure Kubernetes Cloud in Jenkins: API endpoint, credentials, namespace.
  * Define Pod Templates: container image, label, resource limits.
  * Pipeline example: use `label` in `agent { kubernetes { ... } }`.
  * Benefits: dynamic scaling, isolated builds, no static VMs.

* **Scaling Builds with Kubernetes Plugin**:

  * Jenkins creates pods **on-demand** depending on job queue.
  * Pods deleted after build completion → resources freed.
  * Supports parallel builds with isolated environments.
  * Monitor cluster capacity and set resource limits.

* **Common Issues**:

  * Pod creation fails (wrong namespace/credentials, image not found, low resources).
  * Node selectors/taints misconfigured.
  * Networking issues between controller and pods.

* **Troubleshooting**:

  * Check Jenkins logs and `kubectl logs <pod>`.
  * Verify credentials, namespace, and pod image accessibility.
  * Adjust CPU/memory requests & limits.
  * Monitor cluster autoscaling and concurrency.

* **Best Practices**:

  * Lightweight pod images, proper resource limits.
  * Use labels to target builds.
  * Separate namespaces for Jenkins pods.
  * Monitor agent lifecycle and cluster metrics.

* **Interview Shortcut Line:**

  > “Jenkins Kubernetes plugin dynamically provisions ephemeral pods for builds and scales automatically depending on job load.”


---

# Jenkins Node Labels & Node Selection – Quick Revision

* **Node Labels:**

  * Tags assigned to Jenkins agents to indicate OS, tools, or capabilities.
  * Used to select which agent executes a build.

* **Use Cases:**

  * Ensure builds run on compatible agents.
  * Manage parallel execution and resource allocation.
  * Example: `linux-java11`, `windows-docker`, `high-mem`.

* **Pipeline Usage:**

  * Declarative: `agent { label 'label-name' }`
  * Scripted: `node('label-name') { ... }`
  * Multiple labels: `node('label1 || label2')`

* **Common Issues:**

  * Label does not exist → build fails.
  * Agents busy → builds queued.
  * Typo in label name → job fails.

* **Fixes / Tips:**

  * Verify labels in Manage Jenkins → Nodes.
  * Ensure enough agents per label.
  * Assign multiple labels to agents.
  * Use descriptive label names: OS + tool + version.
  * Monitor queue times and adjust agent allocation.

* **Interview Shortcut Line:**

  > “Node labels let you target builds to specific agents with the required OS, tools, or capabilities.”

