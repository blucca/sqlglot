# WHERE-only CASE WHEN membership unnesting

Focused on [SQLGlot #9190](https://github.com/tobymao/sqlglot/issues/9190), reported by joeharris76, after georgesittas’s [NULL-preserving #8990](https://github.com/tobymao/sqlglot/pull/8990).

Base: `eb11aec4be8cb2122afe713a4816662c3f0fb2fa`. This branch widens the uncorrelated membership guard only for a positive condition of a searched `CASE WHEN` under `WHERE`. Positive AND/OR and parentheses preserve the condition’s TRUE set; FALSE and UNKNOWN both take ELSE. Direct `= ANY` uses the same path; other ANY comparisons keep their existing guard.

The branch follows the condition’s actual `If.this` child, keeping THEN/ELSE values, simple CASE, NOT, COALESCE, IS NULL and comparisons to FALSE protected. SELECT/HAVING/ORDER and ON retain the previous behavior. Projection/group support requires post-aggregation handling; ON hoisting must preserve ELSE rows.

## Native checks

- SQLite 3.45.1 and DuckDB 2.0.0: **144 row-multiset/AST checks** over 18 shapes and mixed-NULL, NULL-only, empty and non-NULL subqueries, preserving duplicate outer rows.
- DuckDB: **8 additional ANY checks**, with equality unnested and greater-than preserved.
- Added `TestOptimizer.test_unnest_case_when_where`: 64 condition/subquery cases and 3 protected grouped/aggregate/ON shapes.
- Pure-Python upstream suite: **1727 tests, 61 skips, exit 0** with `SKIP_INTEGRATION=1`. Actual compiled-extension execution was outside this run.

A broader local prototype placed `SUM(a.id)` in JOIN ON (SQLite/DuckDB errors) and a join marker in an ungrouped projection (DuckDB error). Removing the CASE-in-ON guard reduced a 12-row fixture to 2 rows. Those results motivated the WHERE-only change.

QuestDB **10.2.0 / bundled JDK 25.0.2** was also exercised on 13 synthetic LONG input rows. The supplied HAVING form fails before CASE evaluation (`Invalid column: HAVING`); its legacy and candidate forms also fail parsing. The reporter’s QuestDB build and preceding HAVING lowering are the next native acceptance inputs.

On this branch in an existing SQLGlot development environment:

```sh
python -m unittest tests.test_optimizer.TestOptimizer.test_unnest_case_when_where
SKIP_INTEGRATION=1 python -m unittest
```

Implemented and tested by Codex under the blucca account owner’s authorization. Upstream MIT license and attribution are retained.
