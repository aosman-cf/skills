# Pipelines Gotchas

## Critical Issues

### Events accepted but never appear (most common)

HTTP 200 / `send()` resolves, but no data in the sink.

**Causes:**
1. **Schema validation failure** — structured streams accept then **silently drop** invalid events during processing.
2. **First-flush warm-up** — first data takes **3–7 minutes** (pipeline warm-up + namespace/table creation) even with `--roll-interval 10`.
3. **Roll interval not elapsed** — default 300s.
4. **Silent sink failure** — deleted bucket or expired token. Check `recordsWritten > 0` but `filesWritten = 0` in sink metrics.

**Fixes:** validate client-side (Zod); monitor `pipelinesUserErrorsAdaptiveGroups`; poll for ≥5 minutes in tests; verify sink `failure_reason` via `GET /pipelines/{id}`.

### Everything is immutable

Cannot modify stream schema, pipeline SQL, or sink config. Delete and recreate. Use version naming (`events_v1`) and keep SQL in version control.

```bash
curl -X DELETE "$BASE_URL/pipelines/{id}" -H "Authorization: Bearer $API_TOKEN"
curl -X DELETE "$BASE_URL/sinks/{id}"     -H "Authorization: Bearer $API_TOKEN"
curl -X DELETE "$BASE_URL/streams/{id}"   -H "Authorization: Bearer $API_TOKEN"
```

### Worker binding undefined (`env.MY_STREAM`)

1. Use the **stream ID**, not pipeline ID, in `wrangler.jsonc`.
2. Binding field is `"stream"` (June 2026); old `"pipeline"` still works.
3. Redeploy after adding the binding.

### REST API field names ≠ CLI flags

| REST | CLI | |
|------|-----|--|
| `"type": "r2_data_catalog"` | `--type r2-data-catalog` | underscores vs hyphens |
| `"table_name"` | `--table` | |
| `"token"` | `--catalog-token` | |
| `"format": {"type":"parquet"}` | implied | required in REST |

### `wrangler pipelines delete` defaults to "no"

Non-interactive environments answer "no" automatically — use REST `DELETE` for CI/automation.

## Behavioral Notes

- **`__ingest_ts` auto-added** (TIMESTAMP, day-partitioned). Don't put it in your schema.
- **Sinks can't target existing tables** — the sink creates its own. Use PySpark to write to existing tables.
- **JSON-only input** — no Avro/Protobuf/CSV.
- **Naming:** streams/sinks/pipelines use underscores; buckets use hyphens.
- **Metrics lag 5–10 min** after creation. Don't panic on empty early metrics.

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Events not in R2 | Roll interval / first-flush warm-up | Wait 3–7 min, then per `roll-interval` |
| Validation drops | Type mismatch / missing field | Validate client-side; check GraphQL errors |
| 429 | >5 MB/s per stream | Batch; request increase |
| 413 | >1 MB request | Split batches |
| Can't delete stream | Pipeline references it | Delete pipeline first |
| Sink credential error | Token expired/revoked | Recreate sink with valid token |
| Pipeline `failed` | See `failure_reason` | Fix token/bucket/catalog, recreate |

## Pipeline SQL Limitations

- Row-level transforms only — **no GROUP BY / aggregation / window functions** (do aggregation in [R2 SQL](../r2-sql/) at query time).
- CTEs (`WITH`) and `UNNEST` are supported.
- No schema evolution (immutable).

## Limits (Open Beta)

| Resource | Limit |
|----------|-------|
| Streams / Sinks / Pipelines per account | 20 each |
| Payload per request | 1 MB |
| Ingest rate per stream | 5 MB/s |
| Recommended batch size | ~100 events |

## Debug Checklist

- [ ] Stream exists: `wrangler pipelines streams list`
- [ ] Pipeline `running` (not `initializing`/`failed`): `GET /pipelines/{id}`, check `failure_reason`
- [ ] SQL matches schema; sink token valid; bucket + catalog exist
- [ ] Worker redeployed after binding; binding uses **stream ID** under `"stream"`
- [ ] Waited ≥5 min (first flush)
- [ ] Sink metrics: `filesWritten > 0`; error metrics show no drops

## See Also

- [configuration.md](configuration.md) · [api.md](api.md) · [patterns.md](patterns.md)
- [Pipelines docs](https://developers.cloudflare.com/pipelines/)
