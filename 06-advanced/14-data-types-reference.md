# 14 — Data Types Reference

| Category | Types | Notes |
|---|---|---|
| Integer | `smallint`(2B) `integer`(4B) `bigint`(8B) | |
| Auto-increment | `smallserial` `serial` `bigserial`, **`GENERATED ... AS IDENTITY`** (preferred) | |
| Exact decimal | `numeric(p,s)` / `decimal` | money, exactness |
| Floating | `real`(4B) `double precision`(8B) | approximate |
| Money | `money` | locale-bound; prefer numeric |
| Text | `text` `varchar(n)` `char(n)` `"char"` `name` | `text` has no perf penalty |
| Binary | `bytea` | |
| Boolean | `boolean` | `true/false/null`, `'t' 'f' 'yes' 'no' 1 0` |
| Date/time | `date` `time` `timetz` `timestamp` `timestamptz` `interval` | **prefer timestamptz** |
| UUID | `uuid` | `gen_random_uuid()`, `uuidv7()` (18) |
| JSON | `json` `jsonb` `jsonpath` | prefer jsonb |
| XML | `xml` | |
| Array | `type[]` `type[][]` | |
| Range | `int4range int8range numrange tsrange tstzrange daterange` + multiranges | |
| Enum | `CREATE TYPE ... AS ENUM` | |
| Composite | `CREATE TYPE ... AS (...)` | table row types |
| Network | `inet` `cidr` `macaddr` `macaddr8` | |
| Geometric | `point line lseg box path polygon circle` | PostGIS for real GIS |
| Text search | `tsvector` `tsquery` | |
| Bit | `bit(n)` `bit varying(n)` | |
| Object IDs | `oid` `regclass` `regtype` `regproc` `regnamespace` `regrole` | |
| LSN / snapshot | `pg_lsn` `pg_snapshot` `txid_snapshot` | |
| Pseudo | `any anyelement anyarray anyenum anyrange record void trigger event_trigger cstring` | |
| Domain | `CREATE DOMAIN` | constrained base type |

## Typical choices
```sql
id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY     -- or uuid DEFAULT uuidv7()
price        numeric(12,2)
created_at   timestamptz NOT NULL DEFAULT now()
email        text  (+ UNIQUE index on lower(email), or citext)
status       text CHECK (status IN ('new','paid','void'))        -- or ENUM
flags/attrs  jsonb
tags         text[]
period       tstzrange
ip           inet
```

## Literals & casts
```sql
'text'   E'esc\n'   $$dollar quoted$$   $tag$...$tag$   U&'\0041'   B'101'   X'1F'
DATE '2026-11-11'   TIMESTAMPTZ '2026-11-11 14:30+05:30'   INTERVAL '1 day 2 hours'   INTERVAL '1' MONTH
'{1,2,3}'::int[]   ARRAY[1,2]   '{"a":1}'::jsonb   '192.168.0.1'::inet   '550e8400-e29b-41d4-a716-446655440000'::uuid
1e3   0x1F   1_000_000   0b101   0o17
```

## Limits (key)
```
Max database size: unlimited        Max table size: 32 TB (default 8kB blocks)
Max row size: ~1.6 TB (TOAST)       Max field size: 1 GB
Max columns/table: 1600             Max indexes/table: unlimited
Identifier length: 63 bytes         Max index columns: 32
bigint max: 9,223,372,036,854,775,807     int max: 2,147,483,647
```

## Date/time formatting
```
to_char(ts, 'YYYY-MM-DD HH24:MI:SS TZ')   'Dy, DD Mon YYYY'   'IYYY-IW' (ISO week)   'FMMonth'   'Q'   'Day'   'AM'   'MS' 'US'
DateStyle: SET datestyle = 'ISO, DMY';
IntervalStyle: SET intervalstyle = 'iso_8601';
```
