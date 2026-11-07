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

### Option A: single HTML file (no npm, Node or server)

Download `SQL-Interview-Practice.html` and double-click it to open it in Chrome or Edge. Everything, including the
SQL engine, is embedded in this one ~11 MB file. To regenerate it:

```bash
npm run build:single   # → dist-single/SQL-Interview-Practice.html
```

### Option B: development server

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

- **46 questions**: 11 Easy, 18 Medium, 17 Hard. They cover the patterns that come up most in data-engineering and
  analytics interviews: filtering and aggregation, HAVING, NULL handling, CASE, duplicates, self joins, anti joins,
  Nth highest, top-N per group, running totals, LAG/LEAD, gaps & islands, de-duplication, recursive CTEs,
  percentiles, pivots, cohort retention, moving averages and string aggregation.
- **"12 interview patterns" pack** (`src/questions_patterns.js`): dedup of a CDC feed with ties and NULL keys,
  month-to-date running totals, year-over-year growth, self-join pairs, pivot counts, clickstream sessionization,
  independent and strict (time-ordered) funnels, a retention cohort triangle, recursive org reach, a 7-day moving
  average over a date spine, and longest login streaks.
- **7 related dummy tables**:

  | table | rows | used for |
  | --- | --- | --- |
  | `employees` | 25 | org chart, salaries (ties, people earning more than their manager) |
  | `customers` | 25 | NULL and case-different duplicate emails |
  | `products` | 20 | 4 categories; some never sold |
  | `orders` | 48 | Jan–Jun 2024, NULL discounts, statuses, consecutive-day streaks |
  | `order_updates` | 24 | CDC feed with retried/duplicate versions, timestamp ties, NULL keys |
  | `web_events` | 40 | clickstream with exact-30-minute and 31-minute gaps, out-of-order funnel events |
  | `app_logins` | 32 | daily logins with same-day duplicates and month-crossing streaks |
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
`TOP (n)` → `LIMIT`, `DATEDIFF(day|minute|month, a, b)`, `timestampdiff`, `unix_timestamp`,
`sequence` / `GENERATE_SERIES` date spines, `OPTION (MAXRECURSION n)`, Spark `datediff(end, start)`, `DATEADD`, `date_add`, `DATETRUNC`,
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
src/data.js                 the tables (schema + rows)
src/questions.js            core question bank: prompt, hints, solutions (Spark + T-SQL), explanation, pitfalls
src/questions_patterns.js   "12 interview patterns" pack
scripts/build-single.mjs    builds the self-contained single HTML file
src/dialect.js      Spark SQL / T-SQL → DuckDB translation + dialect linter
src/grader.js       result comparison + feedback / error advice
src/db.js           DuckDB-WASM boot, seeding, query execution with timeout
src/main.js         UI
```

### Adding a question

Append an object to `QUESTIONS` in `src/questions.js`. The Spark solution is the reference the grader compares
against. Open `http://localhost:5173/?selftest=1` and run `window.__selftest` in the browser console. It checks
that both the Spark and the T-SQL solution produce the reference output and pass the linter.
