# Common Character Bypasses

## Introduction

It is not uncommon to run into`character limitations`when trying to`exploit`an otherwise simple`SQL injection`. Perhaps a developer implemented a`whitelist`of characters the field is allowed to accept, or perhaps a`WAF`doesn't like it when you set your "username" to certain strings. In any case, the ability to be flexible with your`SQL injection payloads`can be very useful.

## Blind SQL Injection

To practice`SQL injection`with a`character filter`, let's revist the first SQLi vulnerability that we discovered in the`Identifying Vulnerabilities`section:

Code:java```
`// IndexController.java (Lines 50-76)@GetMapping({"/find-user"})publicStringfindUser(@RequestParamStringu,Modelmodel,HttpServletResponseresponse)throwsIOException{Patternp=Pattern.compile("'|(.*'.*'.*)");Matcherm=p.matcher(u);Stringu2=u.toLowerCase();if(!u2.contains(" ")&&!m.matches()){try{Stringsql="SELECT * FROM users WHERE username LIKE '%"+u+"%'";Listusers=this.jdbcTemplate.query(sql,newBeanPropertyRowMapper(User.class));UserDetailsImpluserDetails=(UserDetailsImpl)SecurityContextHolder.getContext().getAuthentication().getPrincipal();model.addAttribute("userDetails",userDetails);model.addAttribute("users",users);return"find-user";}catch(BadSqlGrammarExceptionvar10){System.out.println(var10.getSQLException().getMessage());model.addAttribute("errorMsg","Invalid search query");return"error";}catch(Exceptionvar11){var11.printStackTrace();model.addAttribute("errorMsg","Invalid search query");return"error";}}else{model.addAttribute("errorMsg","Illegal search term");return"error";}}`
```

The call to`.contains(" ")`is self-explanatory - we can't use`spaces`- so let's take a look at the`RegEx`pattern to better understand what else`BlueBird`is looking for to count a term as an`illegal`search term.

Using[regex101.com](https://regex101.com/)to automatically generate an`explanation`for us, we can see that the pattern is supposed to match`single quotes`as well as strings with two`single quotes`somewhere within.

**********![Regex tool showing pattern ('|(.+'.+'.+)) with test strings and explanation.](https://academy.hackthebox.com/storage/modules/188/charbypass/3.png)Since the`u`variable is concatenated between two`single quotes`in`IndexController.java`, an`SQL injection`payload would require breaking out with a`single quote`. Based on the pattern alone, this doesn't seem possible. Unfortunately for the developer,[Matcher.matches()](https://www.javatpoint.com/post/java-matcher-matches-method)only returns`true`if the`entire`value of`u`matches the pattern, and not only part of it as he may have assumed. What this means is that while a`single quote`and strings surrounded by`single quotes`are detected, we can insert one`single quote`into a payload without matching the`RegEx`pattern.

Let's put this assumption to the test by running various payloads against the RegEx pattern. To do so we can create a short Java program like this one:

![Java program using regex to match input strings. Outputs "true" or "false" based on matches.](https://academy.hackthebox.com/storage/modules/188/charbypass/10.png)

Entering`a'`into the`"Find User"`search bar results in a`"Invalid search query"`error, but if we try`a'--`we should see that we successfully injected our payload into the`SQL query`.

**********![Web page showing user search results for Maria Petrova with a URL query find-user?u=a'--. Sidebar includes navigation and security trends like SQL Injection.](https://academy.hackthebox.com/storage/modules/188/charbypass/5.png)We can verify the injection completely by live-debugging the application and setting a breakpoint at the`findUser()`function in`IndexController.java`to see the full SQL query that is run:

![Java code for user search with SQL query vulnerability highlighted. Uses regex to validate input and queries database for usernames.](https://academy.hackthebox.com/storage/modules/188/charbypass/9.png)

Unfortunately, payloads such as`' and 1=1--`still fail, due to the`spaces`. This is very easy to work around however. We can simply replace all spaces with PostgreSQL’s multi-line empty comments (`/**/`), as the query will evaluate it to a space character and the query should still work. So we can try the payload`'/**/and/**/1=1--`, and see that it works.

**********![Web page displaying user search results with profiles. URL query includes encoded parameters. Sidebar shows navigation and security trends like SQL Injection.](https://academy.hackthebox.com/storage/modules/188/charbypass/6.png)## Union-Based SQL Injection

At this point we have a`PoC`for`blind SQL injection`, but we don't have to stop there. Coming back to the`single quotes`, we can get around this filter by expressing strings differently.`PostgreSQL`allows you to use`two dollar-signs`to mark the start and end-points of a string for better readability, and we can use this to get around the`RegEx`pattern matching`single quotes`and develop a`PoC`payload for`union-based SQL injection`.

So we would want to try the payload`' union select '1','2','3'--`, but after replacing`spaces`and`single quotes`it would look like:`'/**/union/**/select/**/$$1$$,$$2$$,$$3$$--`. After submitting this we get an`"Invalid Search Query"`error. Taking a look at the`PostgreSQL`logs (which we previously enabled) we can see that the statement failed because the first and second selects have a different number of columns.

```
icantthinkofaname23@htb[/htb]`$tail/opt/bluebird/pg_log/postgresql-2023-02-15_052440.log<SNIP>
2023-02-15 06:27:18.389 EST [14374] bbuser@bluebird ERROR:  each UNION query must have the same number of columns at character 67
2023-02-15 06:27:18.389 EST [14374] bbuser@bluebird STATEMENT:  SELECT * FROM users WHERE username LIKE '%'/**/union/**/select/**/$$1$$,$$2$$,$$3$$--%'
<SNIP>`
```

We can check the code to find the correct number of columns, or we can simply keep adding one until we succeed. Either way, the correct number of columns is`6`and so once we try the payload`'/**/union/**/select/**/$$1$$,$$2$$,$$3$$,$$4$$,$$5$$,$$6$$--`we should see that the`union-based SQL injection`was successful!

**********![Web page showing user search results with profiles. URL query includes encoded SQL injection attempt. Sidebar contains navigation options.](https://academy.hackthebox.com/storage/modules/188/charbypass/7.png)## Comparative Precomputation (Blind SQLi)

Although we already proved that`union-based SQL injection`is possible in this specific case, let's pretend that we were restricted to`blind SQL injection`for a moment. Using typical algorithms to dump characters from the database,`7 requests`are required per character on average (e.g. the`bisection`algorithm that[sqlmap](https://sqlmap.org/)uses). In some cases however, it may be possible to (blindly) dump`1 or more characters per request`.

Take the following payload for example:

Code:sql```
`' AND id=(SELECT ASCII(SUBSTRING(password,1,1)) FROM users WHERE username='itsmaria')--`
```

It will only match one user, and that is the user whose`id`equals the`ascii value`of the`first character`of`itsmaria's password`. If we swap out the`spaces`and`single quotes`to bypass the character filters, our payload will look like this:

Code:sql```
`'/**/AND/**/id=(SELECT/**/ASCII(SUBSTRING(password,1,1))/**/FROM/**/users/**/WHERE/**/username=$$itsmaria$$)--`
```

If we try it out, we should see the user with ID`36`appear, which corresponds to the character`$`. This is expected, since the password hashes are stored as`bcrypt`hashes which have the format`$2b$12$...`, so we now have a`blind SQL injection PoC`which can dump`one character per request`.

**********![Web page displaying user search results for "Frank Mcculley" with developer tools open, showing HTML and CSS for a profile link. URL query includes encoded SQL injection attempt.](https://academy.hackthebox.com/storage/modules/188/charbypass/8.png)Note: For the purposes of this module, you pretty much have to test exploits against the live website over port 8080. In the real world you`always want to test/develop exploits locally`before running them against any production sites. Ideally, changing IP/PORT should be the only changes you have to make.

## Challenge

As an extra challenge, write a simple Python script which automatically encodes payloads for you. A further step could be that it also sends the query and returns the output.
