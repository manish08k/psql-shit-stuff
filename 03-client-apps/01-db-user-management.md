# Client Apps — Database & User Management

Common connection flags for all tools: `-h host  -p port  -U user  -w|-W  --maintenance-db=db  -e (echo SQL)`

## createdb
```bash
createdb mydb
createdb -O owner -T template0 -E UTF8 -l en_US.UTF-8 --locale-provider=icu --icu-locale=en-US mydb "comment"
createdb -D tablespace -h host -p 5432 -U postgres mydb
createdb --strategy=wal_log mydb
```

## dropdb
```bash
dropdb mydb
dropdb -i mydb                 # interactive confirm
dropdb --if-exists -f mydb     # force: terminate connections
```

## createuser
```bash
createuser alice
createuser -P alice                       # prompt for password
createuser -s admin                       # superuser
createuser -d -r alice                    # createdb, createrole
createuser -l -c 10 alice                 # login, connection limit
createuser -g group1 alice                # add to role
createuser --replication repl
createuser --interactive
createuser -S -D -R alice                 # no superuser/createdb/createrole
createuser --valid-until='2027-01-01' alice
```

## dropuser
```bash
dropuser alice
dropuser -i alice
dropuser --if-exists alice
```
