# 04 — The `\d` Family (list / describe objects)

Modifiers (after the command letters): `S` include system objects · `x` expanded output · `+` more detail.
Example: `\dtS+x pg_class` → system tables, verbose, expanded.

## Relations
```
\d[S+x] [pattern]       no pattern: tables, views, matviews, sequences, foreign tables
                        with name: columns, types, defaults, indexes, constraints, FKs, triggers, rules, partitions
\dt[S+x] [pat]          tables               \dv[S+x] views              \dm[S+x] materialized views
\di[S+x] [pat]          indexes              \ds[S+x] sequences          \dE[S+x] foreign tables
\dP[itnx+] [pat]        partitioned relations (i=indexes t=tables n=nested)
```

## Schemas, databases, tablespaces
```
\dn[S+x] [pat]          schemas
\l[+] [pat]  | \list    databases
\db[+x] [pat]           tablespaces
```

## Functions, types, operators
```
\df[anptwS+x] [pat [argtype ...]]   functions (a=aggregate n=normal p=procedure t=trigger w=window)
\da[Sx] [pat]           aggregates
\do[S+x] [pat [arg [arg]]]   operators
\dT[S+x] [pat]          data types           \dD[S+x] domains
\dC[+x] [pat]           casts                \dc[S+x] encoding conversions
\dO[S+x] [pat]          collations
\dL[S+x] [pat]          procedural languages
\dA[+x] [pat]           access methods
\dAc[+x] [am [type]]    operator classes     \dAf[+x] [am [type]]  operator families
\dAo[+x] [am [family]]  operators in families \dAp[+x] [am [family]] support functions
```
Examples: `\df int*pl * bigint` · `\df+x public.my_func` · `\do + integer integer`

## Roles & privileges
```
\du[S+x] [pat]  | \dg   roles
\drg[Sx] [pat]          role memberships (who is in what)
\dp[Sx] [pat]  | \z     access privileges (tables/views/sequences)
\ddp[x] [role pat [schema pat]]   default privileges
\drds[x] [role pat [db pat]]      per-role / per-database settings
```

## Config / extensions / replication / misc
```
\dconfig[+x] [pat]      server parameters (non-default only; \dconfig * = all; \dconfig *mem*)
\dx[+x] [pat]           extensions            \dX[x] [pat]   extended statistics
\dRp[+x] [pat]          publications          \dRs[+x] [pat] subscriptions
\dy[+x] [pat]           event triggers
\dd[Sx] [pat]           comments on constraints / operator classes / families / rules / triggers
\dF[+x]  \dFd  \dFp  \dFt     text search configs / dictionaries / parsers / templates
\des[+x] \det[+x] \deu[+x] \dew[+x]    foreign servers / foreign tables / user mappings / FDWs
\dl[+x]  = \lo_list     large objects
```

## Patterns
- Unquoted names fold to lowercase: `\dt FOO` matches `foo`. Quote to keep case: `\dt "FOO"`.
- `*` any characters · `?` one character · (literal inside double quotes).
- `schema.object`: `\dt sales.*` · `\dt *.orders` · `\dt *.*` (everything).
- `db.schema.object` allowed only if db = connected database.
- Regex classes allowed: `\dt user[0-9]*`. Pattern is auto-anchored.
- No pattern = objects visible on `search_path` (user objects only unless `S`).

## Typical use
```
\dt+                       tables with sizes
\dt+ app.*                 one schema
\d+ orders                 full table description (FKs, indexes, triggers, partition info)
\di+ orders*               indexes with size
\dv+ report_*              view definitions
\df+ calc_*                function source + volatility
\sf calc_tax               print function body
\dp orders                 who can do what
\ddp                       default privileges
\du+                       roles with descriptions
\dconfig *wal*             WAL-related settings
\dx+ pg_stat_statements    extension objects
\l+                        databases with sizes/tablespaces
\dn+ ; \dn+ app            schemas, owners, privileges
\dRp+ ; \dRs+              replication
\drg                       role membership
```

## See what psql runs
```
\set ECHO_HIDDEN on         show the SQL behind \d  (or: psql -E)
\set ECHO_HIDDEN noexec     show only, don't run
```
