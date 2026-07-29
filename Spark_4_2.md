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

## 3. Standardized CDC using DSv2 & CHANGES Clause

### Problem before Spark 4.2

Prior to Spark 4.2 and Delta Lake DSv2, implementing Change Data Capture (CDC) on Delta tables required complex custom logic:
- Consumers had to manually track versions/timestamps to detect changes using Delta's transaction log or file modification times.
- Change streams often involved custom merges, lookups, or incremental queries on `_change_type` metadata if available.
- Handling deletes and updates consistently was difficult and error-prone.
- CDC pipelines were non-standardized and required substantial engineering effort.
- Streaming reads didn’t have built-in CDC support, leading to inefficient incremental processing.

### What Changed in Spark 4.2 / DSv2?

Spark 4.2 introduced Delta Storage version 2 (DSv2), which includes native CDC support using a standardized **CHANGES** clause in SQL:
- You can directly query table changes between specific versions or timestamps using `CHANGES BETWEEN` or `CHANGES SINCE`.
- The change feed includes inserts, updates, and deletes in a consistent manner.
- Underlying transaction log enhancements provide atomic, reliable CDC streams.
- Removes the need for custom CDC logic or external bookkeeping.
- Enables both batch and streaming queries to consume change data easily and efficiently.

### CHANGES Clause Example

```sql
-- Get all changes since a timestamp
SELECT * FROM sales CHANGES SINCE '2023-05-01T00:00:00Z';

-- Or get changes between specific versions
SELECT * FROM sales CHANGES BETWEEN 15 AND 20;
```

### Benefits
- Simplifies CDC pipeline development with built-in support
- Supports efficient incremental ETL and data replication
- Handles inserts, updates, and deletes explicitly and consistently
- Enables real-time analytics with minimal latency

---

## 4. Real-Time Mode in PySpark

### Problem before Spark 4.2

- Real-time ingestion and CDC in PySpark were cumbersome without native streaming CDC support.
- Users had to rely on manual version tracking, checkpointing, or polling approaches to identify changes.
- Streaming CDC often missed deletes and relied on complex workarounds.
- Latency and reliability issues made real-time ETL difficult.
- Ensuring exactly-once semantics during incremental processing was challenging.

### What Changed in Spark 4.2 ?

With the introduction of DSv2 and native CDC streaming, PySpark supports real-time ingestion of changes using the
readChangeFeed option:
- PySpark structured streaming can now read CDC changes as a continuous stream.
- CDC includes inserts, updates, and deletes in a consistent format.
- Exactly-once fault-tolerant streaming pipelines can be built easily.
- Latency is reduced to micro-batch intervals.
- CDC streams can be used to build real-time dashboards, alerting, or ETL jobs with minimal engineering effort.

### Example: Streaming CDC in PySpark

from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("RealTimeCDC").getOrCreate()

'' Enable reading change feed from a Delta table with CDC enabled (DSv2)
changes_df = (
    spark.readStream
         .format("delta")
         .option("readChangeFeed", "true")  # Stream CDC changes
         .table("sales")
)

'' Write the streaming changes to console or any sink
query = (
    changes_df.writeStream
              .format("console")
              .option("truncate", "false")
              .start()
)

Query.awaitTermination()

### Benefits

- Continuous, low-latency, and fault-tolerant CDC ingestion
- Supports updates and deletes, not just inserts
- Simplifies building real-time ETL pipelines and dashboards
- Standardized API reduces custom logic and engineering overhead

### Use Cases:

- Incremental ETL pipelines
- Real-time dashboards and reporting
- Event-driven data architectures

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
