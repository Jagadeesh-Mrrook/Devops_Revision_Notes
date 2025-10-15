### Quick Revision Notes — CloudWatch Logs Setup

**1. Purpose:**

* Used to collect, monitor, and analyze logs from servers or applications.
* Centralized log management system for AWS.

**2. How Logs Reach CloudWatch:**

* Logs are first generated and stored locally by apps (e.g., `/var/log/app.log`).
* CloudWatch Agent or CloudWatch Logs Agent is installed on the instance.
* Agent continuously pushes local logs to CloudWatch Logs service.

**3. Setup Steps:**

1. **Install Agent** (on EC2/VM):

   ```bash
   sudo yum install amazon-cloudwatch-agent
   ```
2. **Create config file** (`amazon-cloudwatch-agent.json`) — specify log file paths.
3. **Start Agent:**

   ```bash
   sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
     -a fetch-config -m ec2 -c file:/path/to/config.json -s
   ```
4. **Logs appear in CloudWatch → Log groups → Log streams.**

**4. Example Config (for Java App):**

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/app/access.log",
            "log_group_name": "ecommerce-app-access",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/app/error.log",
            "log_group_name": "ecommerce-app-error",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```

OBOBOBOBOBOB**5. Real-World Flow:**

OBOBOB* Application writes logs → CloudWatch Agent reads logs → Sends to CloudWatch Log Groups.
OBOBOB* CloudWatch can trigger alerts, dashboards, or Lambda automation from these logs.
OBOBOB
**6. Key Terms:**

* **Log Group:** Category of logs (e.g., app/environment-based)
* **Log Stream:** Sequence of logs from one instance/app
* **Metric Filter:** Extracts custom metrics from logs
* **Subscription Filter:** Sends logs to S3, Lambda, or other services

**7. Analogy:**
Like Node Exporter fetches metrics → CloudWatch Agent fetches logs.

---
---

## Quick Revision — Frontend Logging (Errors, Asset Load Times)

### 🔹 Concept / What

* Collects logs, performance metrics, and errors from the **user's browser**.
* Includes JS errors, API/network failures, asset load performance, user interactions, and custom events.

### 🔹 Why / Purpose

* Detect UI issues and broken scripts.
* Track slow asset loading and rendering times.
* Analyze user interactions and UX.
* Supports troubleshooting and performance optimization.

### 🔹 How it Works

```
[User Browser] → Frontend SDK (Sentry/CloudWatch RUM) → Collect logs & metrics → Send to backend → Dashboards/Alerts
```

**CloudWatch RUM Setup:**

1. CloudWatch Console → Application Monitoring → RUM
2. Create App Monitor: name, domain, sampling rate, allowed domains
3. Integrate JS snippet (NPM/CDN)
4. Verify data on RUM Dashboard

**Collected Data:** JS errors, API failures, page load times, device/browser info, custom events

### 🔹 Common Issues

* High log volume → app/network performance impact
* Sensitive data in logs
* SDK/script initialization errors
* Ad-blockers blocking logs

### 🔹 Fixes

* Mask PII
* Ensure SDK loads after app initialization
* Enable retry logic for failed submissions
* Monitor script load times

### 🔹 Best Practices

* Use structured logging (JSON)
* Set log levels carefully
* Sample logs in high-traffic apps
* Integrate with backend logs for end-to-end visibility
* Set CloudWatch alarms/alerts
* Archive logs (S3/ELK)

### 🔹 AWS Components

| Component          | Purpose                 |
| ------------------ | ----------------------- |
| CloudWatch RUM     | Frontend logs & metrics |
| CloudWatch Metrics | Real-time monitoring    |
| CloudWatch Alarms  | Threshold-based alerts  |

---
---

## Quick Revision — Backend Logs and Metrics

### 🔹 Concept / What

* **Logs:** Server-side events (errors, warnings, info, exceptions).
* **Metrics:** System/app performance data (CPU, memory, request rate, latency, DB query time).
* **EMF:** Embed metrics in logs (JSON) for automatic extraction.

### 🔹 Why / Purpose

* Debug backend issues.
* Monitor system health in real-time.
* Detect anomalies and trigger alerts.
* Audit/compliance and user behavior analysis.

### 🔹 How it Works

```
[Backend Server/App] → Generate Logs & Metrics → CloudWatch Agent/SDK/EMF → CloudWatch Logs & Metrics → Dashboards & Alarms
```

### 🔹 Common Issues

* Missing logs (agent/IAM issues).
* Metrics not reported (wrong namespace/dimensions).
* Excessive logging → high cost.
* Delayed ingestion.

### 🔹 Fixes

* Check CloudWatch Agent logs.
* Verify IAM permissions.
* Filter log verbosity.
* Correct namespace/dimensions.
* Use batching/compression.

### 🔹 Best Practices

* Structured logging (JSON).
* Separate log groups for apps/environments.
* Use EMF for metrics in logs.
* Set alarms on critical metrics (CPU, error rate).
* Archive logs to S3/Glacier.
* Monitor containers with FluentBit/FluentD.
* Track system, app, and business metrics (request count, latency, error rate, DB queries, queue depth).

### 🔹 AWS Components

| Component          | Purpose                   |
| ------------------ | ------------------------- |
| CloudWatch Logs    | Store backend logs        |
| CloudWatch Metrics | Track performance metrics |
| CloudWatch Alarms  | Threshold alerts          |
| EMF                | Embed metrics in logs     |

### 🔹 Interview Tips

* Mention metrics beyond CPU/memory: request rate, latency, error rate, DB query times, business KPIs.
* Highlight EMF usage.
* Differentiate monitoring vs logging.
* Explain proactive monitoring using alarms.

---
---

### ⚡ Quick Revision – Database Monitoring and Alerts

**Purpose:**
Monitor database performance, availability, and error trends using CloudWatch + EMF.

**Key Metrics to Track:**

* CPU Utilization
* Freeable Memory
* Disk I/O (Read/Write Latency, Throughput)
* Database Connections
* Query Execution Time
* Replica Lag (for read replicas)
* Error Rates (timeouts, connection drops)

**Tools Used:**

* **CloudWatch Metrics** → Auto-tracked for RDS, DynamoDB, etc.
* **CloudWatch Logs + EMF** → For custom DB/application metrics
* **CloudWatch Alarms** → Notify via SNS when thresholds are breached

**EMF (Embedded Metric Format):**

* JSON structure used inside logs to **convert logs → metrics** automatically.
* Helps track **custom DB or query performance** metrics.

**Example:**

```json
{
  "_aws": {
    "Timestamp": 1690000000000,
    "CloudWatchMetrics": [{
      "Namespace": "MyApp/DB",
      "Dimensions": [["DBInstance"]],
      "Metrics": [{"Name": "QueryLatency", "Unit": "Milliseconds"}]
    }]
  },
  "DBInstance": "prod-db",
  "QueryLatency": 350
}
```

**Alert Setup:**

1. Push metrics via EMF → CloudWatch.
2. Create alarms (e.g., CPU > 80% for 5 mins).
3. Trigger SNS → email/Slack notification.

**Benefits:**

* Proactive detection of DB issues.
* Quick troubleshooting with correlated logs.
* Custom metrics help catch query-specific issues.

---
---

# End-to-End Request Tracing — Quick Revision Version

### • Concept / What

Tracks a user request’s full journey — from frontend → backend → database → external services — showing how long each service takes and where issues occur.

### • Why / Purpose / Use Case

* Detect bottlenecks and slow services.
* Improve user experience and performance.
* Trace failures across microservices.
* Critical for debugging and root cause analysis.

### • How it Works / Steps

1. App is instrumented → assigns `trace_id` to each request.
2. Traces collected by agent/daemon (e.g., AWS X-Ray daemon, OpenTelemetry collector).
3. Sent to tracing system (AWS X-Ray, Jaeger, Zipkin, Datadog).
4. Visualized in dashboards (service maps).

**Example Flow:**

```
Frontend → API Gateway → Backend A → Service B → Database
```

### • DevOps vs Developer Role

* **DevOps:** Setup infrastructure, agents, IAM roles, exporters.
* **Developers:** Integrate SDK, propagate trace headers, generate trace data.

### • Common Issues

* Missing trace IDs.
* Agent misconfigurations.
* Excessive sampling → noise or cost.

### • Troubleshooting / Fixes

* Pass `X-Amzn-Trace-Id` across services.
* Check agent logs.
* Adjust sampling rules.

### • Best Practices

* Use correlation IDs in all requests.
* Combine logs + metrics + traces (full observability).
* Use AWS ServiceLens or OpenTelemetry.
* Avoid sensitive data in traces.

### • Common Interview Follow-ups

* Difference: metrics vs logs vs traces.
* How to set up tracing in microservices?
* What is AWS X-Ray sampling?
* How to integrate tracing in EKS or Lambda?

---
---


