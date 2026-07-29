# Apache Spark 4.2: The Biggest Features

## Collab
1. [Ankita Hatibaruah](https://github.com/Ahb98), [LinkedIn](http://linkedin.com/in/ankita-hatibaruah-bb2a62218)          
2. [Pavithra Ananthakrishnan](https://github.com/Pavi-245), [LinkedIn](https://www.linkedin.com/in/pavithra-ananthakrishnan-552416244/)          
3. [Sree Bhavya Kanduri](https://github.com/sreebhavya10), [LinkedIn](https://www.linkedin.com/in/kanduri-sree-bhavya-4001a6246)

## TLD;DR
Apache Spark has long been the go to engine for large scale data processing, analytics, and machine learning. With the release of Apache Spark 4.2, it takes a significant step toward becoming an AI-ready, real time analytics platform rather than just a distributed processing engine. The release introduces capabilities such as a native semantic layer for governed business metrics, automatic Change Data Capture (CDC) in Declarative Pipelines, real-time PySpark streaming, vector search and AI-native SQL functions, and several developer productivity improvements.

In this article, we'll explore the most impactful features introduced in Spark 4.2, understand the problems they solve, and walk through practical examples to see how they can simplify modern data engineering workloads. Whether you're building data warehouses, streaming pipelines, or AI applications, Spark 4.2 brings capabilities that make data processing faster, more consistent, and easier to manage.

## 1. Native Semantic Layer with Metric Views
### The Problem Before Spark 4.2: 
Imagine an e-commerce company. Every team needs the same business metrics.
For example:
- Revenue
- Active Users
- Average Revenue Per User (ARPU)
  
Unfortunately, every team calculates them differently.

BI Dashboard:
```sql
SUM(revenue) /
COUNT(DISTINCT user_id)
```
Finance Report:
```sql
SUM(revenue_amount) /
COUNT(DISTINCT customer_id)
```
Data Science Notebook:
```sql
df.groupBy(...).agg(...)
```
Ad-hoc SQL:
```sql
SELECT
SUM(total_sales)/COUNT(DISTINCT users)
```

Although all of these attempt to calculate ARPU, they may produce different answers because:
- Different column names
- Different filters
- Different business rules
- Different SQL implementations
  
Eventually people stop trusting the data. Instead of discussing business decisions,they spend hours discussing whose number is correct.

### The Spark 4.2 Solution: Spark introduces Metric Views.
Instead of defining metrics everywhere, define them once inside Spark.
Spark now understands business concepts such as:
- Dimensions
- Measures
- Metrics
- Business logic

as first-class objects.
This means Spark no longer only understands tables and columns. It also understands business meaning. 

Example: 
Suppose we have this sales table.
| date       | region | product | user_id | revenue |
|------------|--------|---------|--------:|--------:|
| 2025-01-01 | North  | Laptop  |     101 |   50000 |
| 2025-01-01 | North  | Mouse   |     102 |    1000 |
| 2025-01-01 | South  | Laptop  |     103 |   60000 |

Instead of every analyst writing calculations, we define one Metric View.

```sql
CREATE METRIC VIEW mv_business_metrics AS

SELECT
    date,
    region,
    product_category,

    SUM(revenue) AS revenue,

    COUNT(DISTINCT user_id) AS active_users,

    SUM(revenue) /
    COUNT(DISTINCT user_id) AS arpu

FROM fact_sales

GROUP BY
    date,
    region,
    product_category;
```

This becomes the official business definition. Now Everyone Uses the Same Metric

BI Dashboard:
```sql
SELECT
region,
arpu
FROM mv_business_metrics;
```

Finance Team:
```sql
SELECT
revenue,
active_users
FROM mv_business_metrics;
```

Data Science:
```sql
df = spark.table("mv_business_metrics")

df.filter("region='North'")
```

Reporting:
```sql
SELECT
date,
revenue
FROM mv_business_metrics
```

Nobody rewrites the metric anymore. Everyone queries the same semantic layer.
### Why This Matters:
- Single Source of Truth
- Easier Governance
- Reusable Business Logic
- Faster Analytics
- Lower Maintenance


## 2. Auto CDC in Declarative Pipelines
### The Problem Before Spark 4.2:
Most enterprise systems continuously receive data changes. These are called CDC (Change Data Capture) events.
For example:
- Operation	Meaning
- Insert	New customer
- Update	Customer changed address
- Delete	Customer removed
  
Traditionally, engineers manually write complex MERGE statements.

```sql
MERGE INTO customers t

USING customer_events s

ON t.id = s.id

WHEN MATCHED
THEN UPDATE ...

WHEN NOT MATCHED
THEN INSERT ...

WHEN MATCHED AND s.op='D'
THEN DELETE
```

This looks manageable until real-world challenges appear.
One must also handle:
- Updates
- Deletes
- Duplicate events
- Late-arriving events
- Out-of-order events
- Exactly-once processing
  
A simple MERGE quickly becomes hundreds of lines of logic.

Example CDC Events Incoming stream:
| event_ts | op | id | name     |
|----------|----|---:|----------|
| 10:01    | I  |  1 | Alice    |
| 10:02    | U  |  1 | Alice A. |
| 10:03    | D  |  1 | Alice    |
| 10:04    | I  |  2 | Bob      |
| 10:05    | U  |  2 | Bobby    |

Without automation, developers must manually determine:
- latest version
- delete handling
- ordering
- duplicate removal

### Spark 4.2 Auto CDC:
Instead of describing how to merge records, developers simply describe what the source is and which columns identify records. Spark handles everything else.
Example:
```python

(
spark.readStream
    .table("cdc.customer_events")
    .writeStream
    .option("pipelines.autoCDC.enabled", "true")
    .option("pipelines.cdc.type", "SCD_TYPE_1")
    .option("pipelines.cdc.keys", "id")
    .option("pipelines.cdc.sequence.column", "event_ts")
    .option("pipelines.cdc.operation.column", "op")
    .table("dim_customer")
)
```

Spark automatically performs:
- MERGE
- UPDATE
- DELETE
- INSERT
- Deduplication
- Event ordering
- Exactly-once guarantees

### What Happens Internally?
Incoming Events:
```bash
Insert Alice
Update Alice
Delete Alice
Insert Bob
Update Bob
```

Spark automatically produces:
| id | name  |
|---:|-------|
|  2 | Bobby |

Alice disappears because of the delete event. Bob becomes Bobby because Spark kept only the latest state. No custom MERGE logic required.


### Why Auto CDC Matters:
- Less Code
- Fewer Production Bugs
- Handles Out-of-Order Events
- Automatic Deduplication
- Exactly-Once Guarantees
- Lower Operational Cost

---

## 3. Standardized CDC using DSv2 & CHANGES Clause

### What is Change Data Capture (CDC)?
Change Data Capture (CDC) is a technique used to identify and process only the rows that have changed in a table instead of scanning the entire dataset.

### Typical changes include:

- ➕ Insert
- ✏️ Update
- ❌ Delete

This is especially useful for:

- Incremental ETL pipelines
- Data replication
- Real-time analytics
- Data synchronization

### Problem Before Spark 4.2

Prior to Spark 4.2, every storage format implemented CDC differently.

Developers often had to:

- Track Delta table versions manually.
- Compare timestamps or transaction logs.
- Write custom MERGE logic.
- Handle inserts, updates, and deletes separately.
- Build connector-specific CDC pipelines.
Because every storage system exposed CDC differently, pipelines were difficult to reuse across formats. Spark 4.2 introduces a standardized CDC interface through Data Source V2 to make CDC queries portable across connectors. 

### What Changed in Spark 4.2?
Spark 4.2 introduces a standard CDC interface through Data Source V2 (DSv2).

Instead of relying on connector-specific implementations, Spark provides a unified way to query row-level changes using the CHANGES clause.

The returned change stream contains metadata such as:

| Metadata column  | Description |
|------------------|-------------|
|  _change_type    | insert, delete, update_preimage, update_postimage |
|  _commit_version | Version in which the change occurred |
|  _commit_timestamp | Timestamp of the transaction |

### Example
```sql
Create a Delta Table
CREATE TABLE sales (
    id INT,
    product STRING,
    amount DOUBLE
)
USING DELTA;
```

Insert data
```
INSERT INTO sales VALUES
(1,'Laptop',1200),
(2,'Phone',700);
```
Update a row
```
UPDATE sales
SET amount = 1300
WHERE id = 1;
```
Delete a row
```
DELETE FROM sales
WHERE id = 2;
```
Query Changes
```sql
SELECT *
FROM sales
CHANGES BETWEEN VERSION 1 AND VERSION 3;
```
Example Output

| id  |	product	| amount	| _change_type	| _commit_version|
|----|----------|---------|---------------|----------------|
| 1	| Laptop	| 1200	| update_|preimage	| 2 |
| 1	| Laptop	| 1300	| update_|postimage |	2 |
| 2 |	Phone	| 700	| delete	| 3 |

### Understanding Update Events
An update produces two records.

Before Update
```
id=1 amount=1200
```
After Update
```
id=1 amount=1300
```
CDC Output
```
update_preimage
id=1 amount=1200

update_postimage
id=1 amount=1300
```
This allows downstream systems to know both the previous and new values.

### Benefits

- Standard CDC syntax across DSv2 connectors
- Eliminates custom version tracking
- Consistent handling of inserts, updates, and deletes
- Efficient incremental processing
- Simplifies ETL and replication pipelines
- Better interoperability across storage formats

### Use Cases

- Incremental ETL
- Data warehouse synchronization
- Audit logging
- Database replication
- Change history tracking
- Event-driven architectures

---

## 4. Real-Time Mode in PySpark

### Overview
Spark 4.2 extends Data Source V2 streaming capabilities, allowing PySpark Structured Streaming to consume change data continuously from supported connectors.

Instead of periodically scanning an entire table, streaming jobs process only newly committed changes, reducing latency and improving efficiency. Delta Lake 4.2 also adds new V2 streaming options such as startingVersion, startingTimestamp, and skipChangeCommits for catalog-managed tables. 

### Problem Before Spark 4.2
Building real-time CDC pipelines required significant engineering effort.

Common challenges included:

- Manual polling
- Version tracking
- Timestamp comparisons
- Custom checkpoint logic
- Separate handling of deletes
- Higher latency
- Complex recovery after failures

### What Changed?

PySpark Structured Streaming can continuously read CDC events using the Delta Change Data Feed.

Each micro-batch reads only newly committed changes.

The stream contains:

- Inserts
- Updates
- Deletes
Spark checkpoints ensure fault tolerance and exactly-once processing.

### Example
```sql
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("RealTimeCDC") \
    .getOrCreate()

changes_df = (
    spark.readStream
         .format("delta")
         .option("readChangeFeed", "true")
         .option("startingVersion", 0)
         .table("sales")
)

query = (
    changes_df.writeStream
              .format("console")
              .option("truncate", "false")
              .start()
)

query.awaitTermination()
```

### Example Stream Output
+---+--------+------+----------------+---------------+
|id |product |amount|_change_type    |_commit_version|
+---+--------+------+----------------+---------------+
|1  |Laptop  |1200  |insert          |1              |
|1  |Laptop  |1200  |update_preimage |2              |
|1  |Laptop  |1300  |update_postimage|2              |
|2  |Phone   |700   |delete          |3              |
+---+--------+------+----------------+---------------+

### Streaming Architecture

        Source System
              │
              ▼
        Delta Table
     (Change Data Feed)
              │
              ▼
    PySpark Structured Streaming
              │
              ▼
      Micro-Batch Processing
              │
      ┌───────┼────────┐
      ▼       ▼        ▼
 Dashboard   Kafka   Data Warehouse

### Benefits

- Low-latency change processing
- Exactly-once fault tolerance
- Supports inserts, updates, and deletes
- Reduced I/O by reading only changed rows
- Simple API for real-time pipelines
- Easily integrates with dashboards and downstream systems

### Real-World Use Cases

- Real-time dashboards
- Incremental ETL pipelines
- Fraud detection
- Live inventory tracking
- Customer activity monitoring
- Data synchronization across systems
- Event-driven applications

### Key Takeaways

|Feature | Before Spark 4.2 | Spark 4.2 |
|--------|------------------|-----------|
|CDC Query | Connector-specific | Standardized DSv2 API |
|Incremental Reads | Manual tracking | Native CHANGES support |
|Streaming CDC | Complex implementation | Built-in Structured Streaming support | 
|Delete Handling | Custom logic | Native support |
|Update Handling | Manual MERGE logic | update_preimage / update_postimage |
|Pipeline Development | High engineering effort | Simplified and standardized|

---

## 5. AI-Native SQL and Native Spatial Types

### The Problem Before Spark 4.2

Modern data platforms are increasingly combining analytics with Artificial Intelligence (AI) and geospatial data. Common use cases include:

- Semantic search
- Recommendation systems
- Retrieval-Augmented Generation (RAG)
- Location-based analytics

Before Spark 4.2, these workloads often required integrating Spark with external vector databases or GIS libraries. This increased infrastructure complexity, required additional data movement, and made applications harder to maintain.

### The Spark 4.2 Solution: AI-Native SQL and Native Spatial Types

Spark 4.2 introduces native support for vector search and geospatial data directly in Spark SQL.

### AI-Native SQL

Spark now introduces the **NEAREST BY** clause, allowing developers to perform vector similarity searches directly in SQL.

Suppose we have a table containing product embeddings.

| product_id | embedding |
|------------|-----------|
| 101 | [0.15, 0.62, 0.81] |
| 102 | [0.23, 0.74, 0.44] |
| 103 | [0.11, 0.60, 0.85] |

Finding products similar to a query vector becomes straightforward.

```sql
SELECT product_id
FROM products
NEAREST BY embedding
TO ARRAY(0.20, 0.70, 0.50)
LIMIT 2;
```

Instead of manually calculating similarity scores for every record, Spark automatically returns the nearest matching vectors.

### Native Spatial Types

Spark 4.2 also introduces first-class **GEOMETRY** and **GEOGRAPHY** data types together with built-in spatial functions.

Creating a geographic point is now simple.

```sql
SELECT ST_Point(77.5946, 12.9716);
```

A table can also directly store geographic information.

```sql
CREATE TABLE cities (
    city STRING,
    location GEOGRAPHY
);
```

Previously, these capabilities required external geospatial frameworks such as Apache Sedona or PostGIS. Spark now provides native support.

### Why This Matters

- Native vector search inside Spark SQL
- Easier AI and RAG application development
- Built-in geospatial data support
- Reduced dependency on external systems
- Simpler architecture and maintenance

---

## 6. Python Optimization (Arrow by Default)

### The Problem Before Spark 4.2

PySpark executes distributed processing in the JVM, while Python code runs in separate Python worker processes. Every time data moves between the JVM and Python, Spark must serialize and deserialize the data.

This communication overhead becomes significant for Python-heavy workloads involving Pandas, NumPy, and Python UDFs.

```
JVM
 ↓
Serialize
 ↓
Python Worker
 ↓
Deserialize
 ↓
Execute Python
 ↓
Serialize
 ↓
JVM
```

### The Spark 4.2 Solution: Arrow by Default

Spark 4.2 enables Apache Arrow by default for supported Python workloads.

Instead of exchanging individual objects, Spark transfers data as efficient columnar batches using Apache Arrow.

```
JVM
 ↓
Arrow Column Batch
 ↓
Python Worker
 ↓
Arrow Column Batch
 ↓
JVM
```

The Python code remains exactly the same.

```python
from pyspark.sql.functions import pandas_udf

@pandas_udf("double")
def discount(price):
    return price * 0.9

df.withColumn("discount_price", discount("price"))
```

Spark automatically uses Arrow whenever possible, significantly reducing serialization overhead and improving execution performance.

### Why This Matters

- Faster execution of Python workloads
- Reduced JVM-Python communication overhead
- Better interoperability with Pandas, NumPy, and Arrow-based libraries
- Improved performance for AI and machine learning pipelines
- No application code changes required

---

## 7. SQL Quality-of-Life Improvements

### The Problem Before Spark 4.2

Spark SQL has become increasingly powerful, but many common analytical queries still required verbose SQL or nested subqueries.

Examples include:

- Filtering results after window functions
- Creating fixed time buckets
- Managing SQL function namespaces

Although these limitations were small individually, they made SQL queries longer and more difficult to maintain.

### The Spark 4.2 Solution

Spark 4.2 introduces several quality-of-life improvements that make SQL cleaner and more expressive.

Some notable additions include:

- QUALIFY clause
- TIME_BUCKET function
- SET PATH support
- SQL cursors
- New aggregation functions such as MAX_BY() and MIN_BY()

### Example 1: QUALIFY

Before Spark 4.2:

```sql
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY department
               ORDER BY salary DESC
           ) AS rn
    FROM employees
)
WHERE rn = 1;
```

Spark 4.2:

```sql
SELECT *
FROM employees
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY department
    ORDER BY salary DESC
) = 1;
```

The `QUALIFY` clause removes the need for nested subqueries, making window-function queries cleaner and easier to read.

### Example 2: TIME_BUCKET

Grouping events into hourly windows becomes much simpler.

```sql
SELECT
    TIME_BUCKET(INTERVAL 1 HOUR, event_time) AS hour,
    COUNT(*) AS total_events
FROM logs
GROUP BY hour;
```

Instead of manually truncating timestamps, Spark now provides a built-in function for time-based bucketing.

### Why This Matters

- Cleaner SQL syntax
- Less boilerplate code
- Easier analytical queries
- Better readability and maintainability
- Improved developer productivity
