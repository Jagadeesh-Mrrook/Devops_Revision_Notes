## Jenkins Monitoring & Maintenance — Quick Notes

### **Concept / What**

Ongoing process of tracking Jenkins’ performance and performing cleanups, backups, and troubleshooting.

### **Why / Purpose**

* Keep Jenkins stable and fast.
* Prevent disk space and build queue issues.
* Protect configurations and credentials.

### **How / Steps**

* Monitor health (System Log, Node stats).
* Rotate logs (`/var/log/jenkins/jenkins.log`).
* Backup `$JENKINS_HOME`.
* Configure build retention.
* Troubleshoot via log recorders.

### **Common Issues**

* Slow UI, full disk, agent disconnects, config loss.

### **Troubleshooting / Fixes**

* Check logs, clear workspace, verify node health, restore from backup.

### **Best Practices**

* Enable monitoring tools (CloudWatch, Prometheus).
* Automate backups.
* Set log rotation.
* Delete old builds automatically.
* Keep Jenkins core and plugins updated.

---
---

# Jenkins Logs & Troubleshooting — Quick Revision

### Concept / What

* Main system log: `/var/log/jenkins/jenkins.log` → captures system/plugin/agent events (errors + success)
* Build console output: `$JENKINS_HOME/jobs/<job>/builds/<num>/log`

### Why / Purpose

* Track Jenkins system behavior
* Troubleshoot plugin/agent/build issues
* Ensure system stability, compliance, and recovery

### How / Steps

* Search logs using Linux commands: `grep`, `sed`, `awk`, `tail -f`
* No built-in query language
* External tools (CloudWatch, ELK, Splunk) for aggregation
* Differentiate builds using job name + build number + timestamp

### Common Issues

* Large log files, disk space filling
* Errors buried among success messages
* Permission issues
* Manual search can be cumbersome

### Troubleshooting / Fixes

* Check console output for build errors
* Use system log for Jenkins/plugin/agent errors
* Integrate with CloudWatch/ELK for easier searching
* Backup logs to S3, rotate/delete after ingestion

### Best Practices / Tips

* Console output for quick build error check
* System logs for deeper Jenkins/plugin errors
* Log aggregation for monitoring and historical analysis
* Backup critical logs; rotate regularly
* Use job/build identifiers to differentiate builds

---
---

# Enabling System Logs for Debugging — Quick Revision

### Concept / What

* Log Recorders capture detailed internal operations and plugin/component events.
* More granular than standard system logs; used for targeted debugging.

### Why / Purpose

* Troubleshoot plugin errors, agent issues, or unusual Jenkins behavior.
* Avoid performance impact by limiting to specific components/plugins.

### How / Steps

* Navigate: `Jenkins Dashboard → Manage Jenkins → System Log → Add New Log Recorder`
* Name recorder (e.g., `Git Plugin Debug`)
* Add component/plugin logger
* Set log level (`ALL`, `FINE`, etc.)
* Enable only temporarily; disable after analysis

### Guidelines / Best Practices

* Enable only when errors or failures occur
* Use console output first for build-related errors
* Do not enable globally; too verbose
* Integrate with CloudWatch/ELK/Splunk for large setups
* Manual process: observe error → create recorder → reproduce → analyze → disable

### Troubleshooting / Common Issues

* Wrong component names → logs not captured
* Too verbose → high disk usage
* Older plugins may not respect custom log levels

### Interview Tip

* Explain enabling only when debugging a specific issue
* Log recorder records only the components/plugins added; no global recorder exists

---
---

# Jenkins Build History & Artifact Retention — Quick Revision Notes

### Concept / What

Manage and retain build artifacts and logs for each Jenkins job and build.

### Key Points

* **Job/Pipeline:** Complete Jenkinsfile including all stages.
* **Build/Run:** Each execution of the job.
* **Stage:** Step inside a build.

### Retention

* **Job-level (`buildDiscarder`):** Retains last N builds and their artifacts. Example: last 10 builds/artifacts.
* **Stage-level (`archiveArtifacts`):** Store specific artifacts per build/stage, optionally only if successful.
* **Multi-branch pipelines:** Each branch has its own build/artifact retention.

### External Storage

* **Artifactory/S3:** Store artifacts per branch/feature; independent of Jenkins retention.
* Local artifacts kept for **quick access, debugging, HA**.

### Best Practices

* Archive only required artifacts.
* Use multi-branch pipelines for multiple developers/features.
* Combine Jenkins retention and external storage policies.
* Ensure successful external upload before deleting local copies.

### Key Takeaways

* Artifacts retention is **proportional to build retention** locally.
* External storage is the **source of truth**.
* Multi-branch pipelines isolate artifacts per branch/feature.
* Local retention supports HA, debugging, and convenience.

---
---

# Jenkins Backup & Restore — Quick Revision Notes

### Jenkins Home Backup (JENKINS_HOME)

* **Scope:** Jobs, pipelines, plugins, credentials, job-specific logs.
* **Plugin:** ThinBackup (incremental backups).
* **Setup:**

  * Configure backup directory on master.
  * Schedule backups (hourly/daily/weekly).
  * Incremental backup enabled to save storage.
* **S3 Storage:** Use script to push backups: `aws s3 sync /path/to/thinbackup s3://my-jenkins-backups/`

### System Logs Backup (/var/log/jenkins)

* **Scope:** Master-level logs (system, plugin, agent errors).
* **Method:** Script-based backup, not handled by plugins.
* **Steps:**

  * Copy logs to backup folder.
  * Push to S3 for offsite storage.
  * Use `logrotate` to manage disk space.

### Real-World Approach

* **Hybrid method:** Plugin for `JENKINS_HOME`, scripts for system logs + S3.
* **Incremental backup:** Efficient, reduces storage and time.
* **Separation:** Keep `JENKINS_HOME` backups and system logs backups separate.

### Best Practices

* Test restore periodically.
* Schedule automated backups.
* Implement log rotation for system logs.
* Secure backups and ensure offsite storage (S3) for disaster recovery.
* Use ThinBackup for configs/jobs; scripts for system logs.

---
---

# Jenkins Monitoring — Quick Revision

### Concept / What

Monitoring Jenkins health to ensure master, agents, builds, and pipelines are running optimally using metrics, dashboards, and alerts.

### Key Metrics to Monitor

| Category            | Metrics                                        | Tool/Source                                  |
| ------------------- | ---------------------------------------------- | -------------------------------------------- |
| **JVM**             | Heap memory usage, GC time/count, thread count | Prometheus Plugin                            |
| **Executors**       | Busy/Idle executors, build queue length        | Prometheus Plugin, Jenkins Monitoring Plugin |
| **Build**           | Build duration, success/failure count          | Prometheus Plugin, Metrics Plugin            |
| **Workspace**       | Artifact sizes, disk usage                     | Prometheus Plugin, Metrics Plugin            |
| **Custom/Pipeline** | Stage durations, error counts, artifact counts | Prometheus Plugin, Metrics Plugin            |
| **System/Host**     | CPU %, memory %, disk usage, network I/O       | NodeExporter, CloudWatch                     |

### How It Works

* Install **Prometheus Plugin** on Jenkins for Jenkins internal metrics.
* Optional **NodeExporter** or **CloudWatch** for EC2/system metrics.
* **Grafana dashboards** visualize all metrics together.
* Custom metrics can be defined for pipelines or stages.

### Common Issues / Errors

* High build queue due to overloaded executors.
* Out-of-memory errors in Jenkins JVM.
* Disk full or memory saturation on EC2 instance.
* Pipeline-specific failures undetected without custom metrics.

### Best Practices

* Monitor **all relevant metrics**: JVM, executors, build, workspace, custom, system.
* Combine Prometheus, Jenkins Monitoring, Metrics Plugin, and system metrics in Grafana.
* Set **threshold-based alerts** for proactive monitoring.
* Track **historical trends** for scaling and performance optimization.
* Always include **custom metrics** for critical pipelines or business logic.

---
---
---


