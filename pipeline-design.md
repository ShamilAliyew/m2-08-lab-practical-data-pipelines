# Task 1: Pipeline Architecture Diagram
## 1.1 — Draw the end-to-end architecture                  
                         
                         ┌──────────────────────────┐
                         │  Historical Batch Source │
                         │  Online Retail.xlsx      │
                         └─────────────┬────────────┘
                                       │
                                       ▼
                               Batch Ingestion
                                       │
                                       │
                                       │
┌──────────────────────────┐           ▼
│  Live Transaction Stream │    ┌───────────────┐
│  (Row-by-row events)     │───▶│ Stream        │
└─────────────┬────────────┘    │ Ingestion     │
              │                 │ (Message Bus) │
              │                 └───────┬───────┘
              │                         │
              └──────────────┬──────────┘
                             ▼
                     ┌─────────────────┐
                     │   Raw Layer     │
                     │   Landing Zone  │
                     │  (Parquet)      │
                     └────────┬────────┘
                              ▼
                     ┌─────────────────┐
                     │  Validation     │
                     │ Schema + Rules  │
                     └───────┬─────────┘
                             │
                 ┌───────────┴───────────┐
                 ▼                       ▼
        ┌─────────────────┐     ┌──────────────────┐
        │ Clean Layer     │     │ Quarantine /     │
        │ Valid Records   │     │ Dead Letter Area │
        └────────┬────────┘     └──────────────────┘
                 ▼
         ┌────────────────────┐
         │ Transformation     │
         │ Feature Engineering│
         └────────┬───────────┘
                  ▼
           ┌───────────────┐
           │ Feature Layer │
           │ ML Features   │
           └───────┬───────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
┌───────────────┐      ┌────────────────┐
│ BI Dashboard  │      │ ML Model       │
│ Daily Reports │      │ Customer Value │
└───────────────┘      └────────────────┘


            ┌─────────────────────────────┐
            │ Monitoring & Alerting       │
            │ Freshness, Volume, Quality  │
            └─────────────────────────────┘




    # Data Pipeline Architecture — Text Explanation

This pipeline illustrates the stages that data passes through in a real production environment and demonstrates its architecture. The purpose is to process both historical batch data and real-time transaction events, validate and clean them, and prepare them for analytics and machine learning. The pipeline ensures reliable and high-quality data for both BI dashboards and ML models.

Data comes from two main sources: a historical batch file containing all past transactions in the Online Retail.xlsx dataset, and a live transaction stream simulating real-time customer orders. The batch dataset is loaded once into the pipeline, while the live stream provides new transactions one row at a time. Both sources must be consistent in format before entering the pipeline.

The ingestion layer brings the data into the pipeline and converts it into a consistent format. The batch file is read and structured, while the streaming events are consumed through a message bus such as Kafka. After ingestion, the data is written to the raw storage or landing zone for further processing.

In the raw layer, data is stored without any transformation or cleaning. Files are typically stored in Parquet format and organized by date. This layer serves as a backup of the original data and allows reprocessing if needed in the future.

The validation stage checks the correctness of incoming data. Schema validation ensures that all required columns exist and have the correct data types. Business rules are also enforced, such as ensuring Quantity is greater than 0 and UnitPrice is non-negative. Any invalid records are sent to a quarantine or dead letter area instead of being silently dropped, allowing inspection and potential reprocessing later. Validated data proceeds to the transformation stage.

In the transformation stage, validated data is prepared for analysis. Cleaning operations handle missing values appropriately, aggregations and statistical calculations are performed, and new features are engineered. Examples of transformations include calculating total price per line item, deriving customer-level metrics, and generating features such as average order value or total orders per customer. The transformed data is then sent to the clean layer.

The clean layer stores only validated, cleaned, and standardized data. This layer serves as the primary source for both BI dashboards and ML models. Data is typically stored in Parquet format or an analytical database, ensuring that it is reliable, reproducible, and can be reprocessed if necessary.

The feature layer contains aggregated and derived data specifically prepared for ML models. Examples include the total spending of a customer over the last 30 days, the number of orders, or the average transaction value. Features are updated daily to ensure that ML models have fresh and accurate inputs for predictions.

The pipeline has two main consumers. The BI dashboard provides daily sales summaries, top-selling products, and country-level breakdowns, updated nightly. The ML model predicts high-value customers, drawing features from the feature layer and being retrained on a weekly basis.

Monitoring is integrated across all stages to ensure data quality and pipeline health. It checks data freshness to confirm that data is arriving on time, monitors batch volume to detect missing or incomplete data, observes schema changes to prevent downstream errors, and evaluates quality metrics such as null rates and outliers. Alerts are triggered whenever anomalies are detected, allowing engineers to address issues promptly.




## Task 1.2 — Component Descriptions

**Data Sources**  
Data enters the pipeline from two primary sources. The historical batch file contains all past transaction records in the Online Retail dataset and is loaded once at the start. The live transaction stream delivers new events one row at a time, simulating real-time orders. Both sources provide raw transactional data that serves as input for the ingestion layer, ensuring that the pipeline can handle both historical and streaming workflows efficiently.

**Ingestion Layer**  
The ingestion layer is responsible for collecting data from both the batch file and the live stream, converting it into a consistent internal format. It receives raw data from the sources and writes it into the raw storage (landing zone) without altering the content. This layer handles variable throughput, multiple data formats, and partial failures, ensuring that all incoming data is captured reliably. Common technologies include Python scripts for batch ingestion and message brokers like Kafka for real-time streams.

**Raw Layer (Landing Zone)**  
The raw layer stores the ingested data exactly as it arrived, without any transformation or validation. Input to this layer comes from the ingestion stage, and its output is the preserved raw dataset available for downstream stages. Data is typically stored in Parquet files organized by date for efficient retrieval and reprocessing. This layer acts as a historical record and allows the pipeline to recover or reprocess data if downstream logic changes.

**Validation Stage**  
The validation stage checks incoming records for schema compliance, data type correctness, and business rules. Input comes from the raw layer, and output consists of two streams: valid records sent to the transformation stage and invalid records sent to the quarantine or dead letter area. This stage prevents bad data from propagating through the pipeline. Validation logic includes checking required columns, ensuring numeric fields have sensible values, and flagging inconsistencies such as cancellations with positive quantities.

**Quarantine / Dead Letter Area**  
This component temporarily holds invalid records identified during validation. Its input is the invalid subset of data from the validation stage, and it outputs nothing to the main pipeline unless records are manually or automatically reprocessed. Typically, this area uses storage formats like Parquet or JSON for easy inspection. The purpose is to preserve failed records for auditing, debugging, and potential correction without affecting the main data flow.

**Transformation Stage**  
The transformation stage converts validated data into analysis-ready and ML-ready formats. Input is the validated dataset from the previous stage, and output is cleaned, enriched, and feature-engineered data sent to the clean layer. Common transformations include handling missing values, aggregating customer-level statistics, deriving new columns, and computing features such as total order value. Technologies used often include Pandas for small datasets, Spark for large-scale batch processing, and SQL-based transformations in analytical databases.

**Clean Layer**  
The clean layer stores fully validated and transformed data that is safe for analysis and modeling. Input comes from the transformation stage, and output is consumed by both the BI dashboard and ML models. Data is usually stored in Parquet format or an analytical database like DuckDB, optimized for query performance. This layer provides a reliable, reproducible source of truth, enabling consistent analytics and retraining of ML models without reprocessing raw data.

**Feature Layer**  
The feature layer contains aggregated and derived data specifically prepared for ML training and real-time inference. Input is clean and transformed data, and output is a set of ML-ready features. This layer often includes time-windowed aggregates, rolling averages, or customer behavior metrics, updated on a daily or incremental schedule. Storage may be in a feature store or Parquet files, ensuring that ML models always have access to consistent and up-to-date features.

**Consumers (BI Dashboard and ML Model)**  
The pipeline has two primary consumers. The BI dashboard accesses the clean layer to produce daily sales summaries, top product reports, and country-level analyses, typically updated nightly. The ML model uses the feature layer to predict high-value customers and other metrics, with features updated daily and models retrained weekly. Both consumers rely on the layered storage architecture to access reliable, consistent, and high-quality data.

**Monitoring & Alerting**  
Monitoring oversees the entire pipeline, ensuring data freshness, completeness, schema consistency, and quality metrics. It receives input from all layers, tracking volume, null rates, outliers, and anomalies, and outputs alerts or dashboards for operational visibility. Technologies can include Prometheus, Grafana, or custom Python scripts. Effective monitoring detects problems early, reduces downtime, and guarantees that data and ML models remain accurate and reliable.


# Task 2: Validation and Error Handling Design

## Task 2.1 — Validation Rules for Online Retail Dataset

### Schema Validations (Structural Correctness)

- `InvoiceNo`: must be a non-empty string; required for all records.  
- `StockCode`: must be a non-empty string; required.  
- `Description`: string; optional but recommended.  
- `Quantity`: integer or float; required.  
- `InvoiceDate`: valid datetime format (YYYY-MM-DD HH:MM:SS); required.  
- `UnitPrice`: float; required.  
- `CustomerID`: integer; optional but required for customer-level aggregation.  
- `Country`: string; required for geographical analysis.  

These checks ensure that every record conforms to the expected structure and data types before any further processing.

### Value Range Validations (Sensible Values)

- `Quantity`: must not be zero for normal sales; can be negative only for cancellations.  
- `UnitPrice`: must be greater than or equal to 0; negative values indicate data errors unless part of a cancellation.  
- `InvoiceDate`: must not be in the future; historical dates should fall within the dataset range.  
- `CustomerID`: if present, must be positive integers.  
- `StockCode`: must be alphanumeric and non-empty; certain known codes (like promotional codes) may have specific patterns.  

These validations ensure that numeric and datetime fields have sensible, expected values, reducing errors in downstream calculations and ML feature engineering.

### Business Rule Validations (Domain Logic)

- If `InvoiceNo` starts with 'C', then `Quantity` must be negative (cancellation) and `UnitPrice` can be positive or zero.  
- If `InvoiceNo` does not start with 'C', `Quantity` must be positive.  
- For a given `InvoiceNo`, `InvoiceDate` must be the same across all line items.  
- Total line amount (`Quantity * UnitPrice`) should be greater than zero for normal sales.  
- `Country` must match known countries in the dataset (e.g., UK, Germany, France, etc.).  
- Duplicate `InvoiceNo` and `StockCode` combinations should be flagged for review if they occur in the same invoice (possible data entry errors).  

Business rules enforce logical consistency between fields and capture domain-specific knowledge about sales, cancellations, and customer behavior. These rules are critical for ensuring that analytics and ML models operate on reliable, meaningful data.

## 2.2 — Design the error handling flow

### Schema Validation Failures

**Scenario:** A record fails schema validation if required fields are missing, have the wrong type, or are malformed (e.g., `InvoiceNo` is empty, `Quantity` is a string).  

**Handling Flow:**  
- The record is **rejected entirely** and not processed further in the pipeline.  
- It is written to a **dead letter queue (DLQ)** or **quarantine table** in JSON or Parquet format, preserving the original data and a timestamp for audit purposes.  
- An **alert is triggered** via monitoring tools (e.g., email, Slack, or Prometheus/Grafana) summarizing the type and count of schema failures.  
- After the source issue is corrected (e.g., missing columns fixed in the CSV or API), the quarantined records can be **retried** by re-ingesting them from the DLQ. This ensures no historical data is lost and the pipeline remains consistent.

### Value Range Validation Failures

**Scenario:** Numeric or datetime fields are outside acceptable ranges (e.g., `UnitPrice < 0`, `Quantity = 0`, `InvoiceDate` in the future).  

**Handling Flow:**  
- The record is **flagged for review**; some fields may be corrected automatically if simple rules apply (e.g., converting negative `UnitPrice` to zero if acceptable).  
- Uncorrectable records are sent to the **quarantine area**, with an error log capturing the failing field and its value.  
- Operators receive **real-time alerts** if the number of violations exceeds a threshold, helping to detect systematic data entry errors.  
- Quarantined records can be **manually corrected** or **reprocessed** after verification. For example, negative prices might be corrected based on historical averages or partner confirmation before reinsertion into the pipeline.

### Business Rule Validation Failures

**Scenario:** Cross-field inconsistencies or domain-specific rules fail (e.g., `InvoiceNo` starts with 'C' but `Quantity` is positive, duplicate invoice lines, or `Country` not recognized).  

**Handling Flow:**  
- The record is **partially rejected**; fields violating business rules are flagged, while other fields may continue to downstream transformation if safe.  
- The violating record is written to a **quarantine table** with metadata: which rule failed, record ID, and ingestion timestamp.  
- Operators are **alerted automatically** with dashboards showing the affected rules, frequency of violations, and affected batches.  
- Recovery involves **correcting the source or applying transformation fixes**. Once the issue is resolved, quarantined records can be **re-injected** into the pipeline, either individually or in batch mode, ensuring historical data integrity.

### General Error Handling Principles

- **Idempotent Operations:** All stages, including retries of failed records, are designed to be idempotent so reprocessing does not create duplicates.  
- **Checkpointing:** Long-running batch processes save progress periodically, allowing pipeline resumption from the last checkpoint if failure occurs.  
- **Monitoring & Alerting:** Alerts differentiate failure types (schema, value, business rules) and provide actionable details (record IDs, batch timestamps, number of failures).  
- **Auditing:** Every quarantined record retains full lineage metadata: source file, ingestion timestamp, transformation version, and failure reason. This ensures compliance and traceability.  
- **Recovery Workflow:** Operators can query the quarantine area, validate corrections, and re-submit affected records. Automated scripts can reprocess multiple batches while preserving ordering and feature calculations.

This design ensures that **all invalid or inconsistent records are captured, monitored, and recoverable**, minimizing data loss, improving pipeline reliability, and maintaining trust in downstream analytics and ML outputs.


# Task 3: Transformation and Storage Design
## Task 3.1 — Transformation and Storage Design

This section defines all transformations applied to validated data in the Online Retail pipeline. Each transformation specifies its input, output, and idempotency considerations.

### 1. Cleaning Operations

**Input:** Validated records from the validation stage, potentially still containing edge cases such as missing optional fields, minor duplicates, or inconsistent text formatting.  

**Output:** Cleaned dataset with remaining NaN values handled, duplicate rows removed, and standardized formats (e.g., trimming whitespace, consistent case in string columns).  

**Idempotency:** Safe to re-run. Cleaning operations overwrite or normalize existing fields without duplicating data, so reprocessing the same dataset produces the same result.

### 2. Derived Columns

**Input:** Cleaned transaction data.  

**Output:** Dataset enriched with additional columns:
- `line_total = Quantity * UnitPrice`
- `is_cancellation = InvoiceNo.startswith("C")`
- `date = InvoiceDate.date()`
- `hour = InvoiceDate.hour`  

**Idempotency:** Safe to re-run. Derived columns are deterministic calculations based on existing data, so multiple executions produce identical results.

### 3. Customer-Level Aggregations

**Input:** Transaction-level data with derived columns.  

**Output:** Aggregated metrics per customer:
- `total_revenue = sum(line_total)`
- `order_count = count(distinct InvoiceNo)`
- `product_diversity = count(distinct StockCode)`
- `recency = days since last transaction`  

**Idempotency:** Safe if aggregation keys are consistent. To ensure idempotency, sort and deduplicate transactions by unique invoice ID before aggregation. Re-running the aggregation produces the same summary values.

### 4. Feature Engineering for ML

**Input:** Aggregated customer data and recent transaction history within the observation window (e.g., last 90 days).  

**Output:** Feature table ready for ML model:
- `avg_spending_90d` — average spend in last 90 days
- `num_transactions_5d` — number of transactions in last 5 days
- `most_frequent_category` — most purchased product category
- `cancellation_rate` — fraction of orders that were canceled
- Any additional engineered features needed for the high-value customer prediction model  

**Idempotency:** Safe if observation window boundaries and feature calculations are deterministic. Use stable timestamps and consistent filtering to ensure that re-running the feature engineering step does not alter prior results.

### 5. Storage Layer Output

After all transformations, the data is stored in layered storage:
- **Clean Layer:** Stores the fully cleaned and validated transaction-level data (Parquet format, partitioned by date).  
- **Feature Layer:** Stores customer-level aggregated metrics and ML features (Parquet or feature store format).  

This design ensures that any stage can be re-run safely without corrupting downstream consumers or producing inconsistent results.


