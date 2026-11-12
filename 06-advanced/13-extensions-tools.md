# 13 — Extensions & Ecosystem Tools

## Contrib (ship with PostgreSQL: `CREATE EXTENSION name;`)
| Extension | Purpose |
|---|---|
| `pg_stat_statements` | query statistics |
| `auto_explain` | log slow plans (LOAD / preload) |
| `pg_trgm` | trigram / fuzzy / LIKE indexes |
| `btree_gin`, `btree_gist` | scalar types in GIN/GiST, exclusion constraints |
| `hstore` | key/value type |
| `citext` | case-insensitive text |
| `ltree` | hierarchical tree paths |
| `intarray` | fast int[] ops |
| `tablefunc` | crosstab / pivot, connectby |
| `pgcrypto` | hashing, encryption, gen_random_bytes |
| `uuid-ossp` | uuid_generate_v1/v4 |
| `unaccent` | strip accents for FTS |
| `fuzzystrmatch` | levenshtein, soundex, metaphone |
| `postgres_fdw`, `file_fdw`, `dblink` | remote/file access |
| `pg_buffercache` | shared buffer inspection |
| `pg_prewarm` | warm cache |
| `pgstattuple`, `pg_visibility`, `pageinspect`, `pg_freespacemap` | bloat / low-level inspection |
| `amcheck` | corruption checks |
| `pg_walinspect`, `pg_surgery` | WAL / repair tools |
| `pgrowlocks` | row lock info |
| `sslinfo`, `passwordcheck`, `auth_delay` | security helpers |
| `tsm_system_rows`, `tsm_system_time` | sampling methods |
| `cube`, `earthdistance` | geometry / distance |
| `isn`, `lo`, `seg`, `tcn`, `xml2`, `bloom`, `pg_overexplain` | misc |
```sql
SELECT * FROM pg_available_extensions ORDER BY name;
\dx
```

## Popular third-party extensions
| Extension | Purpose | Notes |
|---|---|---|
| **PostGIS** | geospatial | `CREATE EXTENSION postgis;` |
| **pgvector** | vector similarity / embeddings | `CREATE EXTENSION vector;` |
| **pg_cron** | in-DB cron scheduler | preload `pg_cron` |
| **pg_partman** | partition automation | see rollover file |
| **TimescaleDB** | time-series | hypertables, compression |
| **Citus** | distributed / sharding | |
| **pg_repack** | online bloat removal | `pg_repack -d db -t t` |
| **pgaudit** | audit logging | |
| **pg_hint_plan** | planner hints | |
| **hypopg** | hypothetical indexes | |
| **pg_stat_kcache**, **pg_qualstats**, **pg_wait_sampling** | deeper stats | |
| **wal2json**, **pgoutput**, **test_decoding** | logical decoding plugins | |
| **pglogical** | enhanced logical repl | |
| **pgcrypto / pg_jsonschema / pg_graphql / pgmq** | misc app features | |

### pgvector quick
```sql
CREATE EXTENSION vector;
CREATE TABLE items (id bigserial PRIMARY KEY, emb vector(1536));
CREATE INDEX ON items USING hnsw (emb vector_cosine_ops);          -- or ivfflat
SELECT id FROM items ORDER BY emb <=> '[0.1,0.2,...]' LIMIT 5;     -- <-> L2  <#> inner product  <=> cosine
SET hnsw.ef_search = 100;
```

### PostGIS quick
```sql
CREATE EXTENSION postgis;
CREATE TABLE places (id serial, geom geography(Point, 4326));
INSERT INTO places (geom) VALUES (ST_SetSRID(ST_MakePoint(77.59, 12.97), 4326));
CREATE INDEX ON places USING gist (geom);
SELECT * FROM places WHERE ST_DWithin(geom, ST_MakePoint(77.6, 12.9)::geography, 5000);
SELECT ST_Distance(a.geom, b.geom), ST_AsGeoJSON(geom), ST_Area(g), ST_Intersects(a,b), ST_Transform(g, 3857);
```

### pg_cron quick
```sql
SELECT cron.schedule('nightly-vacuum', '0 2 * * *', 'VACUUM ANALYZE');
SELECT cron.schedule_in_database('job', '*/5 * * * *', 'CALL f()', 'otherdb');
SELECT * FROM cron.job;  SELECT cron.unschedule(jobid);
```

### TimescaleDB quick
```sql
CREATE EXTENSION timescaledb;
SELECT create_hypertable('metrics', 'ts');
ALTER TABLE metrics SET (timescaledb.compress);   SELECT add_compression_policy('metrics', INTERVAL '7 days');
SELECT add_retention_policy('metrics', INTERVAL '90 days');
SELECT time_bucket('1 hour', ts) AS b, avg(v) FROM metrics GROUP BY b;
```

## Foreign data wrappers
`postgres_fdw`, `file_fdw`, `oracle_fdw`, `mysql_fdw`, `mongo_fdw`, `tds_fdw`, `redis_fdw`, `parquet_fdw`, `multicorn`
```sql
CREATE EXTENSION file_fdw;
CREATE SERVER files FOREIGN DATA WRAPPER file_fdw;
CREATE FOREIGN TABLE csv_t (a int, b text) SERVER files OPTIONS (filename '/data/f.csv', format 'csv', header 'true');
```

## dblink
```sql
CREATE EXTENSION dblink;
SELECT * FROM dblink('host=h dbname=d user=u password=p', 'SELECT id, name FROM t') AS r(id int, name text);
```

## Client / admin tools
| Tool | Use |
|---|---|
| psql, pgAdmin, DBeaver, DataGrip, TablePlus | clients |
| pgBouncer, pgpool-II, Odyssey | pooling |
| pgBackRest, Barman, WAL-G | backup |
| Patroni, repmgr, pg_auto_failover | HA |
| pgBadger | log analysis (`pgbadger postgresql.log -o report.html`) |
| pg_stat_monitor, pganalyze, Prometheus `postgres_exporter`, Grafana | monitoring |
| pgTAP | unit testing in SQL |
| Flyway, Liquibase, Alembic, Sqitch, golang-migrate, dbmate | schema migrations |
| pgloader | migrate from MySQL/SQLite/MSSQL |
| ora2pg | Oracle → PostgreSQL |
| pg_squeeze | auto bloat control |

## Drivers (connection strings)
```
postgresql://user:pass@host:5432/db?sslmode=require&application_name=app&connect_timeout=5
Python: psycopg (3) / asyncpg / SQLAlchemy      Node: pg / postgres / Prisma / Drizzle
Java: JDBC jdbc:postgresql://host:5432/db       Go: pgx / lib/pq       Rust: sqlx / tokio-postgres
.NET: Npgsql                                    Ruby: pg                PHP: pdo_pgsql
```
