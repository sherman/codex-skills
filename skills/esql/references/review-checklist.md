# ES|QL Review Checklist

Use this checklist to review Elastic ES|QL queries. Keep the final answer focused on issues that can change correctness, cost, or alert behavior.

## Query Boundaries

- Split files on lines made mostly of `=`. Blank space around separators is not meaningful.
- A block may be a complete query, a fragment, or an annotated bad example. Do not treat `PROBLEM:` text as query text.
- For fragments, review only the visible behavior and state missing assumptions.

## Source Scope and Filtering

- Flag very broad `FROM` patterns when a narrower data stream, namespace, cluster, service, or metric family is available.
- Check for a time filter when the query is meant to run over a bounded window. If the user says the query is executed as an alert and the alert runner injects the time range outside the ES|QL text, treat the time bound as satisfied and do not report "missing time bound" as a finding. You may mention the external time-window assumption if it matters.
- A final `LIMIT` does not reduce scanned documents.
- Prefer filters before `STATS` when they are row predicates such as environment, service, metric name, cluster, namespace, or timestamp.
- Keep aggregate predicates after `STATS`, for example `WHERE c > 100`.
- Do not recommend moving a per-aggregation `WHERE` out of `STATS` when the query intentionally computes several conditional aggregates from the same input.

## Syntax and Version

- Verify SQL habits: ES|QL uses `WHERE` after `STATS` for aggregate filters, not SQL-style `HAVING`.
- `LIMIT N BY group` is valid ES|QL, but it should be preceded by a deterministic `SORT` when the query needs the top or latest row per group.
- Conditional aggregations use `STATS value = AGG(field) WHERE condition`.
- `TS` enables time-series aggregation functions inside `STATS`; `FROM` does not have the same time-series semantics.
- For any command or function that may be new or changed in 9.4/9.5, verify official docs before asserting compatibility.

## Aggregation Grain

- Identify the intended grain before judging `BY`: service, host, shard, namespace, cluster, time bucket, or full alert.
- Flag `STATS ... BY high-cardinality fields` such as trace IDs, user IDs, raw `@timestamp`, request IDs, or concatenated identifiers unless the user is intentionally listing individual events.
- For metrics, raw `@timestamp` often creates one group per sample. It may be valid for exact pivoting, but it is fragile when publishers have jitter.
- If pivoting metrics with conditional `STATS`, check whether all metrics are guaranteed to share the exact same timestamp and grouping dimensions. If not, recommend a time bucket or `TS` query.
- Question `MAX`, `MIN`, or `AVG` used as a duplicate resolver. `MAX` is only safe when the input contract guarantees one value per metric per group, or when the maximum is actually the intended statistic.

## Time-Series Metrics

- In this user's observability queries, metrics are generally emitted as deltas unless the query comments or field contract say otherwise. Do not flag `SUM(metric)` over a delta metric as a counter/gauge mistake. Still check duplicate ingestion, grouping grain, and whether the alert intentionally sums across all sources.
- Prefer `TS` for metric aggregations that need counter handling, per-time-series reduction, or fair aggregation across uneven publish intervals.
- For counters, raw sums or averages over counter values are usually wrong. Look for `RATE`, `INCREASE`, or another time-series function under `TS`.
- For gauges, confirm whether `MAX`, `MIN`, `AVG_OVER_TIME`, `LAST_OVER_TIME`, or percentile matches the alert's real intent.
- Check bucket size. Very small buckets can explode result sets; very large buckets can hide spikes or smear incidents.
- Check whether the query mixes metrics with different dimensional cardinality. Missing dimensions can turn calculations into nulls or misleading totals.

## Nulls, Missing Data, and Multivalues

- Treat blanket `COALESCE(metric, 0)` as risky. Missing may mean zero, data loss, failed pivot, or ingestion lag depending on the metric.
- Check denominators before division. Avoid divide-by-zero and low-sample alert noise.
- For percentages, make sure the zero case matches the real semantics. Example: if `good = 0` and `dropped > 0`, drop percentage should not become `0`.
- For string output, guard important fields with `COALESCE(TO_STRING(field), "<missing>")` when nulls would make alert details useless.
- Arithmetic and comparisons over multivalued fields can return null. Use `MV_*` functions, `MV_EXPAND`, or an explicit single-value contract.

## Arithmetic and Types

- Integer division rounds toward zero when both operands are integer types. Use `100.0`, `1048576.0`, `::DOUBLE`, or `TO_DOUBLE` when fractional math matters.
- Avoid awkward mixed types in `COALESCE` and `CASE`, especially when later doing floating-point math.
- Avoid rounding before threshold comparison unless the alert intentionally compares rounded values.
- Prefer named intermediate fields for complex formulas. This makes denominator handling and null behavior reviewable.
- Replace magic constants with named fields when they encode units, thresholds, or policy.

## Alert Determinism

- `LIMIT` without `SORT` returns arbitrary rows from the reviewer perspective. If the query alerts on top offenders, require a meaningful `SORT`.
- If more rows can match than the alert returns, make the selection deterministic and explain what is being sacrificed.
- Low sample sizes can produce noisy rates or percentages. Consider minimum request/event thresholds.
- Average can hide spikes. For incident detection, consider max, percentile, bucket-count, or sustained-threshold logic.
- Build expensive `details` strings after filtering to the rows that will be returned.

## Output and Maintainability

- Use `KEEP`/`DROP` for response shape and alert payload clarity, not as a claimed performance fix after aggregation.
- Name repeated complex expressions once with `EVAL`, then reuse the alias.
- Prefer meaningful field names: `request_rate`, `error_pct`, `period`, `cluster`, `namespace`, `host`.
- Comments should explain business meaning or data contracts, not claim false performance benefits.
- Preserve localized alert text from the original query unless the user asks to rewrite copy.
