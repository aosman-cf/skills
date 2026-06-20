# Pipelines Patterns

## Fire-and-Forget Producer

```typescript
export default {
  async fetch(req, env, ctx) {
    const event = { event_id: crypto.randomUUID(), event_type: "page_view", timestamp: new Date().toISOString() };
    ctx.waitUntil(env.MY_STREAM.send([event]));  // don't block the response
    return new Response("OK");
  }
};
```

## Client-Side Validation with Zod

Structured streams drop invalid events silently during processing. Validate before sending for immediate feedback.

```typescript
import { z } from "zod";

const EventSchema = z.object({
  event_id: z.string(),
  category: z.enum(["purchase", "view"]),
  amount: z.number().positive().optional(),
});

const validated = EventSchema.parse(rawEvent);  // throws synchronously
await env.MY_STREAM.send([validated]);
```

## Scheduled Collector Worker

```jsonc
// wrangler.jsonc
{
  "name": "collector",
  "pipelines": [{ "stream": "<STREAM_ID>", "binding": "EVENT_STREAM" }],
  "triggers": { "crons": ["*/5 * * * *"] }
}
```

```typescript
export default {
  async scheduled(event, env, ctx) {
    const items = await (await fetch("https://api.example.com/data")).json();
    const events = items.map(i => ({
      event_id: crypto.randomUUID(),
      timestamp: new Date().toISOString(),
      category: i.type,
      amount: i.value,
    }));
    await env.EVENT_STREAM.send(events);
  },
};
```

## Logpush → Pipelines

Pipelines is a native Logpush destination — ingest Cloudflare logs, transform with SQL, store as Iceberg/Parquet.

| Scope | Datasets |
|-------|----------|
| Zone | `http_requests`, `firewall_events`, `dns_logs` |
| Account | `workers_trace_events` |

```sql
INSERT INTO http_logs_sink
SELECT
  ClientIP,
  EdgeResponseStatus,
  to_timestamp_micros(EdgeStartTimestamp) AS event_time,
  upper(ClientRequestMethod) AS method,
  sha256(ClientIP) AS hashed_ip          -- redact PII at ingest
FROM http_logs_stream
WHERE EdgeResponseStatus >= 400;
```

Configure via Dashboard (**Logpush → Create a job → Pipelines** destination) or API.

## Pipelines + Queues Fan-out

```typescript
await Promise.all([
  env.ANALYTICS_STREAM.send([event]),  // long-term storage + SQL
  env.PROCESS_QUEUE.send(event),       // immediate processing + retries
]);
```

| Need | Use |
|------|-----|
| Long-term storage, SQL queries | Pipelines |
| Immediate processing, retries, DLQ | Queues |
| Both | Fan-out |

## Observability (GraphQL Analytics)

Same R2 API token works. Endpoint: `https://api.cloudflare.com/client/v4/graphql`.

```bash
curl -X POST "https://api.cloudflare.com/client/v4/graphql" \
  -H "Authorization: Bearer $API_TOKEN" -H "Content-Type: application/json" \
  -d '{"query": "query { viewer { accounts(filter: {accountTag: \"'$ACCOUNT_ID'\"}) { pipelinesIngestionAdaptiveGroups(filter: {pipelineId: \"PIPELINE-UUID-WITH-DASHES\", datetime_geq: \"2026-03-01T00:00:00Z\"}, limit: 10) { sum { ingestedRecords ingestedBytes } dimensions { datetimeHour } } } } }"}'
```

| Dataset | Shows | Sum fields |
|---------|-------|-----------|
| `pipelinesIngestionAdaptiveGroups` | Ingested into streams | `ingestedRecords`, `ingestedBytes` |
| `pipelinesOperatorAdaptiveGroups` | Processed, decode errors | `recordsIn`, `bytesIn`, `decodeErrors` |
| `pipelinesDeliveryAdaptiveGroups` | Delivered to sinks | `deliveredBytes` |
| `pipelinesSinkAdaptiveGroups` | Written to sinks | `recordsWritten`, `bytesWritten`, `filesWritten` |
| `pipelinesUserErrorsAdaptiveGroups` | Dropped events (validation) | `count` (by `errorType`) |
| `pipelinesUserErrorsAdaptive` | Detailed errors (24h) | — |

Error types: `missing_field`, `type_mismatch`, `parse_failure`, `null_value`.

> **Sink/pipeline IDs need dashes for GraphQL** but wrangler may show them without: `b909fe6e544844abbd63f6dcbc81d602` → `b909fe6e-5448-44ab-bd63-f6dcbc81d602`. Metrics take 5–10 min to populate.

### Detecting Silent Data Loss

If a sink's bucket is deleted or its token expires, events are accepted but lost. Tell-tale: `recordsWritten > 0` but `filesWritten = 0`. Always verify data lands in R2 within the roll interval and R2 SQL returns expected counts.

## Performance Tuning

| Goal | Config |
|------|--------|
| Low latency | `--roll-interval 10` (many small files) |
| Query performance | `--roll-interval 300` + automatic compaction |
| Cost optimal | `--compression zstd --roll-interval 300` |

## Schema Evolution (Immutable Pipelines)

Pipelines can't change. Use versioning + dual-write:

```bash
npx wrangler pipelines streams create events_v2 --schema-file v2.json
```
```typescript
await Promise.all([env.EVENTS_V1.send([event]), env.EVENTS_V2.send([event])]);
// query across versions with UNION ALL in R2 SQL
```

## End-to-End: Streaming Analytics Dashboard

```
External APIs → Collector Worker (cron) → Pipeline → R2 (Iceberg) → Dashboard Worker → R2 SQL
```

1. Create bucket + enable catalog ([r2-data-catalog](../r2-data-catalog/configuration.md))
2. Create stream + sink + pipeline (here)
3. Collector Worker with cron + stream binding (above)
4. Dashboard Worker querying R2 SQL ([r2-sql/patterns.md](../r2-sql/patterns.md))
5. Enable automatic compaction

## See Also

- [configuration.md](configuration.md) · [api.md](api.md) · [gotchas.md](gotchas.md)
- [r2-sql](../r2-sql/) — query ingested data
