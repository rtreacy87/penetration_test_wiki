# Introduction to PostgreSQL

## Introduction

In this module we will be exploring a few`advanced SQL injection`techniques as well as some[PostgreSQL](https://www.postgresql.org/)-specific attacks from a`white-box approach`. As this is an advanced module, an understanding of[SQL syntax, SQL injections](https://enterprise.hackthebox.com/content-library?s=SQL%20syntax,%20SQL%20injections)and[Python3](https://enterprise.hackthebox.com/content-library?s=Python3)is expected to fully grasp the concepts explained. Although this module will focus on`PostgreSQL`, the same techniques can be adapted to work with other SQL variants, as it is a[standardized](https://en.wikipedia.org/wiki/SQL)language.

## Interacting with PostgreSQL

Before we get into injection vulnerablities, let's take a moment to familiarize ourselves with two of the most common tools for interacting with`PostgreSQL`databases:[psql](https://www.postgresql.org/docs/current/app-psql.html)and[pgAdmin4](https://www.pgadmin.org/).

#### psql (PostgreSQL Interactive Terminal)

[psql](https://www.postgresql.org/docs/current/app-psql.html)is a`command-line`tool for interacting with`PostgreSQL`databases that comes pre-packaged with the`PostgreSQL`server and works on`Linux`or`Windows`.

You can install`psql`on a`Linux`distribution with this single command:

```
icantthinkofaname23@htb[/htb]`$sudoaptinstallpostgresql-client-15`
```

Note: It's possible that the distribution of`Linux`you are running does not have version`15`. In that case, you can install version`13`and everything will work fine with minimally adapted steps.

To install`psql`on`Windows`, you should first download the`PostgreSQL`installer from[postgresql.org](https://www.postgresql.org/download/windows)and then during the installation process unselect everything except for`Command Line Tools`.

![Setup window for selecting components to install. Options include PostgreSQL Server, pgAdmin 4, Stack Builder, and Command Line Tools (checked). Buttons for Back, Next, and Cancel.](https://academy.hackthebox.com/storage/modules/188/introduction/0.png)

Once it's done installing, you may use`psql.exe`from the installation directory (`C:\Program Files\PostgreSQL\15\bin`by default) or you can add the directory to the system`PATH`variable to be able to use it from anywhere:

![Steps to edit environment variables in Windows.](https://academy.hackthebox.com/storage/modules/188/introduction/1.png)

Once you've installed`psql`on your operating system of choice, you can connect to a`PostgreSQL`database with the following command:

```
icantthinkofaname23@htb[/htb]`$psql -h127.0.0.1[-p PORT]-U acdbuser acmecorpPassword for user acdbuser: 
psql (15.1 (Debian 15.1-1+b1), server 13.9 (Debian 13.9-0+deb11u1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

acmecorp=>`
```

Once connected, you can list databases with the`\l`command or`\l+`for extended details.

```
`acmecorp=> \l
                                             List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges   
-----------+----------+----------+---------+---------+------------+-----------------+-----------------------
 acmecorp  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | 
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
(4 rows)`
```

To switch to a database you can use the`\c <DATABASE>`command. In this case we are already in the`acmecorp`database.

To list the tables in a database (after you've switched to it), you can use the`\dt`command or`\dt+`for extended information.

```
`acmecorp=> \dt+
                                        List of relations
 Schema |    Name     | Type  |  Owner   | Persistence | Access method |    Size    | Description 
--------+-------------+-------+----------+-------------+---------------+------------+-------------
 public | departments | table | postgres | permanent   | heap          | 8192 bytes | 
 public | dept_emp    | table | postgres | permanent   | heap          | 72 kB      | 
 public | employees   | table | postgres | permanent   | heap          | 176 kB     | 
 public | salaries    | table | postgres | permanent   | heap          | 72 kB      | 
 public | titles      | table | postgres | permanent   | heap          | 80 kB      | 
(5 rows)`
```

Last, but not least, you can query the database simply by entering the query and making sure it's terminated with a semicolon. Multi-line queries work as well.

```
`acmecorp=> SELECT first_name, last_name, email FROM employees LIMIT 5;
 first_name |  last_name  |           email           
------------+-------------+---------------------------
 Kathleen   | Flint       | knflint82@acme.corp
 Henry      | Watson      | hywatson40@acme.corp
 Ruth       | Perez       | rhperez84@acme.corp
 Leon       | Tappin      | lntappin80@acme.corp
 Donita     | Fairweather | dafairweather92@acme.corp
(5 rows)`
```

#### pgAdmin4

[pgAdmin4](https://www.pgadmin.org/)is a`GUI`application for interacting with`PostgreSQL`databases that works on`Linux`and`Windows`.

To install`pgAdmin4`on`Linux`, run the following commands:

```
icantthinkofaname23@htb[/htb]`$curl-fsS https://www.pgadmin.org/static/packages_pgadmin_org.pub|sudogpg --dearmor -o /usr/share/keyrings/packages-pgadmin-org.gpg$sudosh-c'echo "deb [signed-by=/usr/share/keyrings/packages-pgadmin-org.gpg] https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs)pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list && apt update'$sudoaptinstallpgadmin4`
```

Note: If you are using`Kali`or`ParrotOS`(like the`Pwnbox`), you will want to replace`$(lsb_release -cs)`in the second command with`bullseye`, otherwise the installation will fail.

To install`pgAdmin4`on`Windows`you can download the installer from[pgadmin.org](https://www.pgadmin.org/download/pgadmin-4-windows/)and follow the installation steps, or you can reuse the installer we used to install`psql.exe`, just make sure the`pgAdmin4`option is checked this time in the installation process.

The first time you launch`pgAdmin4`you will have to set a master password. This is simply to protect the credentials you will later use to connect to databases.

![pgAdmin interface prompting to set a master password for securing and unlocking saved passwords and credentials. Input field with Cancel and OK buttons.](https://academy.hackthebox.com/storage/modules/188/introduction/2.png)

To connect to a`PostgreSQL`server, go to`Dashboard > Add New Server`, fill out the details and press`Save`.

![pgAdmin "Register - Server" window under the Connection tab. Fields for host name/address (127.0.0.1), port (5432), maintenance database, username, password, and options for Kerberos authentication and saving password. Close, Reset, and Save buttons.](https://academy.hackthebox.com/storage/modules/188/introduction/3.png)

Once you've done that, you can access the`server`on the`left-hand side`under whatever name you chose. Viewing databases and tables is very intuitive with this graphic interface, and running queries is as simple as`right-clicking`on a`database`and selecting the`Query Tool`.

![pgAdmin interface showing a query on the "acmecorp" database. Query: SELECT * FROM employees WHERE LENGTH(username) > 15;. Data output displays employee details like ID, username, email, password, first name, and last name.](https://academy.hackthebox.com/storage/modules/188/introduction/6.png)

## Practice

To finish off this section install`psql`or`pgAdmin4`, spawn and connect to the target database (`acmecorp`) with the credentials`acdbuser:AcmeCorp2023!`, and then answer the questions below.
