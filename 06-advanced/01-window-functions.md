# 01 — Window Functions (complete)

## Syntax
```sql
function(args) [FILTER (WHERE cond)] OVER (
  [PARTITION BY expr, ...]
  [ORDER BY expr [ASC|DESC] [NULLS FIRST|LAST], ...]
  [frame_clause]
)
-- named window
SELECT ..., sum(x) OVER w FROM t WINDOW w AS (PARTITION BY g ORDER BY d);
-- inherit/extend a named window
SELECT sum(x) OVER (w ROWS 2 PRECEDING) FROM t WINDOW w AS (PARTITION BY g ORDER BY d);
```

## Frame clause
```sql
{ ROWS | RANGE | GROUPS } BETWEEN frame_start AND frame_end [ frame_exclusion ]
frame_start / frame_end:
  UNBOUNDED PRECEDING | n PRECEDING | CURRENT ROW | n FOLLOWING | UNBOUNDED FOLLOWING
frame_exclusion:
  EXCLUDE CURRENT ROW | EXCLUDE GROUP | EXCLUDE TIES | EXCLUDE NO OTHERS
```
- Default with ORDER BY: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
- Default without ORDER BY: whole partition
- `ROWS` = physical rows · `RANGE` = value range (peers tie) · `GROUPS` = peer groups
- `RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` (date/time ORDER BY)

## Ranking functions
```sql
row_number() OVER w          -- 1,2,3,4 (unique)
rank()       OVER w          -- 1,1,3,4 (gaps)
dense_rank() OVER w          -- 1,1,2,3 (no gaps)
percent_rank() OVER w        -- (rank-1)/(n-1)
cume_dist()  OVER w          -- rows <= current / total
ntile(4)     OVER w          -- bucket 1..4
```

## Value functions
```sql
lag(x [, offset [, default]])  OVER w     -- previous row
lead(x [, offset [, default]]) OVER w     -- next row
first_value(x) OVER w
last_value(x)  OVER (w ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)  -- needs full frame!
nth_value(x, n) OVER w
-- IGNORE NULLS / RESPECT NULLS: not available in 18 as far as I know (verify on the docs page for your version); use the fill-forward workaround in the patterns below
```

## Aggregates as window functions
```sql
sum(x) OVER (PARTITION BY g)                                       -- partition total on every row
sum(x) OVER (PARTITION BY g ORDER BY d)                            -- running total
sum(x) OVER (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)  -- 7-row moving sum
avg(x) OVER (ORDER BY d ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING)  -- centered moving avg
count(*) OVER ()                                                    -- grand total
count(*) FILTER (WHERE x > 0) OVER (PARTITION BY g)
min(x) OVER w, max(x) OVER w
array_agg(x) OVER (ORDER BY d ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
string_agg(x, ',') OVER (PARTITION BY g ORDER BY d)
```

## Ready-made patterns
```sql
-- Top N per group
SELECT * FROM (
  SELECT *, row_number() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn FROM emp
) s WHERE rn <= 3;

-- Running total & % of total
SELECT d, amt,
       sum(amt) OVER (ORDER BY d) AS running,
       round(100.0 * amt / sum(amt) OVER (), 2) AS pct_total
FROM sales;

-- Difference from previous row / growth %
SELECT d, v, v - lag(v) OVER (ORDER BY d) AS diff,
       round(100.0 * (v - lag(v) OVER (ORDER BY d)) / nullif(lag(v) OVER (ORDER BY d),0), 2) AS pct
FROM metrics;

-- Median-ish via percentile (ordered-set aggregates, not window)
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY x) FROM t;

-- Dedup keeping latest
DELETE FROM t WHERE id IN (
  SELECT id FROM (SELECT id, row_number() OVER (PARTITION BY email ORDER BY created DESC) rn FROM t) s WHERE rn > 1);

-- Gaps and islands
SELECT min(d) AS start, max(d) AS stop FROM (
  SELECT d, d - (row_number() OVER (ORDER BY d))::int AS grp FROM days
) s GROUP BY grp;

-- Sessionization (30-min gap)
SELECT *, sum(is_new) OVER (PARTITION BY user_id ORDER BY ts) AS session_id FROM (
  SELECT *, (ts - lag(ts) OVER (PARTITION BY user_id ORDER BY ts) > interval '30 min' OR lag(ts) OVER (PARTITION BY user_id ORDER BY ts) IS NULL)::int AS is_new FROM events
) s;

-- Fill NULLs forward (last non-null)
SELECT d, (array_agg(v) FILTER (WHERE v IS NOT NULL) OVER (ORDER BY d))[count(v) OVER (ORDER BY d)] AS filled FROM t;
-- or: first_value(v) OVER (PARTITION BY grp ORDER BY d) with grp = count(v) OVER (ORDER BY d)

-- Quartiles / buckets
SELECT *, ntile(4) OVER (ORDER BY score) AS quartile FROM t;

-- Rank within group + overall
SELECT *, rank() OVER (PARTITION BY dept ORDER BY sal DESC) AS dept_rank, rank() OVER (ORDER BY sal DESC) AS overall FROM emp;
```

## Rules
- Window functions run after WHERE/GROUP BY/HAVING; allowed only in SELECT and ORDER BY → filter them via subquery/CTE.
- `QUALIFY` does not exist in PostgreSQL; use subquery.
- Combine with `GROUP BY`: window runs over grouped output (`sum(sum(x)) OVER (...)`).
