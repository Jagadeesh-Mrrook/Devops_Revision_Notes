# Distributed Build Setup in Jenkins (Detailed Notes)

## Concept / What

Distributed build setup in Jenkins means splitting responsibilities between the **controller (master)** and multiple **agents (nodes)**.

* **Controller:** Handles orchestration, scheduling, and managing plugins.
* **Agents:** Execute the builds and pipelines.

---

## Why / Purpose / Use Case in Real-World

* **Scalability:** Run many builds in parallel.
* **Different environments:** Agents can run different OS, tools, and software.

  * Example: Windows agents for .NET, Linux for Java.
* **Performance:** Offload heavy workloads from the controller.
* **Flexibility:** Add/remove agents as needed.
* **Cost optimization:** Use dynamic agents that spin up only when required.

---

## How it Works / Steps

1. Configure Jenkins controller (brain).
2. Add agents (static or dynamic).
3. Assign executors (parallel builds per agent).
4. Jenkins distributes jobs to agents based on labels/availability.
5. Monitor agents under **Manage Jenkins → Nodes and Clouds**.

---

## Real-Time Use Case

* A company uses:

  * **Windows agent** → .NET builds.
  * **Linux agent** → Java builds.
  * **Kubernetes pods** → containerized CI/CD pipelines.
* Jobs are distributed → faster builds, correct environments.

---

## Common Issues / Errors

* **Jobs stuck in queue:** No free agents / wrong label.
* **Agent not connecting:** Firewall, credentials, missing Java.
* **Agent offline often:** Network instability, low resources.
* **Load imbalance:** Some agents overloaded, others idle.

---

## Troubleshooting / Fixes

* Check agent logs: `Manage Jenkins → Nodes → Logs`.
* Verify network/port access (SSH, JNLP).
* Ensure Java installed on agents (if not containerized).
* Use node labels correctly in pipelines.
* For Kubernetes: verify RBAC, IAM roles, namespace, pod template.

---

## Best Practices / Tips

* Don’t run heavy builds on controller; use only for orchestration.
* Use **dynamic agents** (cloud/Kubernetes) for scalability & cost savings.
* Assign clear **labels** for environment-specific builds.
* Monitor CPU, memory, and disk usage of agents.
* Secure credentials with Vault/Ansible Vault instead of hardcoding.
* Keep agents lightweight with only required dependencies.


---

# Static vs Dynamic Agents in Jenkins (Detailed Notes)

## Concept / What

* **Static (Permanent) Agents**: Long-lived nodes created manually and kept registered in Jenkins. Always (or usually) online.
* **Dynamic (Ephemeral) Agents**: Provisioned on demand (via cloud plugins, Docker, etc.) and destroyed after the job finishes.

---

## Why / Purpose / Use Case in Real-World

* **Static agents**:

  * Useful for persistent environments (licensed software, GPU hardware, special OS setups).
  * Predictable build performance.
  * Suitable for small teams or jobs that require always-available environments.
* **Dynamic agents**:

  * Cost-efficient: spin up only when needed.
  * Provides clean environments per build.
  * Scales automatically to handle build spikes.
  * Good for shared Jenkins environments with many pipelines.

---

## How it Works / Steps

### Static Agent Setup (via SSH)

1. **Manage Jenkins → Manage Nodes and Clouds → New Node**

   * Choose **Permanent Agent**, click **OK**.
2. **Fill Node Configuration**:

   * Node name: `linux-agent-1`
   * # of executors: `1` (or based on CPU)
   * Remote root directory: `/home/jenkins`
   * Labels: `linux docker`
   * Usage: `Only build jobs with label`
   * Launch method: `Launch agent via SSH`
3. **SSH Launcher Settings**:

   * Host: `10.0.1.23`
   * Credentials: SSH username + private key
   * Java path (if needed): `/usr/bin/java`
4. Save → Jenkins will connect via SSH and start `agent.jar` on the node.

**Jenkinsfile example (static agent):**

```groovy
pipeline {
  agent { label 'linux' }
  stages {
    stage('Build') {
      steps { sh 'mvn clean install' }
    }
  }
}
```

### Dynamic Agent Setup (via Docker Plugin)

1. **Install Docker plugin** in Jenkins.
2. **Manage Jenkins → Configure Clouds → Add a new cloud → Docker**.
3. Provide Docker host URI (e.g., `tcp://docker-host:2376`).
4. Define Docker agent template:

   * Image: `jenkins/agent:latest`
   * Labels: `docker-agent`
   * Remote FS root: `/home/jenkins`
   * Container settings (volumes, memory, env vars as needed).
5. Save configuration.
6. When pipeline requests `agent { label 'docker-agent' }`, Jenkins launches a container and removes it after build.

**Jenkinsfile example (dynamic agent):**

```groovy
pipeline {
  agent { label 'docker-agent' }
  stages {
    stage('Test') {
      steps { sh 'echo Running on dynamic Docker agent' }
    }
  }
}
```

---

## Real-Time Use Cases

* **Static**: Dedicated machine with Oracle DB client installed; always available for integration tests.
* **Dynamic (Docker)**: On-demand containers to run unit tests in isolated, clean environments.
* **Dynamic (Cloud VMs)**: Jenkins spins up EC2 VMs for large-scale builds during peak hours and deletes them afterward.

---

## Common Issues / Errors

* **Static agents**:

  * Node offline (wrong credentials, firewall, Java missing).
  * Disk full on agent (builds fail).
  * Wrong labels (jobs stuck in queue).
* **Dynamic agents**:

  * Image not found or pull failure.
  * Docker host unreachable.
  * Container startup slow due to large images.
  * Job sits in queue if no matching template/label.

---

## Troubleshooting / Fixes

* **Static agents**:

  * Verify SSH connectivity: `ssh user@host`.
  * Ensure Java installed on agent: `java -version`.
  * Check agent logs: `Manage Jenkins → Nodes → [node] → Logs`.
  * Fix label mismatch by updating job or node labels.
* **Dynamic agents**:

  * Check Jenkins logs for Docker plugin errors.
  * Run `docker pull <image>` manually on host to validate image.
  * Reduce image size or pre-pull commonly used images.
  * Verify Docker host daemon status.

---

## Best Practices / Tips

* **Static**:

  * Use for jobs requiring specific hardware or licensed tools.
  * Limit executors to avoid overloading the machine.
  * Monitor disk, CPU, memory usage.
* **Dynamic**:

  * Keep images minimal and pre-cache dependencies.
  * Use consistent label naming.
  * Set idle timeout to auto-remove unused agents.
  * Monitor agent provisioning latency.
* General:

  * Don’t run builds on the Jenkins controller.
  * Secure agent credentials properly.
  * Regularly clean up workspaces and temp files on agents.

---

# Jenkins Agents Connection Methods (SSH & JNLP) – Detailed Notes

## Concept / What

* Jenkins agents connect to the controller to execute builds.
* Main connection methods:

  1. **SSH**: Controller initiates connection to agent.
  2. **JNLP**: Agent initiates connection to controller.

---

## Why / Purpose / Use Case in Real-World

* Distribute build execution to avoid overloading the controller.
* Allow different agents to have different software, OS, or tools.
* Support scaling builds and parallel execution.
* Ensure agents behind NAT/firewalls or external networks can connect.

---

## How it Works / Steps / Syntax

### 1. SSH Connectivity

* **Direction**: Controller → Agent
* **Steps**:

  1. Manage Jenkins → Nodes → New Node → Permanent Agent → Launch method = SSH.
  2. Configure Host, Credentials (SSH key), Remote root directory, Labels.
  3. Controller establishes SSH connection to agent (port 22).
  4. Controller copies `agent.jar` and launches agent process.
* **Protocols & Ports**: SSH, port 22.
* **Real-world setup**: Controller and agents in same VPC/private network, or connected via VPN/peering.

### 2. JNLP Connectivity

* **Direction**: Agent → Controller
* **Steps**:

  1. Manage Jenkins → Nodes → New Node → Launch method = Connect agent to controller.
  2. Controller provides command with JNLP URL and secret.
  3. Run the command on agent CLI: `java -jar agent.jar -jnlpUrl <URL> -secret <SECRET> -workDir <DIR>`
  4. Agent maintains connection with controller.
* **Protocols & Ports**: JNLP/WebSocket over HTTP(S) (ports 8080/443) and JNLP agent port 50000.
* **Real-world setup**: Controller in private subnet; agents outside VPC or behind NAT/firewalls initiate connection.

---

## Practical Guidelines

* Use **SSH** if controller can directly reach agent (same network/VPC).
* Use **JNLP** if agent is outside network or behind NAT/firewall.
* Run builds only on agents, not controller.
* Store credentials securely in Jenkins Credentials plugin.
* Monitor agent logs: Manage Jenkins → Nodes → Logs.
* Automate agent startup using init scripts, systemd, or container entrypoints.
* Use labels for assigning builds to appropriate agents.
* Limit executors per agent to avoid overload.

---

## Common Issues / Errors

* **SSH**: Wrong credentials, firewall blocks, Java not installed.
* **JNLP**: Secret mismatch, agent cannot reach controller URL, blocked ports, firewall/proxy interference.

## Troubleshooting / Fixes

* **SSH**: Test manual SSH, verify Java version, check agent logs.
* **JNLP**: Test URL access via curl, verify secret, check outbound firewall rules, monitor `agent.jar` process logs.

---

## Best Practices / Tips

* Never run builds on controller.
* SSH: preferred in trusted internal networks.
* JNLP: preferred for agents in external networks.
* Keep secrets in Jenkins Credentials plugin.
* Monitor agent connectivity regularly.


---

# Jenkins Agents via Kubernetes & Scaling with Kubernetes Plugin – Detailed Notes

## Concept / What

* Jenkins agents can be dynamically provisioned as **Kubernetes pods**.
* The **Kubernetes plugin** allows ephemeral agents to run builds and scale automatically.
* Each pod exists only for the duration of the build.

---

## Why / Purpose / Use Case in Real-World

* Avoid managing static VM agents.
* Supports auto-scaling depending on the number of queued builds.
* Each agent pod can have a **specific container image** with the required tools.
* Ideal for cloud-native CI/CD pipelines where resources are elastic.

---

## How it Works / Steps / Syntax

### Connecting Agents via Kubernetes

1. Install **Kubernetes plugin** in Jenkins.
2. Configure **Kubernetes Cloud**:

   * API endpoint (`https://<k8s-cluster>`).
   * Credentials (Kubeconfig or token).
   * Namespace for launching pods.
3. Define **Pod Templates**:

   * Container image (e.g., `jenkins/inbound-agent` or custom image).
   * Labels for job targeting.
   * Resource limits (CPU, memory).
4. Pipeline syntax to use Kubernetes agent:

```groovy
pipeline {
    agent {
        kubernetes {
            label 'java-build'
            containerTemplate {
                name 'jnlp'
                image 'jenkins/inbound-agent:latest'
                ttyEnabled true
            }
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean install'
            }
        }
    }
}
```

5. Build triggers:

   * Jenkins creates a pod dynamically.
   * Pod runs agent container and executes the job.
   * Pod is deleted after completion.

### Scaling Jenkins Builds with Kubernetes Plugin

* Jenkins automatically **creates pods on-demand** based on pipeline load.
* Supports multiple parallel jobs using isolated pods.
* After build completion, pods are removed, freeing resources.

---

## Common Issues / Errors

* Pod fails to start due to:

  * Wrong namespace or credentials.
  * Container image not found or inaccessible.
  * Resource limits too low → OOM or CPU throttling.
* Pod scheduling issues due to node selectors/taints.
* Networking issues: Jenkins controller cannot reach pod or pod cannot access required services.
* High build load → cluster resource limits reached → pod creation fails.

---

## Troubleshooting / Fixes

* Check Jenkins logs and pod logs (`kubectl logs <pod-name>`).
* Verify Kubernetes credentials and API connectivity.
* Ensure container image is accessible (registry permissions, correct tag).
* Adjust CPU/memory limits if pods fail due to resources.
* Monitor K8s cluster capacity and autoscaling policies.
* Verify node selectors and taints for pod scheduling.

---

## Best Practices / Tips

* Use lightweight container images for faster startup.
* Define proper resource requests and limits.
* Limit concurrent pod provisioning to avoid overloading cluster.
* Use labels to target specific jobs to specific pod templates.
* Use separate namespaces for Jenkins agents to isolate workloads.
* Keep pipelines modular to leverage ephemeral agent scaling effectively.
* Monitor pod lifecycle and cluster metrics regularly.

---


# Jenkins Node Labels & Node Selection in Pipelines – Detailed Notes

## Concept / What

* Node labels are tags assigned to Jenkins agents to indicate their environment or capabilities.
* Node selection allows jobs/pipelines to run on agents based on these labels.
* Helps control where builds execute in distributed setups.

---

## Why / Purpose / Use Case in Real-World

* Builds may require specific OS, tools, or versions (e.g., Java 17, Docker, Maven).
* Prevents running a job on an agent that cannot satisfy requirements.
* Organizes agent usage for parallel execution and resource allocation.
* Example labels:

  * `linux-java11` → Java 11 builds.
  * `windows-docker` → Windows Docker builds.
  * `high-mem` → resource-intensive builds.

---

## How it Works / Steps / Syntax

### Assign Labels to Nodes

* Go to **Manage Jenkins → Nodes → Select Node → Configure → Labels**.
* Enter one or multiple labels (space-separated).

### Use Labels in Pipeline

**Declarative Pipeline**

```groovy
pipeline {
    agent { label 'linux-java11' }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean install'
            }
        }
    }
}
```

**Scripted Pipeline**

```groovy
node('windows-docker') {
    stage('Build') {
        bat 'docker build -t myapp .'
    }
}
```

### Multiple Labels / Flexible Selection

* Scripted pipelines can use `||` (OR) or `&&` (AND) to select nodes:

```groovy
node('linux-java11 || linux-java17') {
    stage('Build') {
        sh 'mvn clean install'
    }
}
```

### Dynamic Agent Selection

* Combine with cloud-based agents (SSH, JNLP, Kubernetes) to provision agents dynamically with required labels.

---

## Common Issues / Errors

* Label doesn’t exist → “No nodes with the label found.”
* Multiple jobs queued for the same label → agents busy → builds delayed.
* Typo in label name → job fails to find agent.

---

## Troubleshooting / Fixes

* Verify labels in **Manage Jenkins → Nodes**.
* Ensure sufficient agents exist with required labels.
* For dynamic agents, ensure cloud/plugin configuration matches labels.
* Monitor agent availability and executors per node.

---

## Best Practices / Tips

* Use descriptive labels: OS + tool + version (e.g., `ubuntu-java11-maven3`).
* Assign multiple labels if an agent can serve multiple build types.
* Avoid overloading a single label → add more agents or labels.
* Combine labels with resource limits to prevent overutilization.
* Monitor queue times and adjust labels for optimized pipeline execution.


---


