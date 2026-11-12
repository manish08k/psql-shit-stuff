# 06 — Output & Formatting (`\pset` and friends)

## `\pset` options (all, PostgreSQL 18)
| Option (alias) | Values | Notes |
|---|---|---|
| `border` | 0, 1, 2 (latex-longtable: up to 3) | 0 none · 1 inner lines · 2 full frame |
| `columns` | int | width for `wrapped`; 0 = `$COLUMNS` / screen width |
| `csv_fieldsep` | string | CSV separator (default `,`) |
| `expanded` (`x`) | `on` `off` `auto` | no value toggles; `auto` expands only if wider than screen |
| `fieldsep` | string | unaligned separator (default `\|`); tab: `\pset fieldsep '\t'` |
| `fieldsep_zero` | – | NUL separator |
| `footer` | `on` `off` | `(n rows)` footer; no value toggles |
| `format` | `aligned` `asciidoc` `csv` `html` `latex` `latex-longtable` `troff-ms` `unaligned` `wrapped` | default `aligned`; unique prefixes OK |
| `linestyle` | `ascii` `old-ascii` `unicode` | borders for aligned/wrapped |
| `null` | string | how NULL shows (default empty) |
| `numericlocale` | `on` `off` | locale digit grouping |
| `pager` | `on` `off` `always` | no value toggles on/off |
| `pager_min_lines` | int | page only if output ≥ N lines |
| `recordsep` | string | unaligned record separator |
| `recordsep_zero` | – | NUL record separator |
| `tableattr` (`T`) | string | HTML `<table>` attributes (latex-longtable: column widths) |
| `title` (`C`) | string | table title; no value unsets |
| `tuples_only` (`t`) | `on` `off` | no headers/footer |
| `unicode_border_linestyle` | `single` `double` | with linestyle unicode |
| `unicode_column_linestyle` | `single` `double` | |
| `unicode_header_linestyle` | `single` `double` | |
| `xheader_width` | `full`(default) `column` `page` or int | header width in expanded mode |

`\pset` with no args lists the current settings. No value for a boolean option toggles it.
(There is **no** `markdown` format in psql.)

## Shortcuts
```
\x            expanded toggle        \x on|off|auto
\t            tuples only toggle     \a   aligned ⇄ unaligned
\H            HTML toggle            \C 'title'    \f ','    \T 'class=x'
\timing       toggle timing          \pset pager off
```

## Common setups
```
\x auto                                         -- vertical when wide (best default)
\pset null '∅'                                  -- see NULLs
\pset linestyle unicode
\pset border 2
\pset unicode_border_linestyle double
\pset pager off                                 -- or: \setenv PAGER 'less -S'  (no wrap, scroll sideways)
\pset format wrapped \pset columns 100
\pset numericlocale on
\pset title 'Sales report'
\pset footer off
```

## Export formats
```
\pset format csv                    -- then: \o out.csv  SELECT ...;  \o
\pset format html                   -- \H
\pset format asciidoc
\pset format latex                  -- latex-longtable for multi-page
\pset format troff-ms
\pset format unaligned + \pset fieldsep '\t' + \pset tuples_only on      -- TSV
```
```bash
psql --csv -c "SELECT ..." > out.csv
psql -P format=html -c "SELECT ..." > out.html
psql -At -F $'\t' -c "SELECT ..." > out.tsv
```

## Redirecting output
```
\o file            all subsequent results → file      \o            back to terminal
\o |command        pipe results                        \o | gzip > r.gz
\g file            only this query                     \g |cmd
\gx                this query, expanded
\echo text         stdout          \qecho text   → goes to \o target     \warn text → stderr
-L logfile         (CLI) log all output as well as showing it
```

## Per-query formatting
```sql
SELECT * FROM t \g (format=csv tuples_only=off) out.csv
SELECT * FROM t \g (format=aligned expanded=on)
SELECT * FROM t \gx (border=2)
```

## Timing
```
\timing on
SELECT ...;
Time: 12.345 ms          (>1 s also prints  mm:ss.mmm,  hours/days when needed)
```

## Pager
```
\pset pager on|off|always
\setenv PAGER 'less -S -X'
PSQL_PAGER='pspg -X -b' psql
PSQL_WATCH_PAGER='pspg --stream' psql
\pset pager_min_lines 40
```

## `\crosstabview` (pivot in psql)
```sql
SELECT region, month, sum(amount) FROM sales GROUP BY 1,2 \crosstabview
\crosstabview region month sum            -- colV colH colD
\crosstabview region month sum month_no   -- sort horizontal header by another column
```
Query must return ≥ 3 columns; (colV, colH) pairs must be unique.

## Wide output tips
```
\x auto
\pset pager always  +  PAGER='less -S'           -- horizontal scroll
SELECT ... \gx                                   -- one-off vertical
\pset format wrapped
```

## Encoding
```
\encoding            show     \encoding UTF8    set client encoding
SET client_encoding = 'UTF8';
```
