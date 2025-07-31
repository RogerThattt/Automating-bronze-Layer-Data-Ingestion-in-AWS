# Automating-bronze-Layer-Data-Ingestion-in-AWS

Here is a **detailed and structured plan** for implementing a **metadata-driven Bronze Layer Data Ingestion Pipeline in AWS**, adapted from your original Azure Databricks version.

This version uses **Amazon S3**, **AWS Glue**, **AWS Secrets Manager**, **AWS Glue Catalog**, and **Databricks on AWS** (or EMR if preferred) to build a similar ingestion framework.

---

## 🔁 Bronze Layer Data Ingestion Plan (AWS Version)

### ✅ **Objective**

To build a **scalable**, **automated**, and **metadata-driven ETL pipeline** using **AWS and Databricks**. This pipeline ingests various CSV files from **Amazon S3**, dynamically applies schema and data types defined in a **metadata config table**, and writes to **Delta Lake tables in the Bronze layer**.

---

## 📐 Architecture Overview

| Component         | Description                                                                 |
| ----------------- | --------------------------------------------------------------------------- |
| **Source**        | Amazon S3: Organized as `s3://<bucket>/sales/{object_name}/{year}/{month}/` |
| **Processing**    | Databricks on AWS (using PySpark notebooks)                                 |
| **Configuration** | Delta table in Databricks for metadata configuration                        |
| **Security**      | AWS Secrets Manager (to store S3 credentials)                               |
| **Destination**   | Delta Lake tables on S3 (Bronze Layer)                                      |

---

## 🛠️ Step 1: AWS Prerequisites Setup

### 1.1. **S3 Bucket Setup**

* Create an S3 bucket (e.g., `company-data-lake`)
* Folder structure:

  ```
  s3://company-data-lake/sales/agents/YYYY/MM/
  s3://company-data-lake/sales/products/YYYY/MM/
  ```

### 1.2. **AWS Secrets Manager**

* Store your S3 credentials securely:

  * Name: `s3-access`
  * Key: `s3-access-key-id`, `s3-secret-access-key`

### 1.3. **Databricks Secret Scope for AWS**

```bash
# Use Databricks CLI or UI
databricks secrets create-scope --scope aws_secrets
databricks secrets put --scope aws_secrets --key s3-access-key-id
databricks secrets put --scope aws_secrets --key s3-secret-access-key
```

### 1.4. **Databricks Cluster**

* Create a cluster on **AWS Databricks**
* Recommended: **Runtime 12+**, single-node for development

---

## 🗃️ Step 2: Metadata Configuration Table

### 2.1. **Create Metadata Table**

Create a Delta table named `config` in Databricks:

```sql
CREATE TABLE IF NOT EXISTS config (
  system_name STRING,
  source_system STRING,
  source_object STRING,      -- JSON metadata for source
  destination_object STRING  -- JSON metadata for destination
)
USING DELTA
LOCATION 's3://company-data-lake/config/';
```

### 2.2. **Populate Config Table**

Insert JSON metadata:

```sql
-- Agents
INSERT INTO config VALUES (
  'start_sales',
  'csv',
  '{
    "source_bucket": "company-data-lake",
    "source_prefix": "sales/agents",
    "file_name_pattern": "agents_{YYYY}_{MM}_{DD}.csv",
    "columns": [
      {"column_name": "AgentID", "data_type": "integer"},
      {"column_name": "AgentName", "data_type": "string"},
      {"column_name": "DateOfJoining", "data_type": "date"}
    ]
  }',
  '{
    "destination_path": "s3://company-data-lake/bronze/agents"
  }'
);
```

Repeat for `products`, `sales`, etc.

---

## 📒 Step 3: Build Ingestion Notebook in Databricks

### 3.1. **Import Required Libraries**

```python
import json
from datetime import datetime, timedelta
from pyspark.sql.functions import col
```

### 3.2. **Get AWS Credentials Securely**

```python
access_key = dbutils.secrets.get(scope="aws_secrets", key="s3-access-key-id")
secret_key = dbutils.secrets.get(scope="aws_secrets", key="s3-secret-access-key")

spark._jsc.hadoopConfiguration().set("fs.s3a.access.key", access_key)
spark._jsc.hadoopConfiguration().set("fs.s3a.secret.key", secret_key)
spark._jsc.hadoopConfiguration().set("fs.s3a.endpoint", "s3.amazonaws.com")
```

### 3.3. **Load Metadata Config Table**

```python
config_df = spark.read.table("config")
config_list = config_df.collect()
```

### 3.4. **Dynamic Ingestion Loop**

```python
# Set date for T-1 ingestion
yesterday = datetime.now() - timedelta(days=1)
year, month, day = yesterday.strftime("%Y"), yesterday.strftime("%m"), yesterday.strftime("%d")

for row in config_list:
    source_meta = json.loads(row["source_object"])
    dest_meta = json.loads(row["destination_object"])

    file_name = source_meta["file_name_pattern"].format(YYYY=year, MM=month, DD=day)
    file_path = f"s3a://{source_meta['source_bucket']}/{source_meta['source_prefix']}/{year}/{month}/{file_name}"
    print(f"Processing: {file_path}")

    # Load CSV
    df = spark.read.option("header", "true").csv(file_path)

    # Apply schema
    for col_meta in source_meta["columns"]:
        col_name = col_meta["column_name"]
        data_type = col_meta["data_type"]
        df = df.withColumnRenamed(col_name, col_name.lower())
        df = df.withColumn(col_name.lower(), col(col_name.lower()).cast(data_type))

    # Write to Bronze layer
    destination = dest_meta["destination_path"]
    df.write.format("delta").mode("overwrite").save(destination)
    print(f"Written to: {destination}")
```

---

## ✅ Step 4: Validation and Scheduling

### 4.1. **Data Verification**

```sql
SELECT * FROM delta.`s3://company-data-lake/bronze/agents` LIMIT 10;
DESCRIBE delta.`s3://company-data-lake/bronze/agents`;
```

### 4.2. **Automate with Databricks Jobs**

1. Go to **Workflows > Jobs**
2. Click **Create Job**
3. Name: `daily_bronze_ingestion`
4. Select Notebook, attach to Cluster
5. Set **Schedule = Daily**
6. Add notifications if needed

---

## 🔒 Security & Best Practices

* Use **IAM roles** and **fine-grained S3 access** where possible
* Store all credentials in **Secrets Manager**
* Implement **retry logic**, **error handling**, and **logging** in production
* Add **checkpointing** if moving to streaming in the future

---

## 📌 Optional Enhancements

* Add `ingestion_date`, `source_file` as columns in output
* Partition Bronze tables by `year`, `month` for performance
* Implement CDC (Change Data Capture) if source supports it
* Use AWS Glue Catalog for unified metadata management

---


