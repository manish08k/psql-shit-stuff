# 03 — Built-in Functions & Operators Reference

## Strings
```sql
length(s)  char_length(s)  octet_length(s)  lower(s)  upper(s)  initcap(s)
trim(s)  ltrim(s)  rtrim(s)  trim(BOTH 'x' FROM s)  btrim(s, 'x')
substring(s, 2, 3)  substring(s FROM 2 FOR 3)  substring(s FROM '[0-9]+')  left(s, n)  right(s, n)
position('x' IN s)  strpos(s, 'x')  replace(s, 'a', 'b')  translate(s, 'abc', 'xyz')
concat(a, b)  concat_ws(',', a, b)  a || b  repeat(s, n)  reverse(s)
lpad(s, 10, '0')  rpad(s, 10, '.')  split_part(s, ',', 2)  string_to_array(s, ',')  string_to_table(s, ',')
regexp_replace(s, 'pat', 'rep', 'g')  regexp_match(s, 'pat')  regexp_matches(s, 'pat', 'g')  regexp_split_to_array(s, ',')  regexp_count(s, 'pat')  regexp_like(s,'pat')
format('%s %I %L', a, ident, literal)  quote_ident(s)  quote_literal(s)  quote_nullable(s)
md5(s)  sha256(bytea)  encode(b, 'hex'|'base64')  decode(s, 'hex')  convert_to(s,'UTF8')
to_char(n, '999,999.99')  to_char(ts, 'YYYY-MM-DD HH24:MI:SS')  to_number('1,234', '9,999')
ascii('A')  chr(65)  starts_with(s, 'x')  s LIKE 'a%'  s ILIKE '%a%'  s ~ 're'  s ~* 're'  s SIMILAR TO ...
similarity(a,b)  a % b   -- pg_trgm
```

## Numbers
```sql
abs  ceil  floor  round(x, 2)  trunc(x, 2)  mod(a,b)  a % b  power(a,b)  a ^ b  sqrt  cbrt  exp  ln  log(x)  log10  sign
div(a,b)  gcd  lcm  random()  setseed(0.5)  greatest(a,b,..)  least(a,b,..)  width_bucket(x, lo, hi, n)
pi()  sin cos tan degrees radians  min_scale  trim_scale
generate_series(1,10)  generate_series(1,10,2)
x::numeric(10,2)  x::int
```

## Date / time
```sql
now()  current_timestamp  current_date  current_time  localtimestamp  clock_timestamp()
date_trunc('month', ts)  date_trunc('day', ts, 'Asia/Kolkata')
extract(year FROM ts)  extract(epoch FROM ts)  extract(dow|doy|week|quarter|hour|minute FROM ts)  date_part('month', ts)
ts + interval '1 day'  ts - interval '2 hours'  d + 7  d2 - d1 (days)  age(ts1, ts2)  age(ts)
make_date(y,m,d)  make_timestamp(...)  make_timestamptz(...)  make_interval(days => 3)
to_timestamp(epoch)  to_timestamp('2026-11-11','YYYY-MM-DD')  to_date('11/11/2026','DD/MM/YYYY')
ts AT TIME ZONE 'Asia/Kolkata'  SET TIME ZONE 'UTC'
justify_days(i)  justify_hours(i)  justify_interval(i)  isfinite(ts)
(a, b) OVERLAPS (c, d)
date_bin('15 minutes', ts, TIMESTAMP '2026-01-01')
generate_series(ts1, ts2, interval '1 hour')
last day of month: (date_trunc('month', d) + interval '1 month - 1 day')::date
```

## Conditionals / null handling
```sql
CASE WHEN c THEN a WHEN c2 THEN b ELSE z END      CASE x WHEN 1 THEN 'a' END
COALESCE(a, b, c)   NULLIF(a, b)   GREATEST   LEAST
a IS NULL  a IS NOT NULL  a IS DISTINCT FROM b  a IS NOT DISTINCT FROM b
a BETWEEN x AND y   a NOT BETWEEN SYMMETRIC x AND y
a IN (1,2,3)  a = ANY(ARRAY[..])  a <> ALL(ARRAY[..])
x IS TRUE / IS NOT TRUE / IS UNKNOWN
```

## Aggregates
```sql
count(*)  count(x)  count(DISTINCT x)  sum  avg  min  max
bool_and  bool_or  every
string_agg(x, ',' ORDER BY x)  array_agg(x ORDER BY y)  json_agg  jsonb_agg  json_object_agg  jsonb_object_agg
stddev  stddev_pop  variance  var_pop  corr(y,x)  covar_pop  regr_slope(y,x)  regr_intercept  regr_r2
bit_and  bit_or  bit_xor
any_value(x)
agg(x) FILTER (WHERE cond)
agg(DISTINCT x ORDER BY x)
percentile_cont / percentile_disc / mode   (WITHIN GROUP)
```

## Type conversion
```sql
x::type   CAST(x AS type)   to_char / to_number / to_date / to_timestamp
'5'::int  '2026-01-01'::date  'a'::text  '{1,2}'::int[]  '{"a":1}'::jsonb  'x'::regclass  123::oid::regclass
pg_typeof(x)
```

## UUID / misc
```sql
gen_random_uuid()  uuidv7() [18+]  uuid_extract_timestamp(u) [18+]
-- uuid-ossp: uuid_generate_v4()
gen_random_bytes(16)   -- pgcrypto
crypt('pw', gen_salt('bf'))  crypt('pw', stored_hash) = stored_hash
digest('x','sha256')  hmac(...)  pgp_sym_encrypt(text, key)  pgp_sym_decrypt(bytea, key)
```

## System information functions
```sql
version()  current_database()  current_user  session_user  current_schema  current_setting('x')  set_config('x','v',false)
pg_backend_pid()  pg_postmaster_start_time()  pg_conf_load_time()  txid_current()  pg_current_xact_id()
pg_relation_size()  pg_total_relation_size()  pg_table_size()  pg_indexes_size()  pg_database_size()  pg_size_pretty()  pg_size_bytes('1GB')
pg_relation_filepath('t')  pg_get_serial_sequence('t','id')  pg_get_expr  pg_get_viewdef  pg_get_functiondef
has_table_privilege  has_schema_privilege  has_database_privilege  pg_has_role
obj_description  col_description  to_regclass('x')  to_regtype  to_regproc
pg_ls_dir  pg_read_file  pg_stat_file  pg_ls_waldir()
nextval('seq')  currval('seq')  setval('seq', 100)  lastval()
```

## Sequence helpers
```sql
SELECT setval(pg_get_serial_sequence('t','id'), (SELECT max(id) FROM t));
SELECT nextval('seq');
ALTER SEQUENCE seq RESTART WITH 1;
```

## Operators cheat list
```
Comparison: = <> != < > <= >= IS DISTINCT FROM
Logic: AND OR NOT
Math: + - * / % ^ |/ ||/ @ & | # ~ << >>
String: || ~~ (LIKE) ~~* (ILIKE) ~ ~* !~ !~*
Array: @> <@ && || = ANY
JSON: -> ->> #> #>> @> <@ ? ?| ?& || - #- @? @@
Range: @> <@ && << >> -|- + * -
Network: << >>= <<= >> &&
Text search: @@  ||  &&  !!
```
