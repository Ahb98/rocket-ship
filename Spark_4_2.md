Spark 4.2

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

Example CDC Events: 
Incoming stream:
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
