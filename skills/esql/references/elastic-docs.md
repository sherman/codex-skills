# Elastic Docs Links

Use official Elastic docs for exact version and syntax checks. If the user needs a hard 9.4/9.5 statement, verify live docs or a matching local Elasticsearch branch before answering.

## Starting Points

- ES|QL reference: https://www.elastic.co/docs/reference/query-languages/esql
- Commands: https://www.elastic.co/docs/reference/query-languages/esql/commands
- Functions and operators: https://www.elastic.co/docs/reference/query-languages/esql/functions-operators
- Release notes: https://www.elastic.co/docs/release-notes/elasticsearch

## High-Value Pages for Reviews

- `STATS`: https://www.elastic.co/docs/reference/query-languages/esql/commands/stats-by
- `LIMIT`: https://www.elastic.co/docs/reference/query-languages/esql/commands/limit
- `SORT`: https://www.elastic.co/docs/reference/query-languages/esql/commands/sort
- `TS`: https://www.elastic.co/docs/reference/query-languages/esql/commands/ts
- Operators: https://www.elastic.co/docs/reference/query-languages/esql/functions-operators/operators
- Time-series aggregation functions: https://www.elastic.co/docs/reference/query-languages/esql/functions-operators/time-series-functions

## Facts Worth Rechecking

- `STATS` supports per-aggregation `WHERE`; this is not equivalent to filtering rows before `STATS`.
- `LIMIT N BY group` keeps up to N rows per group; use a preceding `SORT` when order matters.
- ES|QL result limits cap output rows, not the number of source documents scanned.
- `TS` is preferred for metric time-series aggregation because it handles time-series functions and avoids common raw-metric pitfalls.
- Integer division truncates when both operands are integer types; use a double operand or cast for fractional division.
- Many scalar operators return null on multivalued inputs.
