# CloudWatch Container Insights – Detailed Explanation

## **1. Concept / What**

CloudWatch Container Insights is a feature inside Amazon CloudWatch that provides pre-built dashboards and analytics for monitoring the performance of Kubernetes workloads running on EKS. It does not collect data by itself; instead, it visualizes the metrics and logs that the CloudWatch agent collects from EKS nodes and sends to CloudWatch Metrics and CloudWatch Logs.

---

## **2. Why / Purpose / Real-World Use Case**

* Provides out-of-the-box monitoring for EKS clusters without requiring Prometheus setup.
* Helps identify issues like high CPU/memory usage, pod restarts, node pressure, and network bottlenecks.
* Offers ready-made dashboards for quick operational visibility.
* Useful for troubleshooting deployments, scaling issues, and performance bottlenecks.
* Supports both infrastructure monitoring (nodes) and workload monitoring (pods/containers).

---

## **3. How It Works / Steps / Syntax**

### **How Container Insights Works**

1. CloudWatch agent is deployed on every worker node using a DaemonSet.
2. The agent scrapes node and pod-level metrics, and optionally collects logs.
3. Metrics are sent to **CloudWatch Metrics** under a dedicated namespace.
4. Logs are sent to **CloudWatch Logs**.
5. Container Insights reads this data and displays dashboards, charts, and analytics.

### **Steps to Set Up Container Insights**

* Create/attach IAM role with the `CloudWatchAgentServerPolicy` to worker nodes.
* Deploy the CloudWatch agent as a DaemonSet in the cluster.
* Apply the ConfigMap containing CloudWatch agent configuration.
* Verify CloudWatch Metrics and Logs are receiving data.
* View dashboards under **CloudWatch → Container Insights**.

### **Example DaemonSet (Simplified)**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cloudwatch-agent
  namespace: amazon-cloudwatch
spec:
  selector:
    matchLabels:
      name: cloudwatch-agent
  template:
    metadata:
      labels:
        name: cloudwatch-agent
    spec:
      serviceAccountName: cloudwatch-agent
      containers:
      - name: cloudwatch-agent
        image: amazon/cloudwatch-agent:latest
        volumeMounts:
        - name: cwagentconfig
          mountPath: /etc/cwagentconfig
      volumes:
      - name: cwagentconfig
        configMap:
          name: cwagentconfig
```

### **Example ConfigMap**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cwagentconfig
  namespace: amazon-cloudwatch
data:
  cwagentconfig.json: |
    {
      "agent": {
        "region": "ap-south-1"
      },
      "metrics": {
        "metrics_collected": {
          "cpu": {},
          "mem": {},
          "disk": {},
          "net": {}
        }
      }
    }
```

---

## **4. Common Issues / Errors**

### **1. Missing Metrics in Dashboard**

* Node IAM role missing CloudWatch permissions.
* DaemonSet not running on all nodes.

### **2. CloudWatch Agent CrashLoopBackOff**

* ConfigMap contains invalid JSON.
* Node memory/CPU not enough for the agent.

### **3. High CloudWatch Costs**

* Too many logs are being pushed.
* Metric collection interval set too low (e.g., 1s or 10s).

---

## **5. Troubleshooting / Fixes**

* Validate the CloudWatch agent ConfigMap.
* Check DaemonSet events and pod logs.
* Ensure IAM role includes `CloudWatchAgentServerPolicy`.
* Disable unnecessary log collection to reduce costs.
* Use longer metric intervals (60s or 300s).

---

## **6. Best Practices / Tips**

* Deploy CloudWatch agent as a DaemonSet so every node is monitored.
* Enable only essential logs to avoid unnecessary CloudWatch charges.
* Combine Container Insights with CloudWatch Alarms for node and pod health.
* Regularly clean old log groups to manage cost.
* Use Container Insights for quick visibility and Prometheus/Grafana for deeper analytics.

---
---

# CloudWatch Metrics, Logs & Alarms – Detailed Explanation

## **1. Concept / What**

**CloudWatch Metrics** are time‑series data points stored in CloudWatch (CPU, memory, disk, network, etc.). For EKS, these metrics are pushed by the CloudWatch agent running as a DaemonSet.

**CloudWatch Logs** store application logs, container logs, kubelet/system logs, and other diagnostic information. Logs are organized into **Log Groups** (categories/folders) and **Log Streams** (individual log files for pods/nodes).

**CloudWatch Alarms** continuously monitor specific metrics and trigger actions (SNS, email, automation) when thresholds are crossed. They help detect issues and alert teams in real time.

---

## **2. Why / Purpose / Real-World Use Case**

* Monitor EKS nodes and pods (CPU, memory, disk, network).
* Troubleshoot pod failures, CrashLoopBackOff, or node pressure.
* Track container/application logs in one centralized location.
* Set alerts for high CPU, memory pressure, pod restarts, or node failures.
* Automate scaling or remediation using alarms.
* Essential for SRE/DevOps observability and production stability.

---

## **3. How It Works / Steps / Syntax**

### **Metrics Flow**

1. CloudWatch agent scrapes EKS node and pod performance metrics.
2. Sends them to CloudWatch Metrics under namespaces like:

   * `CWAgent`
   * `ContainerInsights/`
3. Metrics appear on graphs and dashboards.

### **Logs Flow**

```
EKS Pod/Node → CloudWatch Agent → CloudWatch Logs → Log Groups → Log Streams
```

* **Log Group** = folder/category (e.g., `/aws/containerinsights/cluster/application`).
* **Log Stream** = file inside the log group (e.g., logs from one specific pod or node).
* CloudWatch agent automatically creates these groups/streams unless manually defined.

### **Alarms Flow**

1. Choose a metric (e.g., node CPU, pod restarts).
2. Set a threshold (e.g., CPU > 80% for 5 minutes).
3. Attach an action:

   * SNS (email/SMS)
   * Auto Scaling
   * Lambda function
4. When triggered, CloudWatch sets state: `OK` → `ALARM` → `OK`.

### **Example CloudWatch Alarm (JSON representation)**

```json
{
  "AlarmName": "HighCPU-EKS-Node",
  "MetricName": "node_cpu_utilization",
  "Namespace": "ContainerInsights",
  "Statistic": "Average",
  "Period": 300,
  "EvaluationPeriods": 1,
  "Threshold": 80,
  "ComparisonOperator": "GreaterThanThreshold",
  "AlarmActions": ["arn:aws:sns:ap-south-1:123456789012:ClusterAlerts"]
}
```

---

## **4. Common Issues / Errors**

### **Metrics Issues**

* Missing metrics due to IAM role not having `CloudWatchAgentServerPolicy`.
* CloudWatch agent not running on nodes.

### **Log Issues**

* Wrong log group name in CloudWatch agent ConfigMap.
* No pod stdout logs being sent.
* Log volumes not mounted correctly.

### **Alarm Issues**

* Alarms not triggering due to missing metric data.
* Incorrect threshold or evaluation period.
* SNS subscription in "Pending confirmation" state.

---

## **5. Troubleshooting / Fixes**

* Validate CloudWatch agent ConfigMap (JSON correctness).
* Check DaemonSet pod logs using `kubectl logs`.
* Verify CloudWatch Metrics namespace (`CWAgent` or `ContainerInsights`).
* Ensure correct log group names and retention policies.
* Confirm SNS subscription is verified.
* Increase evaluation periods for stable alerting.

---

## **6. Best Practices / Tips**

* Always set **log retention** (default is indefinite → high cost).
* Use **SNS topics** for sending alarm notifications.
* Use **60s metric intervals** for balanced cost and visibility.
* Organize log groups with meaningful names when created manually.
* Use tags for cost tracking and log organization.
* Create alarms for:

  * Node CPU and memory
  * Disk usage
  * Pod restarts
  * Kubelet or CNI errors (optional)
* Rely on Container Insights dashboards for quick visibility, and use custom dashboards only when required.

---
---
# Prometheus – Detailed Explanation (Including All Components)

## **1. Concept / What**

Prometheus is an open‑source metrics monitoring system used heavily with Kubernetes. It scrapes metrics from Kubernetes components, nodes, pods, and external systems, stores them in a time‑series database, and exposes them for querying through PromQL. Prometheus is typically paired with Grafana for dashboards and Alertmanager for notifications. Prometheus does **not** collect logs – only metrics.

---

## **2. Why / Purpose / Real‑World Use Case**

* Monitor Kubernetes pods, nodes, deployments, and cluster health.
* Power Kubernetes autoscaling (HPA) using CPU, memory, and custom metrics.
* Debug performance issues such as latency, throttling, and resource pressure.
* Monitor non‑Kubernetes systems (EC2, Jenkins, SonarQube) using exporters.
* Alert SRE/DevOps teams when metrics cross thresholds.
* Create detailed dashboards using Grafana.

---

## **3. How It Works / Steps / Syntax**

### **Prometheus Architecture (Simple Flow)**

```
Prometheus Server → scrapes → Targets (exporters, kubelets, pods)
                   ↓
           Stores time-series data (TSDB)
                   ↓
          Queried by Grafana / Alerts via Alertmanager
```

### **Scrape Process**

Prometheus discovers and scrapes:

* Kubernetes nodes via kubelet/cAdvisor
* Kubernetes object metrics via kube-state-metrics
* Containers via cAdvisor
* Applications via /metrics endpoints
* External EC2 servers via Node Exporter

### **Installation (Helm – kube-prometheus-stack)**

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

This installs the entire Prometheus monitoring ecosystem.

### **Scrape Config Example (external EC2)**

```yaml
scrape_configs:
  - job_name: 'external-ec2'
    static_configs:
    - targets:
      - '10.0.1.25:9100'
      - '10.0.1.30:9100'
```

---

## **4. Prometheus Components (Installed via kube-prometheus-stack)**

### **1. Prometheus Server**

* Core component.
* Scrapes metrics and stores them in internal TSDB.
* Provides UI for Querying (PromQL).

### **2. Alertmanager**

* Sends alerts through SNS, email, Slack, PagerDuty, OpsGenie.
* Works with Prometheus alert rules.

### **3. Grafana**

* Visualization layer.
* Uses Prometheus as data source.
* Pre-built Kubernetes dashboards.

### **4. Node Exporter** (DaemonSet)

* Runs on every Kubernetes node.
* Exposes OS-level metrics:

  * CPU, memory, disk, network
  * Load average
  * Filesystem usage

### **5. kube-state-metrics**

* Exposes Kubernetes object metrics:

  * Deployment replicas
  * Pod status
  * DaemonSets, StatefulSets
  * HPA metrics
  * Job successes/failures

### **6. cAdvisor (via kubelet)**

* Exposes **container-level** metrics:

  * Per-container CPU & memory usage
  * Throttling
  * OOM kills
  * Container lifecycle info

### **7. Prometheus Operator**

* Makes Prometheus Kubernetes-native.
* Manages Prometheus configuration via CRDs.
* Handles auto-discovery for pods/services/endpoints.
* Simplifies creation of Alert rules, ServiceMonitors, PodMonitors.

---

## **5. Common Issues / Errors**

### **1. Prometheus high memory usage**

* Caused by scraping too many metrics or long retention.
* Fix: reduce retention, disable unused scrape jobs.

### **2. “Target Down” problems**

* Endpoint unreachable or port blocked.
* Fix: check Service, endpoints, and network policy.

### **3. Missing container metrics**

* cAdvisor disabled or kubelet misconfigured.
* Fix: enable cAdvisor metrics.

### **4. Prometheus pod crash due to disk full**

* TSDB retention too long.
* Fix: set `--storage.tsdb.retention.time=7d` (or shorter).

---

## **6. Troubleshooting / Fixes**

* Check Prometheus UI targets page for scrape status.
* Use PromQL queries to verify metrics.
* Validate Prometheus CRDs: ServiceMonitor, PodMonitor.
* Check Prometheus logs using `kubectl logs`.
* Ensure network policies/Security Groups allow scraping of external EC2 exporters.

---

## **7. Best Practices / Tips**

* Run Prometheus & Grafana on a **separate monitoring node group** (tainted).
* Limit Prometheus CPU/memory resources.
* Shorten retention (7–15 days).
* Disable noisy/unused metrics.
* Store long-term metrics in Thanos/AMP if needed.
* Use Grafana for dashboards instead of manual Prometheus graphs.
* Monitor both Kubernetes and external EC2 machines using exporters.

---
---

# Prometheus Exporters – Detailed Explanation

## **1. Concept / What**

Prometheus Exporters are small agents/processes that expose metrics in Prometheus format (`/metrics` endpoint). Prometheus scrapes these endpoints to collect system, application, or Kubernetes-related metrics. Exporters convert raw system/application data into a standardized metrics format that Prometheus understands. Exporters do **not** store metrics — they only expose them.

---

## **2. Why / Purpose / Real-World Use Case**

* Prometheus alone cannot directly read system or Kubernetes object data.
* Exporters enable Prometheus to monitor:

  * Kubernetes objects (deployments, pods, HPAs)
  * Container metrics
  * Node/VM metrics
  * Applications like Jenkins, SonarQube, MySQL, Nginx, Redis
  * External EC2 instances and on-prem servers
* Enable alerting and dashboards for both Kubernetes and EC2/on-prem workloads.

---

## **3. How It Works / Steps / Syntax**

### **Flow Overview**

```
Exporter → exposes /metrics → Prometheus scrapes it → Stores in TSDB → Grafana visualizes
```

### **Scrape Example**

Prometheus configuration referencing an exporter endpoint:

```yaml
scrape_configs:
  - job_name: 'external-ec2'
    static_configs:
    - targets:
      - '10.0.1.25:9100'
      - '10.0.1.30:9100'
```

Prometheus scrapes these EC2 instances' Node Exporter metrics.

---

## **4. Main Exporters Used in Kubernetes Environments**

### **1. Node Exporter (OS-Level Metrics)**

* Runs as DaemonSet on every Kubernetes node.
* Also installed on standalone EC2 servers.
* Provides:

  * CPU usage
  * Memory usage
  * Disk usage
  * Network statistics
  * Load averages
  * Filesystem metrics

### **2. kube-state-metrics (Kubernetes Object Metrics)**

* Does **not** provide CPU or memory metrics.
* Provides **desired vs actual state** of Kubernetes objects.
* Runs as Deployment inside Kubernetes.
* Metrics include:

  * Deployment replicas (desired, available, unavailable)
  * Pod phase (Running, Pending, Failed)
  * Pod restart counts
  * Node conditions (Ready/NotReady, MemoryPressure, DiskPressure)
  * DaemonSet rollout status
  * StatefulSet replicas
  * HPA desired vs current replicas
  * Job/CronJob success and failure counts
  * PVC/PV statuses

### **3. cAdvisor (Container Metrics via Kubelet)**

* Integrated into Kubelet; not installed separately.
* Provides container-level metrics:

  * Per-container CPU and memory usage
  * CPU throttling
  * OOM kills
  * Container lifecycle

### **4. Blackbox Exporter**

* Provides network probing:

  * HTTP checks
  * DNS checks
  * TCP checks
  * Ping/ICMP checks
* Used to monitor service uptime/availability.

### **5. Application-Specific Exporters**

Examples:

* Jenkins Prometheus plugin
* SonarQube Prometheus exporter
* MySQL exporter
* Redis exporter
* Kafka exporter
* Nginx exporter
* RabbitMQ exporter

Used for monitoring external EC2 services and application workloads.

---

## **5. Common Issues / Errors**

### **1. Exporter Not Reachable**

* Wrong endpoint or port.
* Network policy or security group blocking access.
* Fix: verify service, endpoint, and network rules.

### **2. Duplicate Metrics**

* Multiple exporters exposing overlapping metrics.
* Fix: disable or filter conflicting metrics.

### **3. High Cardinality Metrics**

* Exporters generating too many labels (e.g., per-pod/per-container dynamically).
* Fix: drop or filter unnecessary metrics; use relabeling.

---

## **6. Troubleshooting / Fixes**

* Check Prometheus UI: `http://<prometheus>:9090/targets` for exporter status.
* Test exporter endpoint: `curl http://<target>:9100/metrics`.
* Check exporter logs using `kubectl logs <exporter-pod>`.
* Verify scrape configurations (ServiceMonitor/PodMonitor).
* Ensure network policies/Security Groups allow access.

---

## **7. Best Practices / Tips**

* Use exporters only for needed metrics (avoid metric bloat).
* Rely on ServiceMonitor/PodMonitor for Kubernetes-native discovery.
* Run Node Exporter as DaemonSet on all EKS nodes.
* For EC2, install Node Exporter + application exporters.
* Avoid exposing exporter endpoints publicly.
* Use Grafana dashboards for visibility.
* Manage high-cardinality metrics carefully to prevent Prometheus overload.

---
---

# Prometheus Alerts & Alertmanager – Detailed Explanation

## **1. Concept / What**

**Prometheus Alerts** are metric-based alerting rules written using PromQL. Prometheus continuously evaluates these rules and fires alerts when conditions become true (e.g., CPU > 80%).

**Alertmanager** is the component that receives alerts from Prometheus and handles:

* Notification delivery (Slack, Email, PagerDuty, OpsGenie, Webhooks)
* Grouping related alerts
* Deduplication
* Silencing and inhibition

Prometheus detects the issue. Alertmanager sends the notifications.

---

## **2. Why / Purpose / Real-World Use Case**

* Detect cluster and application issues early.
* Receive alerts for problems like:

  * High CPU/memory usage
  * Pod CrashLoopBackOff
  * Node NotReady
  * Disk pressure
  * HPA scaling failures
  * Deployment rollout failures
* Automate incident response via:

  * Lambda triggers
  * PagerDuty notifications
  * Slack or email alerts
* Avoid alert duplication by grouping and inhibiting alerts.

---

## **3. How It Works / Steps / Syntax**

### **Alert Flow**

```
Prometheus → Alert Rule becomes true → Fires alert
        ↓
Alertmanager → Groups / deduplicates
        ↓
Sends notification → Slack / Email / PagerDuty / Webhook
```

### **Prometheus Alert Rule Example**

```yaml
groups:
- name: node-alerts
  rules:
  - alert: HighNodeCPU
    expr: node_cpu_seconds_total{mode!="idle"} / ignoring(mode) group_left node_cpu_seconds_total > 0.80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage on node"
      description: "CPU has been above 80% for more than 5 minutes."
```

* `expr:` PromQL condition
* `for:` how long the condition must remain true
* `labels:` severity/metadata
* `annotations:` notification message

### **Alertmanager Notification Configuration (Slack Example)**

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: slack-notifications

receivers:
- name: slack-notifications
  slack_configs:
  - channel: "#eks-alerts"
    api_url: "https://hooks.slack.com/services/TOKEN"
```

You can configure email, Slack, PagerDuty, OpsGenie, webhooks, etc.

---

## **4. Real-World Scenarios / Use Cases**

### **1. Pod restart storms**

Alert: "Pod xyz restarted more than 5 times in 10 minutes."

### **2. Deployment unavailable replicas**

Alert: "Deployment nginx has 2/5 replicas available."

### **3. Node down**

Alert: "Node ip-10-0-1-20 NotReady for more than 2 minutes."

### **4. Disk pressure**

Alert about node disk nearing full capacity.

### **5. High API server latency**

Alert when Kubernetes API becomes slow.

---

## **5. Common Issues / Errors**

### **1. Alerts not firing**

* Wrong PromQL query
* Metric not scraped
* Invalid rule syntax

### **2. Alertmanager not sending notifications**

* Wrong endpoint URL
* Receiver misconfiguration
* Failing webhook

### **3. Too many alerts (alert storm)**

* No grouping
* No `for:` condition

### **4. Duplicate alerts**

* Multiple Prometheus servers scraping same targets

---

## **6. Troubleshooting / Fixes**

* Check alert rules: `http://<prometheus>:9090/rules`
* Check firing alerts: `http://<prometheus>:9090/alerts`
* Check Alertmanager: `http://<alertmanager>:9093`
* Validate PromQL expressions in the Prometheus UI
* Check Alertmanager receivers under the Status tab
* Test Slack/email/webhook endpoints manually

---

## **7. Best Practices / Tips**

### **Prometheus Side**

* Always use `for:` in alerts to avoid false positives.
* Use severity labels (warning, critical).
* Use recording rules for heavy PromQL queries.
* Avoid alerting on instantly fluctuating metrics.

### **Alertmanager Side**

* Use grouping to combine similar alerts.
* Use inhibition rules (e.g., suppress pod alerts when node is down).
* Route alerts to correct team channels.
* Silence alerts during maintenance.

### **General**

* Keep messages clear and actionable.
* Test alerts in staging first.
* Store alerts in Git (IaC).
* Standardize alert naming.

---
---

# Log Routing to CloudWatch Logs – Detailed Explanation

## **1. Concept / What**

Log routing to CloudWatch Logs is the process of collecting logs from EKS pods, containers, Kubernetes system components, and worker nodes, and sending them to AWS CloudWatch Logs. This enables centralized log storage, search, analytics, retention, and alerting. Logs are primarily collected using the **CloudWatch Agent DaemonSet** or (optionally) **Fluent Bit**, but CloudWatch Agent is sufficient for core EKS logging.

---

## **2. Why / Purpose / Real-World Use Case**

* Centralized log management for all Kubernetes workloads.
* Easier debugging of pod crashes, node issues, application errors.
* Compliance, audit, and long-term retention.
* Query and analyze logs using CloudWatch Logs Insights.
* Integrate logs with CloudWatch metrics and dashboards.
* Forward logs to other systems like OpenSearch, SIEM tools, or Lambda.
* Helps DevOps/SRE teams quickly troubleshoot EKS issues.

---

## **3. How It Works / Steps / Syntax**

### **Logging Flow**

```
EKS Pods / Containers / Kubelet / System Logs
                   ↓
          CloudWatch Agent (DaemonSet)
                   ↓
            CloudWatch Logs Service
                   ↓
         Log Groups and Log Streams
```

### **Key Components**

* **CloudWatch Agent DaemonSet**: Runs on every worker node, collects logs and pushes to CloudWatch.
* **IAM Role for Nodes**: Must include `CloudWatchAgentServerPolicy` to allow pushing logs.
* **ConfigMap (cwagentconfig.json)**: Defines which logs are collected and to which log groups they are sent.
* **Log Groups & Streams**: Automatically created in CloudWatch for storing logs.

### **ConfigMap Example (CloudWatch Agent)**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cwagentconfig
  namespace: amazon-cloudwatch
data:
  cwagentconfig.json: |
    {
      "logs": {
        "metrics_collected": {},
        "force_flush_interval": 5,
        "logs_collected": {
          "files": {
            "collect_list": [
              {
                "file_path": "/var/log/containers/*.log",
                "log_group_name": "/aws/eks/cluster1/application",
                "log_stream_name": "{instance_id}"
              },
              {
                "file_path": "/var/log/kubelet.log",
                "log_group_name": "/aws/eks/cluster1/kubelet",
                "log_stream_name": "{instance_id}"
              }
            ]
          }
        }
      }
    }
```

* Collects container logs.
* Collects kubelet/system logs.
* Sends logs to CloudWatch Log Groups.

---

## **4. Logs That Get Collected**

### **1. Application Logs**

* Microservices logs (stdout/stderr)
* Pod logs from `/var/log/containers/*.log`

### **2. Kubernetes System Logs**

* Kubelet logs
* CNI plugin logs
* CoreDNS logs
* Container runtime logs

### **3. Node System Logs**

* `/var/log/messages`
* `dmesg`

These logs are all sent to their respective CloudWatch Log Groups.

---

## **5. Common Issues / Errors**

### **1. Logs not appearing in CloudWatch**

* Missing IAM permissions
* Incorrect file paths in ConfigMap
* CloudWatch Agent DaemonSet not running

### **2. Wrong or duplicate Log Groups**

* Misconfigured log group names in the agent config

### **3. High CloudWatch costs**

* Large volumes of noisy logs
* No retention policies

### **4. Duplicate logs**

* Using both CloudWatch Agent and Fluent Bit at the same time unintentionally

---

## **6. Troubleshooting / Fixes**

* Check CloudWatch Agent pod logs:

  ```sh
  kubectl logs ds/cloudwatch-agent -n amazon-cloudwatch
  ```
* Verify DaemonSet is running on all nodes:

  ```sh
  kubectl get pods -n amazon-cloudwatch -o wide
  ```
* Confirm container logs exist under `/var/log/containers/`.
* Validate IAM role includes `CloudWatchAgentServerPolicy`.
* Use CloudWatch Logs Insights queries to verify log ingestion.

---

## **7. Best Practices / Tips**

### **Architecture**

* Run CloudWatch Agent as a DaemonSet for guaranteed coverage.
* Use separate log groups for different microservices.
* Enable retention (7–30 days) to reduce cost.

### **Security**

* Least privilege IAM for worker nodes.
* Don’t expose CloudWatch endpoints publicly.

### **Cost & Performance**

* Collect only required log files.
* Avoid collecting entire `/var/log/*` folder.
* Use log filters to reduce noise.

### **Integrations**

* Combine logs with CloudWatch Metrics.
* Use CloudWatch Dashboards or Container Insights.
* Optionally forward logs to OpenSearch or SIEM.

---
---
---


