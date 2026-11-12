# 02 — CTEs, Recursion, Joins, Subqueries, Set Operations, Grouping

## CTE
```sql
WITH a AS (SELECT ...), b AS (SELECT ... FROM a)
SELECT * FROM b;
WITH a AS MATERIALIZED (...)      -- force evaluate once
WITH a AS NOT MATERIALIZED (...)  -- inline into outer query
```

## Data-modifying CTE
```sql
WITH moved AS (DELETE FROM queue WHERE done RETURNING *)
INSERT INTO archive SELECT * FROM moved;

WITH upd AS (UPDATE t SET v = v + 1 WHERE id = 1 RETURNING id)
SELECT * FROM upd;
```

## Recursive CTE
```sql
WITH RECURSIVE tree AS (
  SELECT id, parent_id, name, 1 AS depth, ARRAY[id] AS path FROM node WHERE parent_id IS NULL
  UNION ALL
  SELECT n.id, n.parent_id, n.name, t.depth + 1, t.path || n.id
  FROM node n JOIN tree t ON n.parent_id = t.id
  WHERE NOT n.id = ANY(t.path)                      -- cycle guard
)
SELECT * FROM tree ORDER BY path;

-- built-in cycle / search clauses
WITH RECURSIVE t(id, parent) AS (...)
  SEARCH DEPTH FIRST BY id SET ordercol
  CYCLE id SET is_cycle USING path
SELECT * FROM t;
```

## Joins
```sql
FROM a INNER JOIN b ON a.id = b.a_id
FROM a LEFT  JOIN b ON ...        -- all of a
FROM a RIGHT JOIN b ON ...
FROM a FULL  JOIN b ON ...
FROM a CROSS JOIN b
FROM a JOIN b USING (id)
FROM a NATURAL JOIN b
FROM a JOIN b ON a.x = b.x AND a.y IS NOT DISTINCT FROM b.y
-- anti-join
SELECT a.* FROM a LEFT JOIN b ON a.id=b.a_id WHERE b.a_id IS NULL;
SELECT * FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.a_id = a.id);
-- semi-join
SELECT * FROM a WHERE EXISTS (SELECT 1 FROM b WHERE b.a_id = a.id);
-- self join
SELECT e.name, m.name AS manager FROM emp e LEFT JOIN emp m ON e.mgr = m.id;
```

## LATERAL
```sql
SELECT u.id, o.* FROM users u
CROSS JOIN LATERAL (SELECT * FROM orders WHERE user_id = u.id ORDER BY created DESC LIMIT 3) o;
SELECT u.id, x.val FROM users u LEFT JOIN LATERAL unnest(u.tags) AS x(val) ON true;
SELECT * FROM t, LATERAL jsonb_array_elements(t.data->'items') AS item;
```

## Subqueries
```sql
WHERE x IN (SELECT ...)        WHERE x NOT IN (SELECT ...)   -- beware NULLs; prefer NOT EXISTS
WHERE x = ANY (SELECT ...)     WHERE x > ALL (SELECT ...)
WHERE EXISTS (SELECT 1 ...)
SELECT (SELECT max(x) FROM u WHERE u.id = t.id) AS m FROM t;     -- scalar
FROM (SELECT ...) AS s                                           -- derived table
WHERE (a, b) IN (SELECT a, b FROM u)                             -- row compare
```

## Set operations
```sql
SELECT ... UNION SELECT ...             -- distinct
SELECT ... UNION ALL SELECT ...         -- keep dups (faster)
SELECT ... INTERSECT [ALL] SELECT ...
SELECT ... EXCEPT [ALL] SELECT ...
```

## Grouping
```sql
GROUP BY ROLLUP (a, b)          -- (a,b), (a), ()
GROUP BY CUBE (a, b)            -- all combinations
GROUP BY GROUPING SETS ((a), (b), ())
SELECT a, b, sum(x), grouping(a, b) FROM t GROUP BY ROLLUP (a, b);
```

## Ordered-set / hypothetical aggregates
```sql
percentile_cont(0.5) WITHIN GROUP (ORDER BY x)    percentile_disc(0.9) WITHIN GROUP (ORDER BY x)
mode() WITHIN GROUP (ORDER BY x)
rank(5) WITHIN GROUP (ORDER BY x)    dense_rank(5) WITHIN GROUP (ORDER BY x)
```

## Pivot (crosstab)
```sql
-- manual
SELECT id, sum(v) FILTER (WHERE k='a') AS a, sum(v) FILTER (WHERE k='b') AS b FROM t GROUP BY id;
-- tablefunc extension
CREATE EXTENSION tablefunc;
SELECT * FROM crosstab('SELECT id, k, v FROM t ORDER BY 1,2', 'SELECT DISTINCT k FROM t ORDER BY 1') AS ct(id int, a int, b int);
-- psql
SELECT ... \crosstabview
```

## Sampling / misc
```sql
SELECT * FROM t TABLESAMPLE BERNOULLI (1);          SELECT * FROM t TABLESAMPLE SYSTEM (1) REPEATABLE (42);
SELECT * FROM t ORDER BY random() LIMIT 10;
SELECT DISTINCT ON (g) * FROM t ORDER BY g, ts DESC;
SELECT * FROM t FETCH FIRST 10 ROWS WITH TIES;      -- needs ORDER BY
SELECT * FROM generate_series('2026-01-01'::date, '2026-12-31', interval '1 day') d;
SELECT * FROM t WHERE (a, b) > (1, 2);              -- keyset pagination
SELECT * FROM ROWS FROM (f1(), f2()) AS x(a, b);
```
