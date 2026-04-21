# Dynatrace Query Patterns

Use these patterns as starter shapes when the workflow needs concrete DQL, not as mandatory templates.

Adapt them to the exact entity ids, time window, data object, and field names for the investigation.

## Query Discipline

- Prefer exact entity filters over name-based matching once the entity id is known.
- Prefer exact field equality over `contains` when the field is known.
- Keep time windows narrow before searching logs, spans, or events.
- Change one variable at a time when a query returns no evidence:
  - field name
  - object type
  - time window
  - identifier variant

## Service-Level Metrics

Use when you need to compare a pre-window and post-window or establish whether a service actually regressed.

Pattern:

```text
timeseries {
  requests = sum(<request metric>),
  failures = sum(<failure metric>),
  latency = avg(<latency metric>)
}, by:{ dt.entity.service }
```

Use for:

- rollout comparison
- incident coarse signal
- deciding whether endpoint drill-down is justified

## Logs by Exact Identifier

Use when tracing a request id, order id, GUID, or correlation id through logs.

Pattern:

```text
fetch logs, from: <start>, to: <end>
| filter dt.entity.service == "<SERVICE-ID>"
| filter <field> == "<exact-id>"
| fields timestamp, dt.entity.service, <field>, content
| sort timestamp asc
```

If the field name is not known:

- discover the likely field first
- only then widen the query

## Spans or Traces by Identifier

Use when the identifier is expected to travel through distributed tracing.

Pattern:

```text
fetch spans, from: <start>, to: <end>
| filter dt.entity.service == "<SERVICE-ID>"
| filter <field> == "<exact-id>"
| fields timestamp, span.name, trace.id, <field>, duration
| sort timestamp asc
```

Use for:

- finding the first failing downstream hop
- walking a GUID through service-to-service calls

## Events or BI-Signal Search

Use when validating whether an expected event was emitted.

Pattern:

```text
fetch events, from: <start>, to: <end>
| filter dt.entity.service == "<SERVICE-ID>"
| filter <field> == "<exact-id>"
| fields timestamp, event.name, <field>
| sort timestamp asc
```

Before concluding absence:

- confirm the right object type
- confirm the right field name
- confirm the expected emission delay

## Top Failing Operations

Use when the service shows regression and you need to isolate the operation or endpoint driving it.

Pattern:

```text
fetch spans, from: <start>, to: <end>
| filter dt.entity.service == "<SERVICE-ID>"
| filter <error or failure predicate>
| summarize failures = count(), by:{ span.name }
| sort failures desc
```

Use for:

- rollout drill-down
- incident blast-radius reduction
- direct debugging of one failing path

## Chain-of-Custody Trace

Use when validating BI flow or downstream propagation.

Suggested record shape while investigating:

- system or entity
- data object
- field name
- exact identifier variant searched
- found or not found
- earliest and latest timestamps found

This is usually more reliable than trying to prove end-to-end presence or absence with one large query.
