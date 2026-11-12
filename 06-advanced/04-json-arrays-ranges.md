# 04 — JSON/JSONB, Arrays, Ranges, hstore, Composite

## JSONB access
```sql
data -> 'k'            -- jsonb
data ->> 'k'           -- text
data -> 0              -- array element
data #> '{a,b,0}'      -- path (jsonb)
data #>> '{a,b}'       -- path (text)
data['a']['b']         -- subscript (14+)
(data ->> 'age')::int
```

## JSONB operators
```sql
data @> '{"k":1}'                   -- contains
'{"k":1}' <@ data                   -- contained by
data ? 'k'                          -- key exists
data ?| ARRAY['a','b']              -- any key
data ?& ARRAY['a','b']              -- all keys
a || b                              -- concat/merge
data - 'k'                          -- delete key
data - ARRAY['a','b']
data - 3                            -- delete array idx
data #- '{a,b}'                     -- delete path
data @? '$.a[*] ? (@ > 1)'          -- jsonpath exists
data @@ '$.a > 1'                   -- jsonpath predicate
```

## JSON functions
```sql
jsonb_build_object('a', 1, 'b', 'x')     jsonb_build_array(1,2,3)       to_jsonb(row)     row_to_json(t)
jsonb_set(data, '{a,b}', '1', true)      jsonb_set_lax(...)             jsonb_insert(data, '{a,0}', '1')
jsonb_strip_nulls(data)                  jsonb_pretty(data)             jsonb_typeof(x)
jsonb_array_length(x)                    jsonb_object_keys(x)
jsonb_each(x)  jsonb_each_text(x)        jsonb_array_elements(x)  jsonb_array_elements_text(x)
jsonb_to_record(x) AS r(a int, b text)   jsonb_to_recordset(x) AS r(a int, b text)
jsonb_populate_record(NULL::t, x)        jsonb_agg(x)    jsonb_object_agg(k, v)
jsonb_path_query(data, '$.items[*] ? (@.price > 10)')
jsonb_path_query_array(...)  jsonb_path_query_first(...)  jsonb_path_exists(...)  jsonb_path_match(...)
JSON_TABLE(data, '$.items[*]' COLUMNS (id int PATH '$.id', name text PATH '$.name'))      -- 17+
JSON_VALUE(data, '$.a')  JSON_QUERY(data, '$.a')  JSON_EXISTS(data, '$.a')                 -- 17+
JSON_OBJECT('a': 1)  JSON_ARRAY(1,2)  JSON_OBJECTAGG(k: v)  JSON_ARRAYAGG(x)  x IS JSON
```
```sql
-- update nested
UPDATE t SET data = jsonb_set(data, '{profile,age}', '30');
UPDATE t SET data = data || jsonb_build_object('updated', now());
-- expand
SELECT t.id, e.key, e.value FROM t, jsonb_each(t.data) e;
-- query array of objects
SELECT * FROM t WHERE data->'tags' ? 'sql';
SELECT * FROM t WHERE data @> '{"items":[{"id":5}]}';
-- indexes
CREATE INDEX ON t USING gin (data);                       -- default ops: all keys/values
CREATE INDEX ON t USING gin (data jsonb_path_ops);        -- smaller, only @>
CREATE INDEX ON t ((data->>'email'));                     -- btree on extracted key
```

## Arrays
```sql
ARRAY[1,2,3]    '{1,2,3}'::int[]    ARRAY[[1,2],[3,4]]    int[3]  text[]
a[1]            -- 1-based
a[1:2]          -- slice
array_length(a, 1)  cardinality(a)  array_ndims(a)  array_dims(a)
array_append(a, 4)  array_prepend(0, a)  array_cat(a, b)  a || b  a || 4
array_remove(a, 2)  array_replace(a, 2, 9)  array_position(a, 3)  array_positions(a, 3)
array_to_string(a, ',')  string_to_array('a,b', ',')  array_fill(0, ARRAY[3])
unnest(a)  unnest(a) WITH ORDINALITY  array_agg(x)
a @> b  a <@ b  a && b  a = b  x = ANY(a)  x <> ALL(a)
array_sort(a) [18, verify]  array_reverse(a) [18, verify]
CREATE INDEX ON t USING gin (tags);
SELECT * FROM t WHERE tags @> ARRAY['sql'];     SELECT * FROM t WHERE tags && ARRAY['a','b'];
UPDATE t SET tags = array_append(tags, 'x') WHERE NOT 'x' = ANY(tags);
```

## Ranges & multiranges
```sql
int4range  int8range  numrange  tsrange  tstzrange  daterange   + int4multirange etc.
'[1,10)'::int4range      int4range(1,10,'[]')      daterange('2026-01-01','2026-02-01')
lower(r)  upper(r)  isempty(r)  lower_inc(r)  upper_inc(r)  range_merge(a,b)
r @> 5   r @> r2   r <@ r2   r && r2   r << r2   r >> r2   r -|- r2   r + r2   r * r2   r - r2
CREATE INDEX ON t USING gist (during);
-- no overlapping bookings
ALTER TABLE booking ADD EXCLUDE USING gist (room WITH =, during WITH &&);     -- needs btree_gist
-- temporal constraints [18+]
PRIMARY KEY (id, valid WITHOUT OVERLAPS)      FOREIGN KEY (id, PERIOD valid) REFERENCES p (id, PERIOD valid)
```

## hstore / composite / enum / domain
```sql
CREATE EXTENSION hstore;
'a=>1,b=>2'::hstore    h -> 'a'    h ? 'a'    h || 'c=>3'    hstore_to_jsonb(h)    each(h)    skeys(h)
CREATE TYPE addr AS (street text, city text);     SELECT (a).city FROM t;     SELECT ROW('x','y')::addr;
ALTER TYPE mood ADD VALUE 'meh';
```

## Geometry / network
```sql
point '(1,2)'   box   circle   polygon   a <-> b (distance)   a @> b
inet '192.168.1.1/24'   cidr   macaddr   i << '10.0.0.0/8'   host(i)   network(i)   masklen(i)   i + 1
-- PostGIS: CREATE EXTENSION postgis; geometry/geography types, ST_* functions, GiST indexes
```
