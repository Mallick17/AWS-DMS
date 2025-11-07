# AWS-DMS
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

Table with **data transfer rates per GB** for different transfer scenarios in the Mumbai region (ap-south-1), for various destination types:

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
