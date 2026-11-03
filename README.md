# SQL Interview Sandbox: Databricks & Microsoft Fabric

A browser-based practice environment for SQL interview questions, focused on the two dialects you meet on
Databricks and Microsoft Fabric:

| Dialect toggle | Platforms |
| --- | --- |
| **Spark SQL** | Databricks SQL / notebooks, Fabric Lakehouse (Spark) |
| **T-SQL** | Fabric Warehouse, Lakehouse SQL analytics endpoint |

Queries run locally in your browser on DuckDB-WASM. No server, account, or internet connection is needed once it
is built.

## Run it

```bash
npm install
npm run dev        # http://localhost:5173
```

Production build (static files in `dist/`, can be served by any static web server):

```bash
npm run build
npm run preview    # http://localhost:4173
```

> Open it over http(s), not `file://`: the SQL engine runs in a Web Worker.

## Features

- **34 questions**: 11 Easy, 13 Medium, 10 Hard. They cover the patterns that come up most in data-engineering and
  analytics interviews: filtering and aggregation, HAVING, NULL handling, CASE, duplicates, self joins, anti joins,
  Nth highest, top-N per group, running totals, LAG/LEAD, gaps & islands, de-duplication, recursive CTEs,
  percentiles, pivots, cohort retention, moving averages and string aggregation.
- **4 related dummy tables** (`employees` 25 rows, `customers` 25, `products` 20, `orders` 48). They contain
  deliberate traps: NULL discounts and emails, case-different duplicate customers, salary ties, a customer
  who earns more than their manager, consecutive-day order streaks, and products whose only orders were cancelled.
- **Hint button**: progressive hints, revealed one at a time.
- **Solution button**: the reference solution in both Spark SQL and T-SQL, with an explanation and
  Databricks-vs-Fabric dialect notes.
- **Graded submissions** with targeted feedback when you're wrong:
  - shape problems (wrong number or order of columns, wrong row count, duplicate rows from JOIN fan-out)
  - which columns are right and which are wrong, with a concrete example row
  - numeric diagnosis: fraction vs percentage, wrong sign (datediff argument order), integer truncation,
    rounding, values consistently too low (NULL arithmetic) or too high (missing filter)
  - question-specific mistake detectors (e.g. `= NULL`, filtering a LEFT JOIN in WHERE, ROW_NUMBER dropping ties,
    PARTITION BY a NULLable key, missing `COALESCE(discount, 0)`)
  - friendly explanations of engine errors (unknown column with "did you mean", GROUP BY errors, aggregates in
    WHERE, window functions in WHERE, ambiguous columns, syntax errors, timeouts)
- **Dialect linter**: flags syntax that would fail on the selected platform, for example `LIMIT`/`QUALIFY`/`LEFT ANTI JOIN`
  in T-SQL, or `TOP`/`ISNULL(a,b)`/`GETDATE()` in Spark SQL. A correct result written in the wrong dialect is not
  counted as solved.
- **Free playground**, a schema explorer (click to preview a table or insert a column), progress saved in
  `localStorage`, and a **Reset data** button for DML/DDL experiments.

## How the dialect emulation works

`src/dialect.js` rewrites common dialect-specific syntax into DuckDB SQL before running it. Examples:
`TOP (n)` → `LIMIT`, `DATEDIFF(day, a, b)`, Spark `datediff(end, start)`, `DATEADD`, `date_add`, `DATETRUNC`,
`ISNULL`, `NVL`, `IIF`, `CONVERT`, `STRING_AGG ... WITHIN GROUP`, `PERCENTILE_CONT ... WITHIN GROUP ... OVER`,
`collect_set`, `array_join`, `LEFT ANTI/SEMI JOIN`, `[brackets]`, backticks, Spark `"double-quoted strings"`, and
T-SQL recursive CTEs without `RECURSIVE`. When T-SQL is selected, the engine also uses integer division for
`int / int`, so you can see the classic truncation bug happen.

Known gaps (the explanations call these out where they matter):

- T-SQL `AVG(int_column)` returns an integer on Fabric. The sandbox returns a decimal.
- T-SQL string concatenation with `+` is not emulated (use `CONCAT`).
- `TOP ... WITH TIES`, `TOP ... PERCENT`, `@variables` and `#temp` tables are not supported.
- The sandbox's SQL engine is more permissive than either platform. The linter catches the common cases, but not
  every platform-specific rule.

## Project layout

```
index.html          app shell
src/data.js         the 4 tables (schema + rows)
src/questions.js    question bank: prompt, hints, solutions (Spark + T-SQL), explanation, pitfalls
src/dialect.js      Spark SQL / T-SQL → DuckDB translation + dialect linter
src/grader.js       result comparison + feedback / error advice
src/db.js           DuckDB-WASM boot, seeding, query execution with timeout
src/main.js         UI
```

### Adding a question

Append an object to `QUESTIONS` in `src/questions.js`. The Spark solution is the reference the grader compares
against. Open `http://localhost:5173/?selftest=1` and run `window.__selftest` in the browser console. It checks
that both the Spark and the T-SQL solution produce the reference output and pass the linter.
