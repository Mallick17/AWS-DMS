# AWS-DMS
## DMS Data Flow from RDS to DMS for CDC: Detailed Breakdown
This pipeline captures database changes in near real-time and prepares them for downstream delivery (e.g., to Kinesis Data Streams). It's designed for minimal source impact, with DMS acting as a managed extractor.

<details>
    <summary>Click to view the flow of Data from RDS to Kinesis Data Streams</summary>

#### High-Level Overview for This Segment
```
[Source: RDS (e.g., MySQL/PostgreSQL/Aurora)] 
    ↓ (CDC via binary logs/WAL/supplemental logs)
[AWS DMS Task: Full Load + Ongoing Replication]
    ↓ (JSON-formatted records with metadata)
[Output: Ready for Delivery to Targets like Kinesis Data Streams]
```
- **Latency**: <1s for CDC changes; full load varies by data size (minutes to hours).
- **Throughput**: DMS scales with replication instance size (e.g., 1K-100K records/s).
- **Cost Drivers**: DMS replication instance hours (~$0.036/hour for t3.micro) + any data transfer.
- **Use Case Example**: Capturing order updates in an RDS e-commerce database for real-time syncing to analytics systems.

Now, breaking down every component, configuration, behavior, and parameter.

#### 1. Source: Amazon RDS with CDC Setup
- **What Happens**: RDS serves as the source relational database (supports MySQL, PostgreSQL, Oracle, SQL Server, MariaDB, Aurora variants). CDC detects and captures database changes at the row level, including Data Manipulation Language (DML) operations like INSERT, UPDATE, DELETE, and optionally Data Definition Language (DDL) like ALTER TABLE. This is done without polling—instead, it leverages the database's native logging mechanisms to stream changes efficiently.
- **How CDC Works Here**:
  - **MySQL/Aurora MySQL Compatibility**: Requires binary logging enabled. DMS reads the binary log files (binlogs), which record changes in row format for granular capture. Binlogs are stored in RDS storage (up to 100% of DB size) and rotated based on size/time.
    - Process: RDS writes changes to binlogs → DMS connects via replication user and tails the logs starting from a specific position (e.g., binlog file and offset).
  - **PostgreSQL/Aurora PostgreSQL Compatibility**: Uses logical replication via Write-Ahead Logging (WAL). DMS creates a replication slot (a named cursor in the WAL) to stream decoded changes.
    - Process: RDS enables logical decoding → DMS subscribes to the slot and receives tuple-level changes (e.g., old/new values).
  - **Other Engines (e.g., Oracle, SQL Server)**: Similar log-based—Oracle uses LogMiner/Redo logs; SQL Server uses transaction logs with CDC tables.
  - **Behavior**: Changes are captured as **before/after images** (pre-change and post-change row states) with rich metadata (e.g., transaction ID, commit timestamp, schema name, table name, operation type: I for INSERT, U for UPDATE, D for DELETE). Full load phase snapshots the entire table(s) using consistent reads (e.g., MVCC in PostgreSQL), then seamlessly switches to CDC without data loss.
    - **Transaction Handling**: DMS buffers incomplete transactions until commit or rollback. Atomicity is preserved—e.g., a multi-statement txn appears as a single unit.
    - **Schema Evolution**: If DDL occurs (e.g., ADD COLUMN), DMS can propagate it if configured, but may require task restart for complex changes.
    - **Large Objects (LOBs)**: Handled in chunks (default 32 KB) to avoid memory exhaustion; full LOBs if specified, but risks performance hits.
    - **Error and Recovery**: If RDS crashes or networks fail, DMS resumes from the last checkpoint (stored in its metadata store, like an internal DynamoDB-like table). No data duplication unless retries overlap (rare, mitigated by sequence checks).
- **Configuration for This Pipeline**:
  - **RDS Instance Setup**: Use a production-grade instance (e.g., db.m5.large) with Multi-AZ for high availability (HA) to minimize downtime during CDC. Enable automated backups (point-in-time recovery) and Performance Insights for monitoring CDC overhead.
  - **Parameter Group Modifications** (via RDS Console/CLI):
    - For MySQL: `binlog_format = ROW` (captures full row images, not statements); `binlog_row_image = FULL` (includes all columns, not just changed ones); `binlog_checksum = NONE` (for compatibility); `expire_logs_days = 7` (retain binlogs long enough for DMS catch-up); `max_binlog_size = 1GB` (controls rotation frequency).
    - For PostgreSQL: `rds.logical_replication = 1` (enables WAL decoding); `wal_sender_timeout = 0` (no timeout for DMS slot); `max_replication_slots = 10` (allow multiple DMS tasks); `max_wal_senders = 10` (concurrent streamers).
    - General: Increase `max_connections` by 10-20% (e.g., to 200) to accommodate DMS connections without starving app traffic; `innodb_log_file_size = 1GB` (MySQL) for larger txns.
  - **Replication User Setup**: Create a dedicated user in RDS with privileges: `REPLICATION SLAVE, REPLICATION CLIENT` (MySQL); `rds_replication` role (PostgreSQL). Grant SELECT on tables for full load.
  - **Endpoint Configuration in DMS**: Define RDS as source endpoint with connection string (e.g., JDBC-like: `jdbc:mysql://rds-endpoint:3306/dbname`), credentials, and engine type.
  - **Why This Behavior?**: Log-based CDC is efficient (overhead <5% CPU on source DB) because it avoids table scans or triggers. It's near real-time (sub-second for small changes) as logs are appended synchronously. If parameters are misconfigured (e.g., `binlog_format = STATEMENT`), capture becomes less reliable—losing column-level details, leading to incomplete before/after images or errors in DMS parsing.
- **Little Details & Varying Parameters**:
  - **Overhead Monitoring**: CDC adds ~2-10% IOPS (Input/Output Operations Per Second) due to log writes; use CloudWatch metrics like `BinLogDiskUsage` (MySQL) or `ReplicationSlotLag` (PostgreSQL). If lag >1GB, risk slot eviction and data loss.
  - **Full Load Tuning**: For large DBs (e.g., 1TB), set `MaxFullLoadSubTasks = 8` in DMS (detailed below) to parallelize table dumps, reducing time from hours to minutes. Behavior: Sub-tasks use separate connections, but high concurrency can spike RDS CPU to 80-100%, causing app slowdowns.
  - **Varying Parameters Examples**:
    - High-Volume RDS (e.g., 1M transactions/hour): Increase `binlog_cache_size = 64MB` (MySQL) to buffer more changes before flush; behavior shifts to fewer I/O but higher memory use—if exhausted, txns fail with errors, halting CDC until resolved.
    - Serverless Aurora (as of 2025 updates): CDC behaves with auto-pause/resume; during scale-down (zero activity), replication slots persist but may accumulate lag (1-5 minutes on resume). Set `MinCapacity = 1` to avoid pauses; varying to 0 saves costs but increases initial CDC latency.
    - LOB-Heavy Tables: If `rds.large_objects = 1` (custom param), DMS fetches LOBs on-demand; varying `LobChunkSize = 1024KB` chunks larger, reducing network calls but increasing per-record size (e.g., a 10MB BLOB becomes 10 records, amplifying downstream throughput needs).
    - Edge Case: Multi-Master RDS (e.g., Aurora Global): DMS supports reading from one primary; if failover occurs, manually update endpoint—behavior: Automatic failover in DMS 3.5+ (2025) detects and switches with <1min downtime.
  - **Security & Compliance**: Enable RDS encryption (KMS) for logs; use VPC endpoints for DMS-RDS traffic. IAM database authentication for replication user adds token-based security.

#### 2. AWS DMS: CDC Extraction and Delivery Preparation
- **What Happens**: DMS is the managed migration/replication service that extracts CDC from RDS and formats it for delivery. It runs as a **replication task** on a **replication instance** (EC2-like, e.g., dms.t3.medium), handling both full load (initial data copy) and ongoing CDC.
- **Data Flow Behavior**:
  - DMS connects to RDS via the replication user and starts tailing logs/slots.
  - For full load: Queries tables in parallel (consistent snapshot at task start time), serializes to JSON/Avro/CSV.
  - For CDC: Continuously reads log changes (e.g., every 1-5 seconds), decodes them, and buffers in-memory (up to 1GB default) before formatting.
  - Records are enriched: JSON structure like `{ "metadata": { "timestamp": "2025-11-07T12:00:00Z", "record-type": "data", "operation": "update", "partition-key-type": "schema-table", "schema-name": "public", "table-name": "orders", "transaction-id": "12345" }, "before": { "id": 1, "status": "pending" }, "after": { "id": 1, "status": "shipped" } }`.
  - **Switchover**: After full load, DMS marks "CDC start point" (e.g., binlog position) and begins streaming without gaps.
  - **Latency**: <1s for CDC (tunable); full load: 10-100 rows/s per sub-task depending on instance size.
- **Configuration for CDC**:
  - **Replication Instance**: Choose size based on load (t3.micro for <1K records/s; r5.large for high-volume). Multi-AZ for HA. Storage: Auto-allocated (100GB default).
  - **Endpoint Settings (Source Side)**: As above (RDS connection). Test connection in DMS console.
  - **Task Creation (Console/CLI/API)**:
    - Task Type: Full load + replicate ongoing changes.
    - **Task Settings (JSON)**:
      - `TargetMetadata`: `{ "BatchApplyEnabled": false, "SupportLobs": true, "LobMaxSize": 0, "LimitedSizeLobMode": false, "LobChunkSize": 64, "InlineLobMaxSize": 0 }` (handles full LOBs without truncation).
      - `FullLoadSettings`: `{ "MaxFullLoadSubTasks": 8, "TransactionConsistencyTimeout": 600, "CommitRate": 10000, "ParallelLoadThreads": 0, "ParallelLoadBufferSize": 100 }` (parallelism for speed; timeout waits for open txns).
      - `ChangeProcessingTuning`: `{ "BatchApplyPreserveTransaction": true, "BatchApplyTimeoutMin": 1, "BatchApplyTimeoutMax": 30, "BatchApplyMemoryLimit": 500, "BatchSize": 1000, "CommitTimeout": 1, "MinTransactionSize": 1000, "MemoryKeepTime": 60, "MemoryLimitTotal": 1024, "StatementCacheSize": 50 }` (batches for efficiency; preserves txn boundaries).
      - `ErrorBehavior`: `{ "DataErrorPolicy": "LOG_ERROR", "DataTruncationErrorPolicy": "LOG_ERROR", "DataErrorEscalationPolicy": "SUSPEND_TABLE", "DataErrorEscalationCount": 50, "TableErrorPolicy": "SUSPEND_TABLE", "TableErrorEscalationPolicy": "STOP_TASK", "TableErrorEscalationCount": 50, "RecoverableErrorCount": -1, "RecoverableErrorInterval": 5, "RecoverableErrorThrottling": true, "RecoverableErrorThrottlingMax": 1800, "ApplyErrorDeletePolicy": "IGNORE_RECORD", "ApplyErrorInsertPolicy": "LOG_ERROR", "ApplyErrorUpdatePolicy": "LOG_ERROR", "ApplyErrorEscalationPolicy": "STOP_TASK", "ApplyErrorEscalationCount": 50, "ApplyErrorFailOnTruncationDdl": false, "FullLoadIgnoreConflicts": true }` (logs errors, suspends problematic tables, infinite recoverable retries).
      - `ValidationSettings`: `{ "EnableValidation": true, "ValidationMode": "ROW_LEVEL", "ThreadCount": 5, "FailureMaxCount": 10000, "TableFailureMaxCount": 1000, "InvalidRecordsMaxRecords": 500, "OnlyRowsInvalid": false, "PartitionSize": 10000 }` (post-load row count/hash checks).
    - **Table Mappings**: JSON rules to select tables: `{ "rules": [ { "rule-type": "selection", "rule-id": "1", "rule-name": "SelectAll", "object-locator": { "schema-name": "%", "table-name": "%" }, "rule-action": "include", "filters": [] } ] }`. Add transformations (e.g., rename columns).
    - **CDC-Specific Settings**: `{ "CdcStartPosition": "now", "CdcStartTime": null, "CdcStopPosition": null, "CdcMaxBatchInterval": 30, "CdcMinBatchSize": 1000, "SlotName": "dms_slot", "EnableBitmapAttachments": false }` (starts from current time; batch interval in seconds).
  - **Target Preparation**: While this segment stops at DMS output, configure target endpoint (e.g., Kinesis) with `AddColumnMetadata=true` to include op details.
- **Why This Behavior?**:
  - DMS is **source-agnostic and fault-tolerant**—decouples from RDS via asynchronous replication, resuming from checkpoints on failures. Batching reduces overhead (e.g., 1K records per apply vs. per-row). It's scalable: Instance upgrades add CPU/memory for faster parsing.
  - **Hotspots**: High DDL frequency may pause CDC briefly (1-10s) for schema refresh.
- **Little Details & Varying Parameters**:
  - **Throughput Tuning**: `ParallelLoadThreads=16` speeds full load 4x but increases RDS connections/load—monitor `CPUUtilization`. For CDC, `CdcMaxBatchInterval=5s` reduces latency to <5s but increases API calls downstream.
  - **Error Modes**: `DataErrorPolicy=IGNORE_RECORD` skips bad rows; varying to `SUSPEND_TABLE` isolates issues without stopping entire task. `RecoverableErrorCount=-1` enables infinite retries (default 5s interval), ideal for flaky networks but risks log buildup.
  - **2025 Updates**: Serverless DMS (GA in 2025) auto-scales tasks based on load; for Aurora Serverless v2, CDC handles scale events with zero data loss but potential 10-30s pauses. `AutoScalingSettings` in instance config.
  - **LOB/Complex Types**: `SupportLobs=true` with `LobMaxSize=0` captures unlimited size; varying to 32MB truncates, logging warnings—behavior: Downstream gets partial data, useful for summaries but lossy for blobs.
  - **Monitoring**: CloudWatch metrics: `FullLoadThroughputRowsSource`, `CDCLatencySource` (should <10s), `ReplicationTaskState`. Logs in CloudWatch Logs for debugging (e.g., "Captured change at LSN: 123").
  - **Edge Cases**: Nested transactions (PostgreSQL) preserved; if RDS parameter group changes mid-task, restart DMS. For encrypted RDS, DMS needs KMS key access.

From here, the data is sent to Kinesis Data Streams or any other services.

</details>

## Forensic Audit Trail: How ONE Row Change Is Handled from RDS to DMS

**Complete Data-handling Bible** for **RDS → DMS CDC**.

<details>
    <summary>Click to view the How ONE Row Change Is Handled from RDS to DMS</summary>

────────────────────────
**Forensic Audit Trail: How ONE Row Change Is Handled**
────────────────────────
1. **14:32:07.123456**  
   Your app runs:  
   `UPDATE orders SET status='shipped' WHERE id=98765;`

2. **14:32:07.123789** (333 µs later)  
   RDS storage engine writes the **row image** into the **redo log** (MySQL) or **WAL segment** (PostgreSQL).

3. **14:32:07.124001** (212 µs later)  
   Binary log / WAL sender flushes the change to disk **with commit marker**.

4. **14:32:07.124500**  
   DMS replication slot (named `dms_slot`) **receives the decoded tuple**:
   ```
   {
     "before": {"id":98765, "status":"pending"},
     "after":  {"id":98765, "status":"shipped"},
     "op": "U",
     "lsn": "0/2A3B4C0",
     "ts_ms": 1733500327123
   }
   ```

5. **14:32:07.124800**  
   DMS **ChangeProcessingTuning** engine:
   - Adds to **in-memory batch #427** (current size: 683 records)
   - Tags with **transaction-id 0x9F3C**
   - Applies **column-name casing rule** (uppercase → lowercase)
   - Adds **metadata**:
     ```json
     "metadata": {
       "timestamp": "2025-11-07T14:32:07.123Z",
       "operation": "update",
       "schema-name": "sales",
       "table-name": "orders",
       "partition-key": "sales-orders-98765"
     }
     ```

6. **14:32:07.126000**  
   Batch #427 reaches **1000 records** → DMS **serialises to JSON Lines**:
   ```
   {"metadata":{...},"data":{"before":{...},"after":{...}}}
   ```

7. **14:32:07.126300**  
   DMS **compresses** the 1000-line blob with **zlib level 6** (default).

8. **14:32:07.126600**  
   DMS **encrypts in-flight** with TLS 1.3 to the **target endpoint**.

9. **14:32:07.127100**  
   DMS calls **PutRecords** (batch API) against **Kinesis Data Streams**:
   - Stream: `rds-cdc-stream`
   - 1 request, 1000 records, 1.38 MB payload
   - PartitionKey = `sales-orders-98765` → lands in **shard-000000000003**
   - SequenceNumber returned: `396279487123456789012345678901`

10. **14:32:07.128500**  
    Kinesis **persists** the record **three times** across 3 AZs.

11. **14:32:07.129000**  
    DMS writes **checkpoint** to its internal table:
    `last_lsn = 0/2A3B4C0, last_seq = 3962794871…`

12. **14:32:07.130000**  
    DMS **commits** the transaction batch → **zero duplicates** even if it retries.

That is **EVERY nanosecond** of handling for ONE row.

────────────────────────
**Mini-Checklist: Prove You Covered EVERYTHING**
(put a check-mark when you see it in prod)

- [x] Row image captured in redo/WAL  
- [x] Commit marker included  
- [x] Replication slot lag < 5 MB  
- [x] Before/after images present  
- [x] LOBs chunked at 64 KB  
- [x] Transaction boundary preserved  
- [x] Column casing rule applied  
- [x] Partition key = schema-table-id  
- [x] JSON Lines + zlib  
- [x] TLS 1.3 + SSE-KMS  
- [x] PutRecords batch of 1000  
- [x] SequenceNumber stored  
- [x] Checkpoint every 30 s  
- [x] CloudWatch: CDCLatencySource = 987 ms  

```markdown
# RDS → DMS CDC Data-Handling (14:32:07.123456)
1. App → UPDATE orders → redo/WAL
2. Binlog/WAL flush → commit marker
3. DMS slot → decoded tuple (before/after)
4. In-memory batch #427 → 1000-record threshold
5. Add metadata + lowercase columns
6. JSON Lines → zlib → TLS 1.3
7. PutRecords → shard-0003 → seq 3962794871…
8. 3-AZ durable write
9. Checkpoint lsn 0/2A3B4C0
10. Zero-duplicate guarantee
```

From here, the data is sent to Kinesis Data Streams or any other services.

</details>

## **AWS Database Migration Service (DMS) – Cost Comparison**

**Region:** Asia Pacific (Mumbai)

> This provides a cost comparison between **AWS DMS Serverless** and **AWS DMS On-Demand Instances** for single-AZ and multi-AZ configurations.

### **1. AWS DMS Serverless**

#### **DMS Capacity Units (DCUs)**

* **1 DCU** = 2 GB RAM
* Pricing is billed per DCU-hour.
* Calculations assume **730 hours/month**.

#### **Serverless Pricing**

| Deployment    | Formula                   | Monthly Cost   |
| ------------- | ------------------------- | -------------- |
| **Single AZ** | 730 hrs × 0.113289353 USD | **82.70 USD**  |
| **Multi AZ**  | 730 hrs × 0.226578706 USD | **165.40 USD** |

#### **DMS Homogeneous Data Migration Costs**

* Full-load migration or continuous replication
* Pricing: **0.13 USD per hour**
* Example: **1 hr × 0.13 USD = 0.13 USD**

#### **Total Monthly Serverless Costs**

| Deployment    | Serverless | Homogeneous Migration | **Total**      |
| ------------- | ---------- | --------------------- | -------------- |
| **Single AZ** | 82.70      | 0.13                  | **82.83 USD**  |
| **Multi AZ**  | 165.40     | 0.13                  | **165.53 USD** |

---

### **2. AWS DMS On-Demand Instances**

#### **Instance Selection**

* **t3.large**
* vCPU: 2 | Memory: 8 GiB | Storage: EBS Only
* Quantity: **1 instance**

#### **Compute Pricing**

| Deployment    | Formula             | Monthly Cost   |
| ------------- | ------------------- | -------------- |
| **Single AZ** | 1 × 0.213 × 730 hrs | **155.49 USD** |
| **Multi AZ**  | 1 × 0.426 × 730 hrs | **310.98 USD** |

#### **Storage Costs (EBS gp2, 100 GB)**

| Deployment    | Formula        | Monthly Cost  |
| ------------- | -------------- | ------------- |
| **Single AZ** | 100 GB × 0.114 | **11.40 USD** |
| **Multi AZ**  | 100 GB × 0.228 | **22.80 USD** |

#### **Total Monthly On-Demand Costs**

| Deployment    | Compute | Storage | **Total**      |
| ------------- | ------- | ------- | -------------- |
| **Single AZ** | 155.49  | 11.40   | **166.89 USD** |
| **Multi AZ**  | 310.98  | 22.80   | **333.78 USD** |

### **Summary Comparison**

| Service Type                 | Single AZ (USD) | Multi AZ (USD) |
| ---------------------------- | --------------- | -------------- |
| **DMS Serverless**           | **82.83**       | **165.53**     |
| **DMS On-Demand (t3.large)** | **166.89**      | **333.78**     |

#### **Key Insight**

DMS Serverless offers **significantly lower monthly costs** compared to On-Demand instances, especially for continuous or small-scale migrations.

### The cost details and rates provided cover the entire data flow from **RDS as the source**, through **AWS DMS replication service**, and then to various **destination targets** such as Amazon S3, EC2 instances, another RDS instance, or S3 Tables.

Specifically, the billing components include:

- **RDS**: Your source database instance (not detailed in full if already existing but relevant for baseline)
- **AWS DMS**: Replication instance hourly charges, storage, and data transfer costs during migration/CDC
- **Data Transfer**: Between RDS, DMS, and destination targets varying by availability zones and regions
- **Destination Services**: Storage and request costs on S3 or S3 Tables, EC2 instance running CDC consumers, or additional RDS instances

So the overall costs combine the **DMS infrastructure and operational costs** plus the costs related to your **destination service usage** and any **data transfer fees** incurred along the way.

<details>
    <summary>Click to view the Rates</summary>

### 1. AWS DMS (Database Migration Service)

- **Replication instance hourly rates (example for common types):**  
  - dms.r5.large: ~$0.40 per hour  
  - dms.r5.xlarge: ~$0.80 per hour  
  - dms.t3.medium: ~$0.15 per hour  

- **Storage for replication instance:** $0.10 per GB-month

- **Data transfer:**  
  - Inbound: Free  
  - Outbound cross-AZ in Mumbai: $0.01 per GB (per direction)  
  - Outbound cross-region (Mumbai → SG/US): $0.02 per GB (approximate)  

- **DMS Serverless pricing:** Charged in capacity units (DCUs) at approximately $0.10 per DCU-hour (variable)  

### 2. Amazon S3 (Standard Storage)

- **Storage:** $0.025 per GB-month  
- **PUT, COPY, POST, or LIST requests:** $0.005 per 1,000 requests  
- **GET and all other requests:** $0.0004 per 1,000 requests  
- **Data retrieval (for Glacier, not applicable to standard S3):** Not applicable here  
- **Data transfer out to internet (first 1GB per month free):** $0.09 per GB thereafter  

### 3. Amazon S3 Tables (Managed Iceberg Tables)

- **Storage:** $0.0265 per GB-month (slightly higher than standard S3)  
- **PUT requests:** $0.005 per 1,000 requests  
- **Monitoring (object monitoring) requests:** $0.025 per 1,000 objects  
- **Compaction costs:**  
  - Data processed: ~$0.005 per GB  
  - Objects processed: ~$0.002 per 1,000 objects  
- **Data transfer:** Same as standard S3 (varies by AZ/region)  

### 4. Amazon EC2 Instances (Mumbai Region)

- **Common instance hourly prices:**  
  - t3.medium (2 vCPU, 4 GiB RAM): ~$0.026 per hour  
  - m5.large (2 vCPU, 8 GiB RAM): ~$0.096 per hour  
  - r5.large (2 vCPU, 16 GiB RAM): ~$0.162 per hour  

- **EBS Storage:** Around $0.10 per GB-month (gp3 volume)  
- **Data transfer:**  
  - Same AZ: Free  
  - Inter-AZ in Mumbai: $0.01 per GB (per direction)  
  - Cross-region outbound: $0.02 per GB (approximate)  

### 5. Amazon RDS Instances (Mumbai Region)

- **Common instance hourly prices (on-demand, MySQL/PostgreSQL):**  
  - db.t3.medium: ~$0.067 per hour  
  - db.m5.large: ~$0.15 per hour  
  - db.r5.large: ~$0.267 per hour  

- **Storage:**  
  - General Purpose (SSD): ~$0.115 per GB-month  
- **I/O Requests:** $0.20 per million requests (applicable only to magnetic storage)  
- **Backup storage:** $0.095 per GB-month (beyond free tier)  
- **Data transfer:**  
  - Same AZ: Free  
  - Inter-AZ Mumbai: $0.01 per GB (per direction)  
  - Cross-region outbound: $0.02 per GB (approximate)  

### 6. Other Relevant Rates

- **AWS Glue:**  
  - $0.44 per DPU-Hour (Data Processing Unit = 4 vCPU, 16 GB RAM)  
  - Catalog storage and requests: Free up to certain limits  

- **Amazon MSK Connect:**  
  - $0.11 per MSK Connect Unit (1 vCPU, 4 GiB memory) per hour  

- **Amazon MSK Kafka Broker:**  
  - kafka.m5.large broker: ~$0.21 per hour  
  - Storage: $0.10 per GB-month  

### Summary of Key Rates (Mumbai Region)

| Resource                          | Unit                        | Rate (USD)                           |
|----------------------------------|-----------------------------|------------------------------------|
| AWS DMS replication instance     | per hour (dms.r5.large)     | $0.40                              |
| AWS DMS storage                  | per GB-month                | $0.10                              |
| S3 storage                      | per GB-month                | $0.025                             |
| S3 PUT requests                 | per 1,000 requests          | $0.005                             |
| S3 GET requests                 | per 1,000 requests          | $0.0004                            |
| S3 Tables storage               | per GB-month                | $0.0265                            |
| S3 Tables object monitoring      | per 1,000 objects           | $0.025                             |
| S3 Tables compaction (data)      | per GB                      | $0.005                             |
| S3 Tables compaction (objects)   | per 1,000 objects           | $0.002                             |
| EC2 instance (t3.medium)         | per hour                   | $0.026                             |
| EC2 instance (m5.large)          | per hour                   | $0.096                             |
| RDS instance (db.t3.medium)      | per hour                   | $0.067                             |
| RDS instance (db.m5.large)       | per hour                   | $0.15                              |
| Data transfer (same AZ)           | per GB                     | $0.00                              |
| Data transfer (different AZs Mumbai) | per GB outbound          | $0.01                              |
| Data transfer (cross-region outbound) | per GB outbound          | $0.02 (approximate)                |
| AWS Glue DPU                    | per DPU-hour                | $0.44                              |
| MSK Connect Unit                | per hour                   | $0.11                              |
| MSK Kafka Broker (m5.large)     | per hour                   | $0.21                              |
  
</details>


---
