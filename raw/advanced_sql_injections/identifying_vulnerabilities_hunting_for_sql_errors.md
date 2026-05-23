# Hunting for SQL Errors

## Enabling PostgreSQL Logging

Another way to identify the`SQL`queries which are run, as well as debug your payloads when developing an exploit is to enable`SQL logging`.

To do so in`PostgreSQL`, we first need to find`postgresql.conf`. Usually it is located in`/etc/postgresql/<version>/main/`, but if you can't find it there you can run:

```
icantthinkofaname23@htb[/htb]`$find/ -type f -name postgresql.conf2>/dev/null/etc/postgresql/13/main/postgresql.conf`
```

Once we've located the file, we have to make the following changes to the file:

- Change`#logging_collector = off`to`logging_collector = on`. This enables the logging collector background process [[source](https://postgresqlco.nf/doc/en/param/logging_collector/)].
- `#log_statement = 'none'`to`log_statement = 'all'`. This makes it so all statement types (SELECT, CREATE, INSERT, ...) are logged [[source](https://postgresqlco.nf/doc/en/param/log_statement/)].
- Uncomment`#log_directory = '...'`to define the directory in which the logfiles will be saved [[source](https://postgresqlco.nf/doc/en/param/log_directory/)].
- Uncomment`#log_filename = '...'`to define the filename in which logfiles will be saved [[source](https://postgresqlco.nf/doc/en/param/log_filename/)].
Once the changes have been saved, restart`PostgreSQL`like so:

```
icantthinkofaname23@htb[/htb]`$sudosystemctl restart postgresql`
```

At this point, the log file(s) should start appearing in the folder defined by`log_directory`. We can watch the log messages in near-realtime with the following command:

```
icantthinkofaname23@htb[/htb]`$sudowatch-n1tail<log_directory>/postgresql-2023-02-14_081533.log
<SNIP>
2023-02-14 09:06:04.819 EST [22510] bbuser@bluebird LOG:  execute <unnamed>: SELECT * FROM users WHERE username =$12023-02-14 09:06:04.819 EST [22510] bbuser@bluebird DETAIL:  parameters:$1='bmdyy'2023-02-14 09:06:10.423 EST [22510] bbuser@bluebird LOG:  execute <unnamed>: SELECT * FROM users WHERE username =$12023-02-14 09:06:10.423 EST [22510] bbuser@bluebird DETAIL:  parameters:$1='admin'2023-02-14 09:06:12.999 EST [22510] bbuser@bluebird LOG:  execute <unnamed>: SELECT * FROM users WHERE username =$12023-02-14 09:06:12.999 EST [22510] bbuser@bluebird DETAIL:  parameters:$1='test'2023-02-14 09:06:16.688 EST [22510] bbuser@bluebird LOG:  execute <unnamed>: SELECT * FROM users WHERE username =$12023-02-14 09:06:16.688 EST [22510] bbuser@bluebird DETAIL:  parameters:$1='itsmaria'`
```
