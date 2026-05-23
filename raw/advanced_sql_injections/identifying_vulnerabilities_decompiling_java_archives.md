# Decompiling Java Archives

## Introduction

Imagine that we were contracted to perform a`white-box`security assessment on a target application named`BlueBird`, a[Java Spring Boot](https://spring.io/)web application which uses`PostgreSQL`as its database.

**********![Social media interface with a sidebar for navigation, a text box for posting, and a list of user posts. Trends include topics like Cross-Site Scripting and SQL Injection.](https://academy.hackthebox.com/storage/modules/188/decompiling/0.png)We don't have access to`BlueBird's`source code, but we were given access to the compiled[JAR](https://en.wikipedia.org/wiki/JAR_(file_format))file which, if you aren't familiar with Java, is essentially a Java executable. Let's take a look at two tools we can use to decompile and retrieve`BlueBird's`source code from the`JAR`file so that we can start searching through it for vulnerablities.

Note: It is assumed that you have`Java`installed on your machine. If you don't already have it, head on over to[OpenJDK.org](https://openjdk.org/)and install the latest version.

#### Testing VM

During this module you are given access to a`testing VM`which has the`BlueBird`JAR file, Java installed, and PostgreSQL installed and initialized.

You may connect via SSH using the username`student`and password`academy.hackthebox.com`.

`BlueBird`application files are located in`/opt/bluebird`, as well as`PostgreSQL`log files (more on that in a later section).

```
icantthinkofaname23@htb[/htb]`$/opt/bluebird$ls-lahtotal 45M
drwxr-xr-x 3 root root 4.0K Feb 28 15:07 .
drwxr-xr-x 3 root root 4.0K Feb 28 11:22 ..
-rwxr-xr-x 1 root root  45M Feb 28 11:24 BlueBird-0.0.1-SNAPSHOT.jar
drwxrwxrwx 2 root root 4.0K Feb 28 15:19 pg_log
-rwxr-xr-x 1 root root  319 Feb 28 11:35 serverInfo.sh`
```

Also in`/opt`is a folder named`Pass2`which contains`Pass2-1.0.3-SNAPSHOT.jar`. You can download this, but ignore it for now, it will be used in the skills assessment.

The`student`user may run the following commands with`sudo`, so that you may restart the services if necessary.

```
icantthinkofaname23@htb[/htb]`$sudo-lMatching Defaults entries for student on bb01:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User student may run the following commands on bb01:
    (ALL) NOPASSWD: /usr/bin/systemctl start bluebird
    (ALL) NOPASSWD: /usr/bin/systemctl stop bluebird
    (ALL) NOPASSWD: /usr/bin/systemctl start postgresql
    (ALL) NOPASSWD: /usr/bin/systemctl stop postgresql`
```

## Fernflower

[Fernflower](https://github.com/JetBrains/intellij-community/tree/master/plugins/java-decompiler/engine)is an open-source Java decompiler which is maintained by[JetBrains](https://www.jetbrains.com/)and included in their[IntelliJ IDEA](https://www.jetbrains.com/idea/)IDE. To use this tool, we first need to compile it.

To avoid downloading all the unwanted/extra files in the[official repository](https://github.com/JetBrains/intellij-community/tree/master/plugins/java-decompiler/engine), we can clone an unofficial mirror of the specific folder containing Fernflower, including:

- [github.com/fesh0r/fernflower](https://github.com/fesh0r/fernflower)
- [github.com/MinecraftForge/FernFlower](https://github.com/MinecraftForge/FernFlower)
So let's pick one of these and clone it:

```
icantthinkofaname23@htb[/htb]`$gitclone https://github.com/fesh0r/fernflower.gitCloning into 'fernflower'...
remote: Enumerating objects: 12680, done.
remote: Counting objects: 100% (2795/2795), done.
remote: Compressing objects: 100% (859/859), done.
remote: Total 12680 (delta 1435), reused 2541 (delta 1297), pack-reused 9885
Receiving objects: 100% (12680/12680), 6.39 MiB | 1.84 MiB/s, done.
Resolving deltas: 100% (7209/7209), done.`
```

Once the repository has been cloned, enter its directory and use[Gradle](https://gradle.org/)to build`Fernflower`.

Note: Due to an update on September 1, 2025, Fernflower now uses JDK 21, disrupting the build process with JDK 17. To continue using JDK 17, clone the repository, navigate into the directory and run`git checkout c03cae7`.

```
icantthinkofaname23@htb[/htb]`$./gradlew buildPicked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true

Welcome to Gradle 7.5.1!

Here are the highlights of this release:
 - Support for Java 18
 - Support for building with Groovy 4
 - Much more responsive continuous builds
 - Improved diagnostics for dependency resolution

For more details see https://docs.gradle.org/7.5.1/release-notes.html

Starting a Gradle Daemon (subsequent builds will be faster)

> Task :test
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true

BUILD SUCCESSFUL in 20s
4 actionable tasks: 4 executed`
```

`Gradle`was able to build`Fernflower`successfully because the`JDK`version on the system matched the one used to develop`Fernflower`(specifically,`JDK 17`), however, when trying to build it on a machine with a non-matching`JDK`version, we will get the following error:

```
icantthinkofaname23@htb[/htb]`$./gradlew buildDownloading https://services.gradle.org/distributions/gradle-7.5.1-bin.zip

<SNIP>

Starting a Gradle Daemon (subsequent builds will be faster)
> Task :compileJava FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':compileJava'.
> error: invalid source release: 17

* Try:
> Run with --stacktrace option to get the stack trace.
> Run with --info or --debug option to get more log output.
> Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 15s
1 actionable task: 1 executed`
```

To resolve this issue, we need to use`apt`to install`openjdk-17-jdk`:

```
icantthinkofaname23@htb[/htb]`$sudoaptinstallopenjdk-17-jdkReading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following packages were automatically installed and are no longer required:
  libgit2-1.1 libmbedcrypto3 libmbedtls12 libmbedx509-0 libstd-rust-1.48 libstd-rust-dev linux-kbuild-5.18 rust-gdb
Use 'sudo apt autoremove' to remove them.
The following additional packages will be installed:
  openjdk-17-jdk-headless openjdk-17-jre openjdk-17-jre-headless
Suggested packages:
  openjdk-17-demo openjdk-17-source visualvm fonts-ipafont-gothic fonts-ipafont-mincho fonts-wqy-microhei | fonts-wqy-zenhei fonts-indic
The following NEW packages will be installed:
  openjdk-17-jdk openjdk-17-jdk-headless
The following packages will be upgraded:
  openjdk-17-jre openjdk-17-jre-headless
2 upgraded, 2 newly installed, 0 to remove and 106 not upgraded.
Need to get 278 MB of archives.
After this operation, 244 MB of additional disk space will be used.
Do you want to continue? [Y/n] Y

<SNIP>`
```

Afterward, we need to set it as the default`JDK`for our system, to do so, first we need to know the path to`JDK 17`using`update-java-alternative`along with the`--list`flag:

```
icantthinkofaname23@htb[/htb]`$sudoupdate-java-alternatives --listjava-1.11.0-openjdk-amd64      1111       /usr/lib/jvm/java-1.11.0-openjdk-amd64
java-1.13.0-openjdk-amd64      1311       /usr/lib/jvm/java-1.13.0-openjdk-amd64
java-1.17.0-openjdk-amd64      1711       /usr/lib/jvm/java-1.17.0-openjdk-amd64`
```

The path is`/usr/lib/jvm/java-1.17.0-openjdk-amd64`, thus, we now need to set it as the default with`update-java-alternative`along with the`--set`flag:

```
icantthinkofaname23@htb[/htb]`$sudoupdate-java-alternatives --set /usr/lib/jvm/java-1.17.0-openjdk-amd64`
```

Trying to build`Fernflower`again, we will notice that the error disappears as the issue got resolved.

Once`Gradle`is done,`Fernflower`should have been compiled into a`JAR`file located at`build/libs/fernflower.jar`. We can use this to decompile`BlueBird`like this (make sure`out`is a real folder):

```
icantthinkofaname23@htb[/htb]`$java -jar fernflower.jar BlueBird-0.0.1-SNAPSHOT.jar outPicked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
INFO:  Decompiling class org/springframework/boot/loader/ClassPathIndexFile
INFO:  ... done
INFO:  Decompiling class org/springframework/boot/loader/ExecutableArchiveLauncher
<SNIP>
INFO:  Decompiling class com/bmdyy/bluebird/model/Post
INFO:  ... done
INFO:  Decompiling class com/bmdyy/bluebird/model/User
INFO:  ... done`
```

Once`Fernflower`is done, we can enter`out`and there should be a single`JAR`file containg a bunch of source`.java`files. We can use the following command to extract them all:

```
icantthinkofaname23@htb[/htb]`$jar -xf BlueBird-0.0.1-SNAPSHOT.jarPicked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true`
```

At this point, we should have the source`.java`files inside the`BOOT-INF/classes`directory.

```
icantthinkofaname23@htb[/htb]`$tree.
├── BlueBird-0.0.1-SNAPSHOT.jar
├── BOOT-INF
│   ├── classes
│   │   ├── application.properties
│   │   ├── com
│   │   │   └── bmdyy
│   │   │       └── bluebird
│   │   │           ├── BlueBirdApplication.java
│   │   │           ├── controller
│   │   │           │   ├── AuthController.java
│   │   │           │   ├── IndexController.java
│   │   │           │   ├── PostController.java
│   │   │           │   ├── ProfileController.java
│   │   │           │   └── ServerInfoController.java
│   │   │           ├── model
│   │   │           │   ├── Post.java
│   │   │           │   └── User.java
│   │   │           └── security
│   │   │               ├── jwt
│   │   │               │   ├── AuthEntryPointJwt.java
│   │   │               │   ├── AuthTokenFilter.java
│   │   │               │   └── JwtUtils.java
│   │   │               ├── services
│   │   │               │   ├── UserDetailsImpl.java
│   │   │               │   └── UserDetailsServiceImpl.java
│   │   │               └── WebSecurityConfig.java
│   │   ├── static
│   │   │   └── css
│   │   │       └── styles.css
│   │   └── templates
│   │       ├── edit-profile.html
│   │       ├── error.html
│   │       ├── find-user.html
│   │       ├── forgot.html
│   │       ├── home-logged-in.html
│   │       ├── home-logged-out.html
│   │       ├── login.html
│   │       ├── profile.html
│   │       ├── reset.html
│   │       ├── server-info.html
│   │       └── signup.html
│   ├── classpath.idx
│   ├── layers.idx
│   └── lib
<SNIP>
25 directories, 136 files`
```

## JD-GUI

Another open-source tool we can use to decompile`JAR`files is[JD-GUI](https://github.com/java-decompiler/jd-gui). As the name suggests, this one has a graphic interface which we can use to view the decompiled files. It works well, however the last release was in 2019 and the project might even be discontinued.

We can download the latest release`JAR`file from[here](https://github.com/java-decompiler/jd-gui/releases)and run it like so:

```
icantthinkofaname23@htb[/htb]`$java -jar jd-gui-1.6.6.jar BlueBird-0.0.1-SNAPSHOT.jarPicked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true`
```

![Code editor interface displaying a project structure on the left and Java code for a profile controller on the right. The code includes methods for handling profile edits with validation and SQL queries.](https://academy.hackthebox.com/storage/modules/188/decompiling/1.png)

We can use the UI to view the`.java`source files and even search for strings, variables or methods. Alternatively,[Visual Studio Code](https://code.visualstudio.com/)can be used to look through the source code. You can save the source files by hitting`File > Save All Sources`and then unzipping the created`ZIP`archive.
