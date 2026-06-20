# R2 SQL Configuration

## Prerequisites

- R2 bucket with Data Catalog enabled
- R2 API token with the right permissions
- Wrangler CLI (for CLI queries)

## Enable R2 Data Catalog

R2 SQL queries Iceberg tables in R2 Data Catalog — enable it on the bucket first.

```bash
npx wrangler r2 bucket catalog enable my-bucket
```

Output gives the **Warehouse** (`{ACCOUNT_ID}_{BUCKET}`) and **Catalog URI**. You query by **warehouse** name. See [r2-data-catalog/configuration.md](../r2-data-catalog/configuration.md) for full setup.

## Create an API Token

R2 SQL needs **R2 Storage Admin Read & Write** (which includes R2 SQL Read) — or, where available, add **R2 SQL Read** explicitly.

Dashboard → **R2** → **Manage R2 API tokens** → **Create API token** → **Admin Read & Write**. Copy the token (shown once).

| Permission | Grants |
|------------|--------|
| R2 Storage Admin Read & Write | Storage ops + R2 SQL queries + Data Catalog ops |
| R2 SQL Read | SQL queries only |

> Open-beta limitation: R2 Storage **Admin Read & Write is required even for read-only R2 SQL queries**. May change at GA.

## Configure Auth

### Wrangler CLI

```bash
export WRANGLER_R2_SQL_AUTH_TOKEN=<your-token>
# or a .env file in the project dir (auto-loaded):
# WRANGLER_R2_SQL_AUTH_TOKEN=<your-token>
```

> Wrangler does **not** use the OAuth session from `wrangler login` for R2 SQL — the env var is required.

### REST API

```bash
curl -X POST \
  "https://api.sql.cloudflarestorage.com/api/v1/accounts/$ACCOUNT_ID/r2-sql/query/$BUCKET" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT * FROM default.my_table LIMIT 10"}'
```

## Verify Setup

```bash
npx wrangler r2 sql query "${ACCOUNT_ID}_my-bucket" "SHOW DATABASES"
npx wrangler r2 sql query "${ACCOUNT_ID}_my-bucket" "SHOW TABLES IN default"
```

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| Token authentication failed | Missing/invalid token | Set `WRANGLER_R2_SQL_AUTH_TOKEN`; token needs Admin R&W |
| Catalog not enabled on bucket | Catalog off | `wrangler r2 bucket catalog enable <bucket>` |
| Permission denied | Insufficient perms | Use Admin Read & Write token |
| Table not found | Wrong warehouse/namespace | `SHOW DATABASES`, `SHOW TABLES IN <ns>` |

## See Also

- [r2-data-catalog/configuration.md](../r2-data-catalog/configuration.md) — token + catalog detail
- [api.md](api.md) — SQL syntax · [patterns.md](patterns.md) — query examples
