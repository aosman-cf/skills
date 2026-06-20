# R2 Data Catalog API Reference

Two distinct APIs: the **control-plane REST API** (Cloudflare-specific catalog management) and the **Iceberg REST catalog API** (standard, used via PyIceberg/PySpark).

## Control-Plane REST API

Base: `https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/r2-catalog/{BUCKET}`
Auth: `Authorization: Bearer $API_TOKEN`

### Catalog Management

| Operation | Method | Path |
|-----------|--------|------|
| List catalogs | GET | `/r2-catalog` |
| Get catalog details | GET | `/r2-catalog/{bucket}` |
| Enable catalog | POST | `/r2-catalog/{bucket}/enable` |
| Disable catalog | POST | `/r2-catalog/{bucket}/disable` |
| Store compaction credential | POST | `/r2-catalog/{bucket}/credential` |

```bash
# Get catalog details (status, maintenance_config, credential_status)
curl -s "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/r2-catalog/$BUCKET" \
  -H "Authorization: Bearer $API_TOKEN"

# Store a token for compaction to use when reading/writing files
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/r2-catalog/$BUCKET/credential" \
  -H "Authorization: Bearer $API_TOKEN" -H "Content-Type: application/json" \
  -d '{"token": "'$API_TOKEN'"}'
```

Get-catalog response:
```json
{"result": {
  "id": "1192bf2c-...", "name": "<ACCOUNT_ID>_<BUCKET>", "bucket": "<BUCKET>",
  "status": "active",
  "maintenance_config": {
    "compaction": {"state": "enabled", "target_size_mb": "128"},
    "snapshot_expiration": {"state": "disabled", "min_snapshots_to_keep": 100, "max_snapshot_age": "7d"}
  },
  "credential_status": "present"
}, "success": true}
```

### Namespaces & Tables

| Operation | Method | Path |
|-----------|--------|------|
| List namespaces | GET | `/namespaces` |
| List tables in namespace | GET | `/namespaces/{ns}/tables` |
| **Get table metadata** | GET | `/namespaces/{ns}/tables/{table}` |

Query params on list endpoints: `?return_uuids=true`, `?return_details=true`, `?parent={ns}` (child namespaces), `?page_size=N&page_token=...`.

**Nested namespaces use `%1F` (Unit Separator), not `/` or `.`:** `/namespaces/parent%1Fchild/tables`.

```bash
curl -s "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/r2-catalog/$BUCKET/namespaces" \
  -H "Authorization: Bearer $API_TOKEN"
# {"result": {"namespaces": [["live"]]}, "success": true}
```

### Get Table (metadata introspection)

`GET /namespaces/{ns}/tables/{table}` returns schema, partition spec, sort order, and snapshot info. Equivalent to Iceberg "load table" but on the control plane, with snapshots pruned to the most recent 10.

```bash
curl -s "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/r2-catalog/$BUCKET/namespaces/live/tables/earthquakes" \
  -H "Authorization: Bearer $API_TOKEN"
```

Response shape:
```json
{"result": {
  "identifier": {"namespace": ["live"], "name": "earthquakes"},
  "table_uuid": "019edccf-3ac8-73e3-...",
  "metadata_location": "s3://live-data/__r2_data_catalog/.../metadata/01225-....metadata.json",
  "total_snapshots": 1225,
  "returned_snapshots": 10,
  "metadata": {
    "format-version": 2,
    "current-schema-id": 1,
    "schemas": [ /* Iceberg schema(s) */ ],
    "partition-specs": [ /* ... */ ],
    "sort-orders": [ /* ... */ ],
    "properties": {"write.parquet.compression-codec": "zstd"},
    "current-snapshot-id": 3055729675574597004,
    "snapshots": [ /* most recent 10 */ ],
    "snapshot-log": [ /* ... */ ],
    "refs": {"main": {"snapshot-id": 3055729675574597004, "type": "branch"}}
  }
}, "success": true}
```

| Field | Type | Description |
|-------|------|-------------|
| `identifier` | object | `{namespace: [...], name}` |
| `table_uuid` | UUID | Iceberg table UUID |
| `metadata_location` | string? | R2 path to current metadata file |
| `total_snapshots` | int | Total snapshots before pruning |
| `returned_snapshots` | int | Count in `metadata.snapshots` (max 10) |
| `metadata` | object | Standard [Iceberg TableMetadata](https://iceberg.apache.org/spec/#table-metadata-fields), snapshots/snapshot-log/metadata-log pruned to 10 |

Auth scope: same `Read` scope as list-tables.

### Maintenance Configuration

| Operation | Method | Path |
|-----------|--------|------|
| Get catalog-level config | GET | `/maintenance-configs` |
| Update catalog-level config | POST | `/maintenance-configs` |
| Get table-level config | GET | `/namespaces/{ns}/tables/{table}/maintenance-configs` |
| Update table-level config | POST | `/namespaces/{ns}/tables/{table}/maintenance-configs` |

```bash
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/r2-catalog/$BUCKET/maintenance-configs" \
  -H "Authorization: Bearer $API_TOKEN" -H "Content-Type: application/json" \
  -d '{
    "compaction": {"state": "enabled", "target_size_mb": "256"},
    "snapshot_expiration": {"state": "enabled", "min_snapshots_to_keep": 10, "max_snapshot_age": "7d"}
  }'
```

All fields optional. Table-level config overrides catalog-level.

### Error Format

```json
{"success": false, "errors": [{"code": 10000, "message": "Authentication error"}]}
```

| HTTP | Meaning |
|------|---------|
| 400 | Bad request / invalid params |
| 401 | Authentication failed |
| 403 | Insufficient permissions |
| 404 | Resource not found / catalog not enabled |
| 409 | Conflict (e.g. already enabled/exists) |

## Iceberg REST Catalog API

Standard [Apache Iceberg REST Catalog](https://github.com/apache/iceberg/blob/main/open-api/rest-catalog-open-api.yaml). Base: `https://catalog.cloudflarestorage.com/{ACCOUNT_ID}/{BUCKET}`. Most users go through PyIceberg/PySpark rather than raw REST.

The `/config` route requires `?warehouse={WAREHOUSE}`:
```bash
curl -s "https://catalog.cloudflarestorage.com/$ACCOUNT_ID/$BUCKET/v1/config?warehouse=${ACCOUNT_ID}_${BUCKET}" \
  -H "Authorization: Bearer $API_TOKEN"
```

### PyIceberg Client

```python
from pyiceberg.catalog.rest import RestCatalog

catalog = RestCatalog(name="r2", warehouse=WAREHOUSE, uri=CATALOG_URI, token=TOKEN)
```

| Task | Code |
|------|------|
| List namespaces | `catalog.list_namespaces()` |
| Create namespace | `catalog.create_namespace_if_not_exists("logs")` |
| List tables | `catalog.list_tables("logs")` |
| Create table | `catalog.create_table(("logs", "events"), schema=schema)` |
| Load table | `catalog.load_table(("logs", "events"))` |
| Append | `table.append(pyarrow_table)` |
| Overwrite | `table.overwrite(pyarrow_table)` |
| Scan | `table.scan(row_filter="id > 100").to_pandas()` |
| Rename | `catalog.rename_table(("ns","old"), ("ns","new"))` |

```python
from pyiceberg.schema import Schema
from pyiceberg.types import NestedField, StringType, LongType

schema = Schema(
    NestedField(1, "id", LongType(), required=True),
    NestedField(2, "name", StringType(), required=False),
)
table = catalog.create_table(("logs", "events"), schema=schema)
```

### Schema Evolution

```python
with table.update_schema() as update:
    update.add_column("user_id", LongType(), doc="User ID")  # add as nullable
    update.rename_column("msg", "message")
    update.update_column("id", field_type=LongType())        # widening only (int→long)
```

### Time-Travel

```python
scan = table.scan(snapshot_id=table.snapshots()[-2].snapshot_id)
# or as-of timestamp (ms)
yesterday_ms = int((datetime.now() - timedelta(days=1)).timestamp() * 1000)
scan = table.scan(as_of_timestamp=yesterday_ms)
```

## Maintenance (Manual, via PySpark)

Automatic maintenance (configured via control-plane API/wrangler) is preferred. For manual control or very large tables, use Spark procedures:

```python
spark.sql("CALL r2dc.system.rewrite_data_files(table => 'ns.tbl')")
spark.sql("CALL r2dc.system.rewrite_manifests(table => 'ns.tbl')")
spark.sql("CALL r2dc.system.expire_snapshots(table => 'ns.tbl', older_than => TIMESTAMP '2026-02-01 00:00:00')")
# Orphan removal REQUIRES S3 credentials (vended creds fail with NoAuthWithAWSException)
spark.sql("CALL r2dc.system.remove_orphan_files(table => 'ns.tbl', older_than => TIMESTAMP '2026-02-28 00:00:00')")
```

PyIceberg equivalents (`table.rewrite_data_files(...)`, `table.expire_snapshots(...)`) work for smaller tables; use Spark for >1 TB.

## See Also

- [configuration.md](configuration.md) — enabling catalog, tokens, automatic maintenance
- [patterns.md](patterns.md) — PyIceberg/PySpark workflows
- [gotchas.md](gotchas.md) — error troubleshooting
