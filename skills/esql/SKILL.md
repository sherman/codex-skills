---
name: esql
description: "Review Elastic ES|QL queries for Elasticsearch 9.4/9.5 compatibility, correctness, observability-metrics semantics, alert reliability, and query performance. Use when Codex is asked to inspect, critique, fix, or rewrite ES|QL pipelines, especially Elastic/Kibana alert queries or .esql files whose queries are separated by ===== lines."
---

# ES|QL Query Review

## Workflow

1. Identify the target Elasticsearch version. If the user says `9.4` or `9.5`, treat exact syntax and behavior as version-sensitive and verify against Elastic docs or a local Elastic checkout before making hard version claims.
2. Split `.esql` files on separator lines made mostly of `=` characters. Treat each block as a separate query or example. In example files, text after `PROBLEM:` or `Problem:` is annotation, not ES|QL.
3. Infer the query intent, source grain, time range, whether an alert runner injects the time window, metric type, grouping grain, and alert condition before judging the query.
4. Read `references/review-checklist.md` for any non-trivial review. Read `good-examples.esql` and `bad-examples.esql` when matching this repository's preferred review style or adding new examples.
5. Review in passes: syntax/version, source scope, filter placement, aggregation grain, metrics semantics, null and multivalue behavior, arithmetic, alert determinism, result limits, and output shape.
6. Prefer findings over generic advice. Every finding should name the risky fragment, explain why it matters, and propose a concrete fix or safer rewrite.

## Review Stance

- Be strict about correctness, false negatives, false positives, arbitrary alert rows, and obviously excessive shard scans.
- Be careful with inferred data contracts. If a problem depends on undocumented assumptions such as "one document per metric per timestamp", say that explicitly.
- Do not claim that a final `KEEP` or `DROP` improves aggregation cost. It mostly reduces response payload after upstream work has already happened.
- Do not rewrite the whole query unless the user asks. For review, preserve the original intent and give minimal corrected fragments.
- Treat `good-examples.esql` as style calibration: narrow sources where possible, filter early, name intermediate fields, use conditional `STATS`, use deterministic `SORT` before top-N `LIMIT`, and produce alert-friendly `details`.

## Output Format

Start with findings, ordered by severity. Use this shape:

```text
[severity] Short title
Fragment: `...`
Why: ...
Fix: ...
```

Use severities:

- `blocker`: invalid ES|QL, wrong version, or query cannot return intended result.
- `high`: likely false alert, missed alert, or severe avoidable cluster cost.
- `medium`: fragile metric semantics, cardinality risk, nondeterministic output, or maintainability issue that can affect operations.
- `low`: readability, naming, comments, magic constants, or minor style.

After findings, include a corrected fragment or full corrected query only when it materially helps. If there are no material issues, say so and list residual assumptions that could not be verified from the query alone.

## Version Notes

For exact 9.4/9.5 behavior, prefer current official Elastic documentation or a checked-out matching Elastic branch over memory. Useful starting points are in `references/elastic-docs.md`.
