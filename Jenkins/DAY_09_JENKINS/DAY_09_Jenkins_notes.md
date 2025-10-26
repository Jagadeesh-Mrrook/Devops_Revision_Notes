# Jenkins Monitoring & Maintenance

## 🧾 Detailed Explanation Version

### **Concept / What**

Monitoring and maintenance in Jenkins refer to the continuous process of tracking Jenkins’ performance, system health, build status, and configurations while performing regular cleanup, backups, and troubleshooting. It ensures Jenkins remains stable, performant, and reliable for continuous integration and delivery tasks.

---

### **Why / Purpose / Use Case in Real-World**

* **Reliability:** Prevent unexpected failures or downtime in production Jenkins setups.
* **Performance:** Optimize builds and prevent resource exhaustion (CPU, memory, disk).
* **Data Integrity:** Protect job configurations, credentials, and history through regular backups.
* **Proactive Issue Detection:** Detect and resolve issues early (agent disconnections, failed builds, plugin errors).
* **Scalability:** Maintain a healthy Jenkins instance capable of handling many builds and pipelines.

**Real-World Example:**
In large DevOps teams, Jenkins executes hundreds of builds daily. Without monitoring, the disk may fill due to old logs or artifacts, causing Jenkins to stop responding. Maintenance prevents such incidents.

---

### **How it Works / Steps / Syntax**

1. **Monitor Jenkins Health & Performance:**

   * Use built-in views: `Manage Jenkins → System Log` and `Manage Nodes → Load Statistics`.
   * Configure health metrics and alerts (e.g., via Prometheus plugin or CloudWatch integration).

2. **Review & Rotate Logs:**

   * Default log path: `/var/log/jenkins/jenkins.log` (on Linux).
   * Configure rotation or external log management (e.g., ELK stack, CloudWatch logs).

3. **Backup & Restore Critical Data:**

   * Backup `$JENKINS_HOME` directory (includes jobs, credentials, build history, plugins).
   * Automate daily/weekly backups using scripts or backup plugins.

4. **Maintain Build History:**

   * Configure build retention in global settings or in Jenkinsfile:

     ```groovy
     properties([
       buildDiscarder(logRotator(numToKeepStr: '20'))
     ])
     ```
   * Prevent disk bloat by deleting older builds automatically.

5. **Troubleshoot Issues:**

   * Check system logs for errors, slow jobs, plugin issues.
   * Enable temporary detailed logging: `Manage Jenkins → System Log → Add new log recorder`.
   * Review node logs for agent issues.

---

### **Common Issues / Errors**

* Jenkins becomes slow or hangs due to too many builds.
* Disk full because of old artifacts/logs.
* Configuration lost after crash (no backups).
* Agents disconnect frequently (network or credential issues).
* Debugging difficult without logs enabled.

---

### **Troubleshooting / Fixes**

* Monitor `/var/log/jenkins/jenkins.log` for errors.
* Clear workspace and old builds regularly.
* Increase disk or enable log rotation.
* Verify node connectivity using SSH logs.
* Always maintain a tested restore procedure.

---

### **Best Practices / Tips**

* 🟢 Regularly review Jenkins logs and system health.
* 🟢 Keep Jenkins and plugins updated.
* 🟢 Set automatic build retention policies.
* 🟢 Schedule periodic backups of `$JENKINS_HOME`.
* 🟢 Integrate monitoring tools (CloudWatch, Prometheus, Grafana).
* 🟢 Automate alerts for build failures or disk thresholds.

---
---

# Jenkins Logs & Troubleshooting — Detailed Notes

### Concept / What

Jenkins logs provide detailed information about Jenkins operations, including system events, plugin activity, build errors, and agent interactions. The main log file is located at `/var/log/jenkins/jenkins.log`. Console output for builds is separate and stored under each job's build directory.

### Why / Purpose / Real-World Use Case

* **Purpose:** Track Jenkins system behavior, troubleshoot failures, and monitor plugins and agents.
* **Use Cases:**

  1. Build failures with unclear console errors.
  2. Plugin malfunction or version conflicts.
  3. Agent disconnections or network issues.
  4. Disk space or permission issues affecting Jenkins operations.
* Ensures system stability, compliance, and disaster recovery readiness.

### How it Works / Steps / Syntax

* **System Logs:** `/var/log/jenkins/jenkins.log` stores all Jenkins-related system events and plugin activities. Both errors and success events are logged in the same file.
* **Console Output:** `$JENKINS_HOME/jobs/<job>/builds/<num>/log` stores build-specific logs.
* **Searching Logs:**

  * Use Linux commands: `grep`, `sed`, `awk`, `tail -f`.
  * No built-in query language exists for Jenkins logs.
  * External tools like CloudWatch, ELK, or Splunk can help aggregate and search large logs.
* **Differentiating Builds:**

  * System log contains all builds combined.
  * Console output is per-build and can be identified using job name + build number + timestamp.

### Common Issues / Errors

* Too large log files can fill disk space.
* Errors may be buried among successful events.
* Permission issues may prevent log access.
* Searching specific errors can be manual and cumbersome.

### Troubleshooting / Fixes

* Use console output for build-specific errors; system log for Jenkins-level or plugin errors.
* Search logs manually using Linux commands; filter with `grep` for error keywords.
* For large-scale setups, integrate Jenkins logs with CloudWatch, ELK, or Splunk.
* Maintain backups of logs in S3 or other storage for audit and recovery.
* Rotate or delete logs after backup or ingestion to manage disk space.

### Best Practices / Tips

* Console output is easier for build-level error checking.
* System logs capture both errors and success messages; always check context.
* Enable log aggregation for real-time monitoring and historical analysis.
* Backup critical logs to S3 even if using CloudWatch.
* Rotate logs regularly; deletion is only after successful backup/ingestion.
* Understand that differentiating between builds in system logs requires job/build identifiers.

---
---

# Enabling System Logs for Debugging — Detailed Notes

### Concept / What

System logs for debugging in Jenkins allow capturing detailed internal operations, plugin activity, and runtime events. They are more granular than default Jenkins logs and help troubleshoot specific components or performance issues. This is primarily done via **Log Recorders** in Jenkins UI.

### Why / Purpose / Real-World Use Case

* **Purpose:**

  * Troubleshoot issues not apparent in standard system logs or console outputs.
  * Capture plugin errors, agent communication problems, or unusual Jenkins behavior.
* **Real-world use cases:**

  1. Debug intermittent plugin failures.
  2. Investigate agent disconnections or timeouts.
  3. Analyze performance bottlenecks in Jenkins modules.
* Ensures targeted debugging without affecting overall system performance.

### How it Works / Steps / Syntax

#### 1️⃣ Using Log Recorders (UI-based)

1. Navigate to: `Jenkins Dashboard → Manage Jenkins → System Log`
2. Click **Add new log recorder**
3. Provide a **name** (e.g., `Git Plugin Debug`)
4. Click **Add** to include specific component/plugin
5. Select **log level**: `ALL`, `FINE`, `INFO`, `WARNING`, `SEVERE`
6. Save and monitor logs

#### 2️⃣ Logging Guidelines

* Enable only **temporarily** for debugging specific issues
* Do not enable globally — too verbose, affects performance
* Enable for a specific **component/plugin** based on the error observed
* Disable or lower log level after analysis

#### 3️⃣ Advanced Options (Optional)

* Groovy scripts via Script Console or Java system properties can also set logging levels, but UI-based log recorders are standard for declarative pipelines.

### Common Issues / Errors

* Too verbose logs if level `ALL` is used globally → high disk usage
* Wrong component/package name → logs don’t appear
* Some plugins in older versions may not respect custom log levels

### Troubleshooting / Fixes

* Reduce log level if performance or disk usage is impacted
* Verify correct component/plugin names for loggers
* Update plugins to latest versions for proper logging support
* Use external aggregation tools for large log volume management

### Best Practices / Tips

* Enable system logs **only when errors or failures occur**
* Create log recorders for **specific plugins/components**
* Use console output first for build-related errors
* Once debugging is complete, disable log recorders to avoid unnecessary logs
* For large-scale setups, integrate with **CloudWatch, ELK, or Splunk**
* Manual process: observe errors → create recorder → reproduce → analyze → disable

### Interview Tip

* If asked “when do you enable system-level debugging?”:

> Enable detailed logging only when troubleshooting a specific issue. Observe symptoms (build failure, plugin error), create a log recorder for the affected component/plugin, reproduce the issue, analyze logs, and then disable the debug logging.

* If asked “does a log recorder record everything?”:

> A log recorder only records the components/plugins you add. Setting log level to ALL for that logger captures all events for that component, but no global recorder exists for all Jenkins activities.

---
---

# Jenkins Build History & Artifact Retention — Detailed Notes

### Concept / What

Build history and artifact retention in Jenkins refers to how Jenkins stores and manages the artifacts and logs generated by builds across multiple runs of a job. It includes retaining a specific number of builds, controlling which artifacts are stored, and handling long-term storage in external repositories.

### Why / Purpose / Real-World Use Case

* Ensures important build artifacts are available for debugging, testing, or deployment.
* Prevents disk space exhaustion on Jenkins master/agents.
* Allows selective retention of successful builds/artifacts.
* Supports high availability and redundancy when combined with external storage (e.g., Artifactory, S3).
* In multi-branch pipelines, ensures each branch has independent artifact history.

### How it Works / Steps / Syntax

#### 1️⃣ Job vs Build vs Stage

* **Job/Pipeline:** Complete Jenkinsfile including all stages.
* **Build/Run:** Each execution of the pipeline job is a build.
* **Stage:** Individual steps inside a build.

#### 2️⃣ Job-Level Retention (buildDiscarder)

* Configures **how many builds and artifacts to retain**.
* Example:

```groovy
options {
    buildDiscarder(
        logRotator(
            numToKeepStr: '10',          // Keep last 10 builds
            artifactNumToKeepStr: '10'    // Keep artifacts of last 10 builds
        )
    )
}
```

* Applies to all builds collectively, across all branches in a single job (unless multi-branch pipeline).
* Older builds/artifacts beyond limit are automatically deleted.

#### 3️⃣ Stage-Level Artifact Archiving

* Define **which artifacts to store for a specific build/run** using `archiveArtifacts`:

```groovy
stage('Build') {
    steps {
        sh 'make build'
        archiveArtifacts artifacts: 'build/output/**', onlyIfSuccessful: true
    }
}
```

* `onlyIfSuccessful: true` ensures failed builds’ artifacts are not stored.
* Allows selective retention per stage/build.

#### 4️⃣ Multi-Branch Pipelines

* Each branch is treated as a **separate job** under the multi-branch pipeline.
* Each branch maintains its own build history and artifact retention.
* Example: `feature1` last 10 builds retained independently from `feature2`.

#### 5️⃣ External Artifact Storage (Artifactory)

* Artifacts uploaded per branch/feature for long-term storage:

```groovy
stage('Publish Artifacts') {
    steps {
        script {
            def server = Artifactory.server 'ARTIFACTORY_SERVER'
            def uploadSpec = """{
                \"files\": [
                    {\"pattern\": \"build/libs/*.jar\", \"target\": \"my-service/${env.BRANCH_NAME}/\"}
                ]
            }"""
            server.upload(uploadSpec)
        }
    }
}
```

* Provides **branch-specific artifact isolation**.
* Retention is independent of Jenkins’ local retention.

#### 6️⃣ Local vs External Retention

* **Jenkins local artifacts:** Short-term, tied to last N builds; useful for quick access, debugging, high availability.
* **Artifactory / external storage:** Long-term, permanent, source-of-truth storage.
* Combination ensures HA, security, and audit compliance.

### Common Issues / Errors

* Disk full if `buildDiscarder` not configured and many builds run.
* Overwriting artifacts if multi-branch setup not used.
* Failed builds storing unwanted artifacts if `onlyIfSuccessful` not used.

### Troubleshooting / Fixes

* Configure `buildDiscarder` to limit local storage.
* Use `archiveArtifacts` with `onlyIfSuccessful` per stage.
* Implement multi-branch pipelines for per-branch retention.
* Upload all important artifacts to Artifactory for long-term storage.
* Delete local artifacts only after confirming external storage success.

### Best Practices / Tips

* Use multi-branch pipelines for microservices or multiple developers/features.
* Archive only required artifacts per stage to avoid unnecessary storage.
* Retain short-term artifacts locally for HA/debugging; rely on external storage for long-term.
* Combine Jenkins retention (`buildDiscarder`) and external storage policies for optimal disk management.
* Clearly separate artifacts per branch/feature in Artifactory.
* Always verify successful upload to Artifactory before deleting local copies.

---
---

# Jenkins Backup & Restore — Detailed Notes

### Concept / What

Backup & Restore in Jenkins involves creating copies of Jenkins master configurations, job data, plugins, credentials, and logs, ensuring that the system can be restored in case of failures, data loss, or disaster. It includes backing up `JENKINS_HOME` and system-level logs, with options to store them locally or on external storage like S3.

### Why / Purpose / Real-World Use Case

* Ensure disaster recovery and high availability.
* Maintain audit trails of job configurations and builds.
* Enable restoration after accidental deletion, plugin failures, or system corruption.
* Centralized storage in S3 or other offsite storage for reliability.

### How it Works / Steps / Syntax

#### 1️⃣ Backup Jenkins Home (`JENKINS_HOME`)

* **Scope:** Jobs, pipelines, plugin configurations, credentials, system configs, job-specific logs.
* **Recommended Plugin:** ThinBackup (most widely used in production).
* **Steps:**

  1. Install ThinBackup plugin via Jenkins Plugin Manager.
  2. Go to Jenkins Dashboard → Manage Jenkins → ThinBackup.
  3. Configure backup location (directory on master).
  4. Enable incremental backup (only changed files are copied).
  5. Schedule backups (hourly, daily, weekly).
* **Optional:** Combine with a script to **push backups to S3**.

  ```bash
  # Example: sync local ThinBackup folder to S3
  aws s3 sync /path/to/thinbackup s3://my-jenkins-backups/ --storage-class STANDARD_IA
  ```

#### 2️⃣ Backup Jenkins System Logs (`/var/log/jenkins`)

* **Scope:** Master-level logs, including plugin errors, agent connections, and system errors.
* **Important:** Not included in ThinBackup or other Jenkins plugins.
* **Steps:**

  1. Write a script to copy or sync `/var/log/jenkins` to backup storage.
  2. Optionally push to S3 for offsite storage.

  ```bash
  # Backup and sync logs to S3
  cp -r /var/log/jenkins /backup/jenkins-logs-$(date +%F)
  aws s3 sync /backup/jenkins-logs-$(date +%F) s3://my-jenkins-logs/ --storage-class STANDARD_IA
  ```

  3. Implement log rotation using `logrotate` to manage disk space.

#### 3️⃣ Real-World Practice

* **Hybrid Approach:**

  * Plugin handles `JENKINS_HOME` backup (incremental, reliable, scheduled).
  * Script handles system logs and pushes both backups to S3.
* **Incremental Backup:** Reduces storage usage and backup time.
* **Scheduling:** Cron jobs or plugin scheduler ensures backups run automatically.
* **Separation of Concerns:** `JENKINS_HOME` backup (configs/jobs) vs. system logs backup.

### Common Issues / Errors

* Partial backup if Jenkins is running and files are locked.
* Disk full errors if logs are not rotated.
* Permission denied errors if Jenkins user cannot access `/var/log/jenkins` or backup directory.
* S3 sync failures due to network or IAM misconfigurations.

### Troubleshooting / Fixes

* Stop Jenkins temporarily for **full backup**, or use incremental backups while running.
* Use logrotate to prevent disk space issues.
* Ensure proper IAM roles/credentials for S3 access.
* Test restore periodically to verify backup integrity.

### Best Practices / Tips

* Use **ThinBackup plugin** for Jenkins master (`JENKINS_HOME`) backup.
* Use **scripts for system logs backup** and S3 offsite storage.
* Implement **incremental backups** for efficiency.
* Schedule backups (cron or plugin) to run automatically.
* Rotate logs regularly to manage disk usage.
* Keep plugin backups and system logs **separately**.
* Test restores to ensure backups are reliable.
* Secure backups with proper access control.
* Offsite storage (S3) ensures disaster recovery and HA.

---

This note includes both **Jenkins master backups** and **system log backups**, combining plugins and scripts for a **real-world, interview-ready approach**.

---
---

# Jenkins Monitoring — Detailed Notes

### Concept / What

Monitoring Jenkins health ensures Jenkins master, agents, builds, and pipelines are running optimally. It includes monitoring Jenkins metrics, JVM performance, executor usage, build queues, system resources, and optionally custom metrics for pipelines. Tools like **Prometheus**, **Jenkins Monitoring plugin**, **Metrics plugin**, **Grafana**, and **CloudWatch** are commonly used for centralized monitoring, alerting, and trend analysis.

### Why / Purpose / Real-World Use Case

* Detect and prevent **build failures** due to system or Jenkins-related bottlenecks.
* Monitor **JVM and executor health**, queue lengths, and build durations.
* Ensure **infrastructure health** (CPU, memory, disk) for EC2 instances running Jenkins.
* Enable **custom monitoring** for specific pipelines, artifacts, or business logic.
* Centralized dashboards in Grafana allow **alerts, historical trends, and proactive scaling**.

### How it Works / Steps / Syntax

#### 1️⃣ Jenkins Metrics via Prometheus Plugin

* **Metrics exposed:**

  * **JVM:** heap memory usage, GC time/count, thread count
  * **Executors:** total, busy, idle executors; build queue length
  * **Build metrics:** build duration, success/failure count per job
  * **Workspace metrics:** artifact sizes, disk usage
  * **Custom metrics:** pipeline-specific durations, stage error counts, artifact counts
* **Setup Steps:**

  1. Install Prometheus plugin on Jenkins.
  2. Prometheus server scrapes metrics endpoint: `http://<jenkins>:8080/prometheus`.
  3. Configure Grafana dashboards for visualization.

#### 2️⃣ Jenkins Monitoring Plugin

* Provides detailed JVM metrics, executor and queue monitoring, and plugin-level statistics.
* Useful for **matrix monitoring** including thread count, memory usage, build duration per job, and executor metrics.
* Setup via Jenkins UI or declarative configuration.

#### 3️⃣ Metrics Plugin

* Exposes Jenkins and system-level metrics in a consumable format.
* Can be integrated with external monitoring systems (Grafana, Prometheus, CloudWatch).
* Supports **custom metrics** through scripts or pipeline definitions.

#### 4️⃣ Host / EC2 Metrics (Optional but recommended)

* Metrics: CPU %, total memory %, disk usage, network I/O
* Tools: **NodeExporter** (for Prometheus) or **CloudWatch** (AWS)
* Correlate with Jenkins metrics to detect **infrastructure-induced bottlenecks**.

#### 5️⃣ Grafana Visualization

* Combine Prometheus, Jenkins Monitoring plugin, Metrics plugin, and EC2 metrics.
* Dashboards for:

  * Build queue length
  * Build duration trends
  * Executor utilization
  * JVM memory & GC metrics
  * CPU/memory/disk usage of EC2
  * Custom pipeline metrics

### Common Issues / Errors

* High build queue due to overloaded executors.
* Out-of-memory errors in Jenkins JVM.
* Long GC pauses affecting pipeline performance.
* Disk full or memory saturation on EC2 instance.
* Pipeline-specific failures undetected without custom metrics.

### Troubleshooting / Fixes

* Tune JVM heap and GC settings.
* Increase executors or scale Jenkins agents.
* Monitor host-level CPU/memory/disk; scale EC2 or add nodes.
* Enable custom metrics for pipeline-specific monitoring.
* Set alerts on key metrics via Grafana or CloudWatch.

### Best Practices / Tips

* Use **Prometheus plugin** for Jenkins process-level metrics.
* Use **NodeExporter/CloudWatch** for OS-level metrics.
* Combine **Jenkins Monitoring plugin** and **Metrics plugin** for full visibility.
* Always track **custom metrics** for critical pipelines or business logic.
* Combine all metrics in **Grafana dashboards** for holistic monitoring.
* Set threshold-based alerts to prevent failures proactively.
* Correlate Jenkins and EC2 metrics for accurate root cause analysis.
* Historical monitoring allows **trend analysis** and proactive scaling decisions.

---
---
---

