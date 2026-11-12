# 07 — Indexes (advanced)

## Index types — when to use
| Type | Use for | Operators |
|---|---|---|
| **btree** (default) | equality, range, ORDER BY, uniqueness | `< <= = >= > BETWEEN IN`, `LIKE 'abc%'` (C collation / text_pattern_ops) |
| **hash** | equality only | `=` |
| **gin** | arrays, jsonb, tsvector, trigram | `@> <@ && ? @@` |
| **gist** | ranges, geometry, nearest-neighbor, exclusion | `&& @> <@ <<  <->` |
| **spgist** | partitioned search (phone, IP, text prefix) | |
| **brin** | huge append-only, naturally ordered (time) tables | tiny index |
| **bloom** (ext) | many-column equality | |

## Create variants
```sql
CREATE INDEX CONCURRENTLY idx ON t (a);                       -- no write lock; can't run in tx block
CREATE UNIQUE INDEX ON t (lower(email));
CREATE INDEX ON t (a DESC NULLS LAST, b);
CREATE INDEX ON t (a, b) INCLUDE (c, d);                      -- index-only scans
CREATE INDEX ON t (a) WHERE deleted_at IS NULL;               -- partial
CREATE INDEX ON t ((a + b));                                  -- expression
CREATE INDEX ON t (date_trunc('day', ts));
CREATE INDEX ON t USING brin (ts) WITH (pages_per_range = 32);
CREATE INDEX ON t USING gin (jsonb_col jsonb_path_ops);
CREATE INDEX ON t USING gin (to_tsvector('english', body));
CREATE INDEX ON t USING gin (name gin_trgm_ops);
CREATE INDEX ON t USING gist (geom);
CREATE INDEX ON t USING hash (token);
CREATE INDEX ON t (name text_pattern_ops);                    -- LIKE 'x%' in non-C locale
CREATE INDEX ON t (a) WITH (fillfactor = 90, deduplicate_items = on);
CREATE UNIQUE INDEX ON t (a) NULLS NOT DISTINCT;
CREATE INDEX ON t (a) TABLESPACE fast;
```

## Rebuild / maintain
```sql
REINDEX INDEX CONCURRENTLY idx;
REINDEX TABLE CONCURRENTLY t;
-- manual concurrent swap
CREATE INDEX CONCURRENTLY idx_new ON t (a);
DROP INDEX CONCURRENTLY idx_old;
ALTER INDEX idx_new RENAME TO idx_old;
-- invalid leftovers from failed CONCURRENTLY
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
-- unique constraint from index
ALTER TABLE t ADD CONSTRAINT uq UNIQUE USING INDEX idx;
```

## Diagnose
```sql
-- unused
SELECT s.indexrelname, s.idx_scan, pg_size_pretty(pg_relation_size(s.indexrelid))
FROM pg_stat_user_indexes s JOIN pg_index i USING (indexrelid) WHERE s.idx_scan = 0 AND NOT i.indisunique ORDER BY pg_relation_size(s.indexrelid) DESC;
-- duplicate
SELECT indrelid::regclass, array_agg(indexrelid::regclass) FROM pg_index GROUP BY indrelid, indkey HAVING count(*) > 1;
-- FK columns without index
SELECT c.conrelid::regclass, c.conname FROM pg_constraint c
WHERE c.contype='f' AND NOT EXISTS (SELECT 1 FROM pg_index i WHERE i.indrelid = c.conrelid AND (i.indkey::int2[])[0:cardinality(c.conkey)-1] @> c.conkey);
-- size & bloat check
SELECT pg_size_pretty(pg_relation_size('idx'));
CREATE EXTENSION pgstattuple;   SELECT * FROM pgstatindex('idx');
-- is index used?
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
-- hypothetical test
SET enable_seqscan = off;
```

## Index-selection rules of thumb
- Put equality columns first, then range column: `(tenant_id, status, created_at)`.
- Multi-column btree serves leading-column prefixes only.
- Covering: `INCLUDE` non-searched output columns → index-only scan (needs recent VACUUM for visibility map).
- Low-selectivity boolean → use **partial** index instead.
- `LIKE '%x%'` → trigram GIN, not btree.
- Time-ordered giant tables → BRIN.
- Each index slows writes; drop unused.
- Statistics: `ANALYZE t;` after bulk load; `ALTER TABLE t ALTER COLUMN c SET STATISTICS 500;`
