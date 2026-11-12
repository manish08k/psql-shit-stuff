# 05 — Full-Text Search & Fuzzy Matching

## Basics
```sql
to_tsvector('english', 'The quick brown foxes')                 -- 'brown':3 'fox':4 'quick':2
to_tsquery('english', 'quick & fox')                            -- operators: & | ! <-> <N>
plainto_tsquery('english', 'quick fox')                         -- all terms ANDed
phraseto_tsquery('english', 'quick brown')                      -- phrase
websearch_to_tsquery('english', '"quick brown" -slow OR fast')  -- Google-style
to_tsquery('english', 'fox:*')                                  -- prefix match
tsvec @@ tsq                                                    -- match
```

## Ranking / highlight
```sql
ts_rank(tsv, q)   ts_rank_cd(tsv, q)   ts_rank(ARRAY[0.1,0.2,0.4,1.0], tsv, q, 32)
setweight(to_tsvector(title), 'A') || setweight(to_tsvector(body), 'B')
ts_headline('english', body, q, 'StartSel=<b>, StopSel=</b>, MaxWords=35, MinWords=15')
ts_debug('english', 'text')   ts_lexize('english_stem', 'foxes')   ts_stat('SELECT tsv FROM t')
numnode(q)   querytree(q)   ts_rewrite(...)   tsvector_to_array(v)   array_to_tsvector(a)   strip(v)
```

## Production setup
```sql
ALTER TABLE doc ADD COLUMN tsv tsvector
  GENERATED ALWAYS AS (setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
                       setweight(to_tsvector('english', coalesce(body,'')),  'B')) STORED;
CREATE INDEX doc_tsv_idx ON doc USING gin (tsv);
SELECT id, title, ts_rank(tsv, q) AS rank
FROM doc, websearch_to_tsquery('english', 'postgres replication') q
WHERE tsv @@ q ORDER BY rank DESC LIMIT 20;
-- trigger alternative (older):
-- CREATE TRIGGER ... BEFORE INSERT OR UPDATE ... EXECUTE FUNCTION tsvector_update_trigger(tsv,'pg_catalog.english',title,body);
```

## Configuration
```sql
SHOW default_text_search_config;   SET default_text_search_config = 'pg_catalog.english';
\dF   \dFd   \dFp   \dFt
CREATE TEXT SEARCH CONFIGURATION mycfg (COPY = english);
ALTER TEXT SEARCH CONFIGURATION mycfg ALTER MAPPING FOR asciiword, word WITH unaccent, english_stem;
CREATE EXTENSION unaccent;
CREATE TEXT SEARCH DICTIONARY my_syn (TEMPLATE = synonym, SYNONYMS = my_synonyms);
```

## Fuzzy / trigram (pg_trgm)
```sql
CREATE EXTENSION pg_trgm;
SELECT similarity('word', 'wrod');    SELECT 'word' % 'wrod';    SELECT word <-> 'wrod';   -- distance
SET pg_trgm.similarity_threshold = 0.4;
CREATE INDEX ON t USING gin (name gin_trgm_ops);        -- also speeds LIKE '%x%', ILIKE, ~
CREATE INDEX ON t USING gist (name gist_trgm_ops);      -- supports ORDER BY name <-> 'x'
SELECT * FROM t WHERE name % 'jhon' ORDER BY similarity(name, 'jhon') DESC LIMIT 5;
SELECT word_similarity('a','b');   strict_word_similarity
```

## Other text helpers
```sql
CREATE EXTENSION fuzzystrmatch;   levenshtein('a','b')   soundex('x')   metaphone('x',10)   dmetaphone('x')
CREATE EXTENSION citext;          -- case-insensitive text type
CREATE COLLATION ci (provider = icu, locale = 'und-u-ks-level2', deterministic = false);
```
