# R2 Data Catalog Patterns

Practical workflows with PyIceberg (lightweight, no JVM) and PySpark (full Iceberg ecosystem).

## Choosing PyIceberg vs PySpark

| Need | Tool |
|------|------|
| Catalog ops, append/scan, small-medium loads | PyIceberg |
| Batch ETL, INSERT INTO SELECT, DELETE/MERGE, write-back, >1 TB maintenance | PySpark |
| Pure SQL analytics (no writes) | [R2 SQL](../r2-sql/) |

## PyIceberg Connection

```python
import os, pyarrow as pa
from pyiceberg.catalog.rest import RestCatalog

catalog = RestCatalog(
    name="r2",
    warehouse=os.environ["R2_WAREHOUSE"],   # {ACCOUNT_ID}_{BUCKET}
    uri=os.environ["R2_CATALOG_URI"],       # https://catalog.cloudflarestorage.com/{ACCOUNT_ID}/{BUCKET}
    token=os.environ["R2_TOKEN"],
)
catalog.create_namespace_if_not_exists("analytics")
```

## Pattern: Create + Load (PyIceberg)

```python
schema = pa.schema([
    ("id", pa.int64()),
    ("name", pa.string()),
    ("amount", pa.float64()),
])
table = catalog.create_table(("analytics", "events"), schema=schema)

data = pa.table({"id": [1, 2, 3], "name": ["a", "b", "c"], "amount": [80.0, 92.5, 88.0]})
table.append(data)
print(table.scan().to_arrow().to_pandas())
```

## Pattern: Partitioned Time-Series Table (PyIceberg)

```python
from pyiceberg.schema import Schema
from pyiceberg.types import NestedField, TimestampType, StringType
from pyiceberg.partitioning import PartitionSpec, PartitionField
from pyiceberg.transforms import DayTransform

schema = Schema(
    NestedField(1, "timestamp", TimestampType(), required=True),
    NestedField(2, "level", StringType(), required=True),
    NestedField(3, "message", StringType(), required=False),
)
spec = PartitionSpec(PartitionField(source_id=1, field_id=1000, transform=DayTransform(), name="day"))
table = catalog.create_table(("logs", "app_logs"), schema=schema, partition_spec=spec)

# Partition pruning on read
errors = table.scan(row_filter="level = 'ERROR'").to_pandas()
```

## PySpark Session

Requires Iceberg **1.6.1** and vended credentials. S3 keys are only needed for orphan-file removal.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("R2DataCatalog") \
    .config('spark.jars.packages',
        'org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.6.1,'
        'org.apache.iceberg:iceberg-aws-bundle:1.6.1,'
        'org.apache.hadoop:hadoop-aws:3.3.4,'
        'com.amazonaws:aws-java-sdk-bundle:1.12.262') \
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
    .config("spark.sql.catalog.r2dc", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.r2dc.type", "rest") \
    .config("spark.sql.catalog.r2dc.uri", CATALOG_URI) \
    .config("spark.sql.catalog.r2dc.warehouse", WAREHOUSE) \
    .config("spark.sql.catalog.r2dc.token", TOKEN) \
    .config("spark.sql.catalog.r2dc.header.X-Iceberg-Access-Delegation", "vended-credentials") \
    .config("spark.sql.catalog.r2dc.s3.remote-signing-enabled", "false") \
    .config("spark.sql.defaultCatalog", "r2dc") \
    .config("spark.hadoop.fs.s3a.access.key", S3_ACCESS_KEY) \
    .config("spark.hadoop.fs.s3a.secret.key", S3_SECRET_KEY) \
    .config("spark.hadoop.fs.s3a.endpoint", S3_ENDPOINT) \
    .config("spark.hadoop.fs.s3a.path.style.access", "true") \
    .getOrCreate()
spark.sql("USE r2dc")
```

> `X-Iceberg-Access-Delegation: vended-credentials` is required and `s3.remote-signing-enabled` must be `false`. First startup takes ~30–60s for JAR downloads (cached after).

## Pattern: Batch ETL (PySpark)

```python
# Create partitioned table
spark.sql("""
CREATE TABLE IF NOT EXISTS my_ns.events (
    __ingest_ts TIMESTAMP, event_id STRING, category STRING, amount DOUBLE
) PARTITIONED BY (days(__ingest_ts))
""")

# Load from CSV / Parquet
spark.read.option("header","true").csv("data.csv").writeTo("my_ns.events").append()
spark.read.parquet("data.parquet").writeTo("my_ns.events").append()

# Transform between tables
spark.sql("INSERT INTO my_ns.target SELECT col1, col2 FROM my_ns.source WHERE col1 > 0")

# Delete / overwrite
spark.sql("DELETE FROM my_ns.events WHERE amount < 0")
spark.sql("INSERT OVERWRITE my_ns.events SELECT * FROM my_ns.staging")
```

> **Partition large tables** (`PARTITIONED BY (days(__ingest_ts))` or similar). Unpartitioned tables work for small datasets (<1000 files) but degrade at scale.

## Pattern: Inspect Metadata Tables (PySpark)

```python
spark.sql("SELECT * FROM my_ns.events.snapshots").show()
spark.sql("SELECT * FROM my_ns.events.files").show()
spark.sql("SELECT * FROM my_ns.events.history").show()
```

## Pattern: Concurrent Writes with Retry

```python
from pyiceberg.exceptions import CommitFailedException
import time

def append_with_retry(table, data, max_retries=3):
    for attempt in range(max_retries):
        try:
            table.append(data); return
        except CommitFailedException:
            if attempt == max_retries - 1: raise
            time.sleep(2 ** attempt)
```

Optimistic locking: concurrent commits to the same table may conflict. Writes to different partitions are safe.

## Pattern: DuckDB over Catalog Data

```python
import duckdb
arrow = catalog.load_table(("logs", "app_logs")).scan().to_arrow()
con = duckdb.connect(); con.register("logs", arrow)
con.execute("SELECT level, COUNT(*) FROM logs GROUP BY level").fetchdf()
```

## Connecting Any Iceberg Engine

Snowflake, Trino, Spark, DuckDB, etc. connect with the **Iceberg REST catalog** config:

- Catalog URI: `https://catalog.cloudflarestorage.com/{ACCOUNT_ID}/{BUCKET}`
- Warehouse: `{ACCOUNT_ID}_{BUCKET}`
- Token: your R2 API token
- Header: `X-Iceberg-Access-Delegation: vended-credentials`

## Best Practices

| Area | Guidance |
|------|----------|
| Partitioning | Day/hour for time-series; 100–1000 partitions; avoid high-cardinality keys (user_id) |
| File sizes | Target 128–512 MB; rely on automatic compaction |
| Schema | Add columns nullable (`required=False`); only widen types |
| Maintenance | Enable automatic compaction + snapshot expiration (see [configuration.md](configuration.md)) |
| Reads | Filter on partition columns; select only needed columns; batch appends ~100 MB+ |

## See Also

- [api.md](api.md) — API details · [gotchas.md](gotchas.md) — troubleshooting
- [pipelines/patterns.md](../pipelines/patterns.md) — streaming ingest
- [r2-sql/patterns.md](../r2-sql/patterns.md) — SQL analytics
