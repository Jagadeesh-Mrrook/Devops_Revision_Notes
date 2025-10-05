## Database Layer Overview — Quick Revision Notes

* **Purpose:** Store application data (structured & unstructured), support queries, transactions, and high availability.
* **Types:**

  * Relational (SQL): Structured tables, ACID, e.g., MySQL, PostgreSQL, Aurora
  * Non-relational (NoSQL): Flexible schema, scalable, e.g., DynamoDB, MongoDB
* **DevOps Role:** Provision DBs, ensure HA, backups, scaling, security, monitor performance.
* **Why not S3/EBS/EFS:** These don't support structured queries, transactions, indexing, or concurrent access efficiently.


---
---

## Relational vs NoSQL Database Selection — Quick Revision Notes

* **SQL Databases:** Structured data, predefined schema, ACID transactions. Examples: MySQL, PostgreSQL, Aurora
* **NoSQL Databases:** Flexible schema, high throughput, horizontal scalability. Examples: DynamoDB, MongoDB
* **DevOps Role:** Provision & scale DB, monitor metrics, setup HA, backups, security
* **Key Concepts:** Schema (SQL), Flexible Schema (NoSQL), Indexing, Scaling, HA, Backups

---
---

## Connection Pooling and Database Access — Quick Revision Notes

* **Connection Pooling:** Reuse existing DB connections, reduces overhead, libraries: HikariCP, PgBouncer
* **Database Access:** Drivers (e.g., JDBC for Java) connect app to DB
* **Flow:** [App] → [Connection Pool] → [DB]
* **DevOps Role:** Configure DB max_connections, network access (VPC/SG), setup HA (Multi-AZ, read replicas), monitor metrics, scale DB if needed
* **NoSQL Exception:** DynamoDB is serverless, no pooling needed
* **Best Practices:** Proper pool sizing, monitor metrics, backups, secure DB, alert on satur

---
---

## Database Scaling Strategies — Quick Revision Notes

* **Vertical Scaling:** Increase CPU/RAM/storage of a single DB instance. Downtime required (except Aurora supports online vertical scaling).
* **Read Replicas (Horizontal Scaling):** Duplicate primary DB for read-heavy workloads; primary handles writes; monitor replication lag.
* **Sharding:** Split dataset across multiple DB instances (shards) using a shard key; each shard stores a subset of data; used for very large datasets.
* **DevOps Focus:** Monitor metrics, configure vertical scaling, manage read replicas, plan sharding for large datasets, ensure HA and backups.
* **AWS Examples:**

  * RDS: vertical scaling via instance upgrade, horizontal via read replicas, Aurora supports online vertical scaling.
  * DynamoDB: automatic sharding/partitioning, autoscaling for read/write throughput.
* **Best Practices:** Start with vertical scaling, add read replicas as load grows, implement sharding only when necessary, monitor metrics, plan backups and HA.

---
---


## Database Backup and Restore / Snapshot Strategies — Quick Revision Notes

* **Purpose:** HA, disaster recovery, compliance, testing.
* **Managed DBs (RDS/Aurora/DynamoDB):**

  * Automated backups + transaction logs for point-in-time recovery.
  * Manual snapshots as needed.
  * Retention configurable (e.g., 7–15 days).
* **Self-managed DBs (EC2/K8s):**

  * Manual EBS snapshots or DB dumps.
* **DevOps Focus:**

  * Automate and monitor backups.
  * Test restore procedures regularly.
  * Ensure encryption and correct IAM permissions.
* **Best Practices:**

  * Cross-region snapshots for DR.
  * Use snapshots to clone dev/test environments.
  * Incremental snapshots for large DBs.
* **AWS Examples:**

  * RDS: automated & manual snapshots, PITR.
  * DynamoDB: on-demand backups, continuous backups, cross-region replication.
  * EBS: snapshots for volumes attached to self-managed DBs.

---
---

## DATABASE SECURITY — Quick Revision Version

### • Concept / What

Protects database data from unauthorized access, loss, or misuse using encryption, IAM, and network restrictions.

### • Why / Purpose / Use Case

* Data confidentiality & integrity
* Compliance (GDPR/HIPAA)
* Secure app-DB communication in cloud/DevOps setups

### • How it Works / Key Points

* **Encryption at Rest:** KMS-managed, applies to DB, snapshots, backups, read replicas
* **Encryption in Transit:** SQL via SSL/TLS (DB port), NoSQL via HTTPS SDK calls
* **Authentication & Authorization:** IAM roles/policies, least privilege principle
* **Network Security:** Private subnets, Security Groups, only app server access
* **Monitoring & Auditing:** CloudTrail, CloudWatch, RDS Enhanced Monitoring

### • Common Issues / Errors

* Security Group misconfigurations
* SSL/TLS not enforced
* KMS key deletion
* Overly permissive IAM roles

### • Troubleshooting / Fixes

* Test DB port connectivity
* Check SSL/TLS settings in parameter group
* Validate IAM permissions
* Detect public access using AWS Config/Security Hub

### • Best Practices / Tips

* Enable KMS encryption at DB creation
* Enforce SSL/TLS for all DB traffic
* Use IAM authentication instead of passwords
* Keep DBs in private subnets
* Enable audit logging & monitoring
* Rotate keys and credentials periodically

### ✅ Summary Flow

```
Client → ALB → Application Server (JDBC/SDK) → DB (TLS/HTTPS) → Data at Rest Encrypted (KMS) → Backup/Snapshots
```

---
---

## DATABASE CONNECTIVITY — Quick Revision Version

### • Concept / What

Manages how the application communicates with the database using drivers, ports, APIs, and connection pools.

### • Why / Purpose / Use Case

* Secure and efficient app ↔ DB communication
* Handles high load via connection pooling
* Enables DevOps to manage network, high availability, and scaling

### • How it Works / Key Points

* **SQL DBs:** JDBC/DB driver → DB port (3306/5432) → RDS/Aurora
* **NoSQL DBs:** SDK → HTTPS REST API → DynamoDB/MongoDB
* **Connection Pooling:** Reuse existing connections, max/min limits (HikariCP, PgBouncer)
* **DevOps Role:** Security groups, SSL/TLS, IAM policies, DB scaling
* **Dev Role:** Driver integration and query logic

### • Common Issues / Errors

* Connection refused (firewall/security group issues)
* Pool exhaustion
* SSL/TLS mismatch
* Misconfigured IAM roles

### • Troubleshooting / Fixes

* Test DB port connectivity (`telnet` / `nc`)
* Validate JDBC/SDK configuration
* Monitor and tune connection pool
* Check IAM policies for NoSQL access
* Enable CloudWatch/monitoring dashboards

### • Best Practices / Tips

* Keep DBs in private subnets
* Enforce SSL/TLS or HTTPS
* Monitor connection usage; scale read replicas as needed
* Use IAM roles instead of static credentials
* DevOps manages infrastructure, Devs manage driver logic
* Tune connection pool regularly

### ✅ Summary Flow

```
Frontend Client → ALB → Application Server →
   SQL DB: JDBC Driver → Port 3306 → RDS Instance (TLS optional)
   NoSQL DB: SDK → HTTPS REST API → DynamoDB / MongoDB
Connection Pool manages re-use and scaling of connections.
Encrypted data in transit via TLS/HTTPS. Access controlled by IAM policies and Security Groups.
```


---
---

# Database Layer — High Availability & Failover (Quick Revision)

- **Multi-AZ Deployment:**  
  - Primary + Standby in different AZs.  
  - Automatic failover if primary fails.

- **Automated Snapshots:**  
  - Taken by AWS, stored in internal S3, replicated across AZs.  
  - Logical snapshots visible in RDS console.  

- **Double-Failure Scenario:**  
  - Both primary & standby fail → manual restore from snapshot.  
  - Restore via RDS console, CLI, or SDK.  
  - For cross-region recovery, copy snapshot to target region first.

- **Disaster Recovery:**  
  - Use snapshots for PITR.  
  - Cross-region copies for full-region failure.

- **Best Practices:**  
  - Enable Multi-AZ + automated backups.  
  - Configure cross-region snapshot replication for DR.  
  - Regularly test restores and monitor backups.


---
---

# Database Layer: Common Issues & Troubleshooting (Quick Revision)

## Concept / What
Typical database problems and troubleshooting steps to maintain **high availability, performance, and reliability**.

---

## Why / Purpose / Use Case
- Ensure smooth database operations.
- Handle **connection limits, replication lag, deadlocks, slow responses**.
- DevOps role: monitor, alert, and scale database infrastructure.

---

## Common Issues
- **Connection limits** → max concurrent connections exceeded.
- **Replication lag** → replicas fall behind primary → stale reads.
- **Deadlocks** → transactions block each other → failures.
- **Slow responses** → high load, unoptimized queries, or limited resources.

---

## Troubleshooting / Fixes
- Use **connection pooling** to reuse connections.
- Scale DB vertically or add **read replicas**.
- Monitor **replication lag metrics**; optimize write traffic.
- Check **slow queries** via logs or performance insights.
- Implement **retry and error-handling logic** in applications.

---

## Best Practices / Tips
- Continuously monitor metrics: CPU, memory, connections, ReplicaLag.
- Set alerts for spikes or slow responses.
- Optimize queries and indexing to prevent deadlocks.
- Separate **read-heavy vs write-heavy workloads** using replicas.
- Automate backups and snapshot retention for disaster recovery.

---

## Interview-Friendly Explanation Phrases
- *“Sometimes DB hits connection limits; I monitor and use pooling.”*
- *“Replica lag may cause stale reads; we monitor ReplicaLag metrics.”*
- *“Deadlocks occur when transactions block each other; logs help identify and fix.”*
- *“DB can slow due to high load or inefficient queries; we scale or optimize as needed.”*


---
---
