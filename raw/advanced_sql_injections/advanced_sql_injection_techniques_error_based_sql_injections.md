# Error-Based SQL Injection

## Introduction

`Error-based SQL injection`is an`in-band`technique where attackers use`database error messages`to exfiltrate data. In this section will we go through an`error-based SQL injection`in`BlueBird`to better understand this technique.

## A Closer Look at the SQL Injection in /forgot

You may remember the second vulnerability that we discovered earlier had some interesting exception handling which returned a`stacktrace`or a generic error message depending on the client IP:

Code:java```
`// AuthController.java (Lines 121-164)@PostMapping({"/forgot"})publicStringforgotPOST(@RequestParamStringemail,Modelmodel,HttpServletRequestrequest,HttpServletResponseresponse)throwsIOException{if(email.isEmpty()){response.sendRedirect("/forgot?e=Please+fill+out+all+fields");returnnull;}else{Patternp=Pattern.compile("^.*@[A-Za-z]*\\.[A-Za-z]*$");Matcherm=p.matcher(email);if(!m.matches()){response.sendRedirect("/forgot?e=Invalid+email!");returnnull;}else{try{Stringsql="SELECT * FROM users WHERE email = '"+email+"'";Useruser=(User)this.jdbcTemplate.queryForObject(sql,newBeanPropertyRowMapper(User.class));Longvar10000=user.getId();StringpasswordResetHash=DigestUtils.md5DigestAsHex((""+var10000+":"+user.getEmail()+":"+user.getPassword()).getBytes());var10000=user.getId();StringpasswordResetLink="https://bluebird.htb/reset?uid="+var10000+"&code="+passwordResetHash;logger.error("TODO- Send email with link ["+passwordResetLink+"]");response.sendRedirect("/forgot?e=Please+check+your+email+for+the+password+reset+link");returnnull;}catch(EmptyResultDataAccessExceptionvar11){response.sendRedirect("/forgot?e=Email+does+not+exist");returnnull;}catch(Exceptionvar12){StringipAddress=request.getHeader("X-FORWARDED-FOR");if(ipAddress==null){ipAddress=request.getRemoteAddr();}if(ipAddress.equals("127.0.1.1")){model.addAttribute("errorMsg",var12.getMessage());model.addAttribute("errorStackTrace",Arrays.toString(var12.getStackTrace()));}else{model.addAttribute("errorMsg","500 Internal Server Error");model.addAttribute("errorStackTrace","Something happened on our side. Please try again later.");}return"error";}}}}`
```

Let's throw a typical`' or 1=1--`injection at it to see what happens.

**********![BlueBird forgot password page with email input field, submit button, and "Invalid email!" error message.](https://academy.hackthebox.com/storage/modules/188/error/0.png)We should get an`Invalid email!`error, since the`RegEx`pattern failed to match. Taking a closer look at the`RegEx`pattern, we can see that it is quite a loose one which simply looks for`<CHARS>@<CHARS>.<CHARS>`. We can match this quite easily while still providing an`SQL injection`payload by appending`--@bluebird.htb`as a comment.

**********![Regex tool testing email pattern ^.*@[A-Za-z]*\\.[A-Za-z]*$ with test strings and explanation. Shows matches and match information.](https://academy.hackthebox.com/storage/modules/188/error/1.png)So let's try the injection again, this time with the payload`' or 1=1--@bluebird.htb`.

**********![BlueBird error page displaying "500 Internal Server Error" with message: "Something happened on our side. Please try again later."](https://academy.hackthebox.com/storage/modules/188/error/5.png)This time we got the generic error message as a response. If you recall from the code above, this response is served to all clients who don't have the IP address`127.0.1.1`, which is likely for local debugging. Regarding the IP check, the developers have made a very common mistake. The[X-Forwarded-For](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For)header is first checked to see if it was provided. This is a header which proxies use to note the original client IP. The problem is that since the header is provided from the client (proxy), we as attackers can supply whatever value we want.

To know more about Host header attacks, check the[Introduction to Host Header Attacks](https://academy.hackthebox.com/module/189/section/2012)section from the[Abusing HTTP Misconfigurations](https://enterprise.hackthebox.com/content-library?s=Abusing%20HTTP%20Misconfigurations)module.

Knowing this, we can try sending the same payload again, except this time intercept the request with`Burp`and add the`X-Forwarded-For`header:

**********![HTTP POST request to localhost:8080/forgot with headers and body. Includes X-Forwarded-For: 127.0.0.1 and an encoded email parameter.](https://academy.hackthebox.com/storage/modules/188/error/6.png)After forwarding the request, we should get a more verbose response from the server, including the stack trace.

**********![BlueBird error page displaying "Incorrect result size: expected 1, actual 362" with a detailed Java stack trace.](https://academy.hackthebox.com/storage/modules/188/error/2.png)This time we should see an error message like this show up. In this case specifically, there was an error since all rows from the`users`table were returned when only one was expected, but the more interesting point is that the error messages are returned to us.

## Exploiting the SQL Injection

Next let's try and force`PostgreSQL`to run into an error and reveal information in the message returned to us. A popular technique when it comes to`error-based SQL injection`is casting a unsuitable`STRING`to an`INT`because the value will be displayed in the error message. To test this out, we can use the payload`' and 0=CAST((SELECT VERSION()) AS INT)--@bluebird.htb`to try and leak the version of the database.

**********![BlueBird error page with SQL error message showing invalid input syntax for integer. Includes attempted SQL injection and PostgreSQL version details, followed by a Java stack trace.](https://academy.hackthebox.com/storage/modules/188/error/3.png)This time you will notice something interesting in the error message.`PostgreSQL`fails to convert`VERSION()`to an`INT`as expected and so it prints the value out in the error message which is returned to us.

The same technique can be used to leak pretty much anything from the database; you just need to get creative. For example, we can get one table name like this:

Code:sql```
`'and1=CAST((SELECTtable_nameFROMinformation_schema.tablesLIMIT1)asINT)--@bluebird.htb`
```

Or you could use[STRING_AGG](https://www.postgresql.org/docs/9.1/sql-expressions.html#SYNTAX-AGGREGATES)to select all table names at once like this:

Code:sql```
`' and 1=CAST((SELECT STRING_AGG(table_name,',')FROMinformation_schema.tablesLIMIT1)asINT)--@bluebird.htb`
```

If it is possible to`stack queries`in the specific`SQL injection`vulnerability you are targetting, you can even use[XML functions](https://www.postgresql.org/docs/9.4/functions-xml.html)to dump entire tables or databases at once like this:

Code:sql```
`';SELECT CAST(CAST(QUERY_TO_XML('SELECT*FROMpostsLIMIT2',TRUE,TRUE,'')ASTEXT)ASINT)--@bluebird.htb`
```

**********![BlueBird error page with SQL error showing invalid input syntax for integer. Includes attempted SQL injection with XML data output and a Java stack trace.](https://academy.hackthebox.com/storage/modules/188/error/4.png)
