# R2 Data Catalog Configuration

Enable the catalog, create tokens, turn on automatic maintenance, and connect clients.

## Prerequisites

- Cloudflare account with an [R2 subscription](https://developers.cloudflare.com/r2/pricing/)
- An R2 bucket
- Node.js 16.17+ for wrangler

## Step 1: Create Bucket + Enable Catalog

```bash
npx wrangler r2 bucket create my-bucket
npx wrangler r2 bucket catalog enable my-bucket
```

`enable` outputs the two values you need everywhere else:

```
Warehouse: 4482a1cd43bf5197657ae1d8636c414a_my-bucket
Catalog URI: https://catalog.cloudflarestorage.com/4482a1cd43bf5197657ae1d8636c414a/my-bucket
```

- **Warehouse** = `{ACCOUNT_ID}_{BUCKET}` (bucket hyphens preserved)
- **Catalog URI** = `https://catalog.cloudflarestorage.com/{ACCOUNT_ID}/{BUCKET}`

Enabling creates `__r2_data_catalog/` metadata directories in the bucket; it does not modify existing objects.

## Step 2: Create an API Token

Dashboard → **R2** → **Manage R2 API tokens** → **Create API token**.

| Permission | Level | Enables |
|------------|-------|---------|
| **R2 Storage** | Admin Read & Write | Bucket + object access, PyIceberg/PySpark data reads/writes. **Admin Write is required even for read-only data access** (open-beta limitation). |
| **R2 Data Catalog** | Read & Write | Namespace/table creation, maintenance config |
| **R2 SQL** | Read | Querying via R2 SQL (add if you also query) |

**Simplest:** one token with Admin R&W (Storage) + R&W (Data Catalog), scoped to your bucket(s). This same token works for the Iceberg REST API, control-plane API, R2 SQL, and GraphQL Analytics.

Copy the token immediately — it is shown once. Token creation also yields S3-compatible **Access Key ID** / **Secret Access Key** (needed only for Spark orphan-file removal).

## Step 3: Enable Automatic Maintenance (Recommended)

R2 Data Catalog runs compaction and snapshot expiration for you.

```bash
# Compaction — merges small files (target size in MB; 64–512, default 128)
npx wrangler r2 bucket catalog compaction enable my-bucket \
  --target-size 128 --token $API_TOKEN

# Snapshot expiration — removes old snapshots AND their unreferenced data files
npx wrangler r2 bucket catalog snapshot-expiration enable my-bucket \
  --token $API_TOKEN --older-than-days 7 --retain-last 10
```

Compaction needs a **stored credential** to access files. `compaction enable` (and the dashboard setup wizard) stores it automatically; if configuring purely via the REST API, call the `/credential` endpoint (see [api.md](api.md)).

> Compaction triggers **hourly** with **no hard throughput cap** (the former 2 GB/hour limit has been lifted). Snapshot expiration deletes unreferenced data files automatically (since April 2026), so manual orphan-file cleanup is rarely needed.

**Target size guidance:**

| Workload | Target |
|----------|--------|
| Latency-sensitive queries | 64–128 MB |
| Streaming ingest | 128–256 MB |
| OLAP / large scans | 256–512 MB |

## Step 4: Verify

```bash
npx wrangler r2 bucket catalog status my-bucket
# or via control-plane API:
curl -s "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/r2-catalog/$BUCKET" \
  -H "Authorization: Bearer $API_TOKEN"
```

Expect `"status": "active"`, `compaction.state: "enabled"`, `credential_status: "present"`.

## Client Connection

### PyIceberg

```python
import os
from pyiceberg.catalog.rest import RestCatalog

catalog = RestCatalog(
    name="r2_catalog",
    warehouse=os.environ["R2_WAREHOUSE"],   # {ACCOUNT_ID}_{BUCKET}
    uri=os.environ["R2_CATALOG_URI"],       # https://catalog.cloudflarestorage.com/{ACCOUNT_ID}/{BUCKET}
    token=os.environ["R2_TOKEN"],
)
print(catalog.list_namespaces())            # connection test
```

### PySpark

See [patterns.md](patterns.md#pyspark-session) for the full Spark session config (requires Iceberg 1.6.1 and `X-Iceberg-Access-Delegation: vended-credentials`).

## Environment Variables Pattern

```bash
# .env (never commit)
R2_CATALOG_URI=https://catalog.cloudflarestorage.com/<ACCOUNT_ID>/<BUCKET>
R2_WAREHOUSE=<ACCOUNT_ID>_<BUCKET>
R2_TOKEN=<api-token>
```

## Disable Catalog

```bash
npx wrangler r2 bucket catalog disable my-bucket
```

Disabling preserves data and metadata; tables become inaccessible via the catalog until re-enabled.

## Security Best Practices

1. Store tokens in env vars / secret managers — never hardcode.
2. Least privilege: read-only tokens for query engines where possible (note open-beta requires Admin Write on Storage).
3. One token per application; rotate by creating new → testing → revoking old.
4. Scope tokens per bucket, not account-wide.

## See Also

- [api.md](api.md) — control-plane + PyIceberg API
- [gotchas.md](gotchas.md) — auth and maintenance troubleshooting
