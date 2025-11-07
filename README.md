# AWS-DMS
Database Migration Service
Below is a clean, concise documentation-style write-up based on your provided calculations.

---

## **AWS Database Migration Service (DMS) – Cost Comparison Documentation**

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

---
