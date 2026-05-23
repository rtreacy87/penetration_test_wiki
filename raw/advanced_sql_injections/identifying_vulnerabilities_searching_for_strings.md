# Searching for Strings

## RegEx

Now that we have the decompiled source files for`BlueBird`we can start searching for vulnerabilities; in this case we are specifically interested in`SQL injection`vulnerabilities.

In most cases, SQL queries are simply`strings`that are passed to a database to be processed. In the case where we want to identify`SQL injection`vulnerabilities, it is necessary to analyze the`SQL queries`being used in the program to see if any are vulnerable. Rather than manually scrolling through lines upon lines of code, we can use`Regular Expressions`(`RegEx`) to significantly speed up our efforts.

In the table below are a few`RegEx`patterns that we can use.

QueryDescription`SELECT|UPDATE|DELETE|INSERT|CREATE|ALTER|DROP`Search for the basic SQL commands. Injection can occur in more than just SELECT statements, exploitation may just be a bit trickier.`(WHERE|VALUES).*?'`Search for strings which include`WHERE`or`VALUES`and then a`single quote`, which could indicate a string concatenation.`(WHERE|VALUES).*" \+`Search for strings which include`WHERE`or`VALUES`followed by a double quote and a plus sign, which could indicate a string concatenation.`.*sql.*"`Search for lines which include`sql`and then a`double quote`.`jdbcTemplate`Search for lines which include`jdbcTemplate`. There are various ways to interact with`SQL`databases in`Java`.`JdbcTemplate`is one of them; others include`JPA`and`Hibernate`.When analyzing source code, it is useful to take note of the`libraries used`as well as the`coding style`so you can adapt your search queries to be more effective. For example, as we look through the results in`BlueBird`, we will notice that the developer always stores`SQL`queries in a variable named`sql`, so that's something we could look for specifically.

## Grep

One way we can use these`RegEx`patterns against the decompiled source files is with[grep](https://man7.org/linux/man-pages/man1/grep.1.html), which is a`command-line`tool used to search for`patterns`in`files`.

The syntax to use`RegEx`with`grep`is`grep -E <RegEx> <File>`, but we can set a few more  arguments to improve our results, such as`--include *.java`to only search for matches in`.java`files, using`-n`to display line numbers,`-i`to ignore case, and`-r`to search recursively through a directory.

As a result, the command we want to use will look something like this:

```
icantthinkofaname23@htb[/htb]`$grep-irnE'SELECT|UPDATE|DELETE|INSERT|CREATE|ALTER|DROP'../com/bmdyy/bluebird/controller/PostController.java:21:    public void createPost(@RequestParam String text, HttpServletResponse response) throws IOException {
./com/bmdyy/bluebird/controller/PostController.java:25:            String sql = "INSERT INTO posts (text, author_id, posted_at) VALUES (?, ?, CURRENT_TIMESTAMP);";
./com/bmdyy/bluebird/controller/PostController.java:26:            jdbcTemplate.update(sql, text, userDetails.getId());
./com/bmdyy/bluebird/controller/AuthController.java:78:        String sql = "SELECT * FROM users WHERE id = ?";
<SNIP>
./com/bmdyy/bluebird/controller/ProfileController.java:109:        response.sendRedirect("/profile/edit?e=Details+updated!");
./com/bmdyy/bluebird/security/services/UserDetailsServiceImpl.java:21:            String sql = "SELECT * FROM users WHERE username = ?";`
```

## Visual Studio Code

Another more visual way to use these`RegEx`patterns is[Visual Studio Code](https://code.visualstudio.com/). We can use the`Search`feature by clicking on the`magnifying glass`on the left-hand side, entering the`RegEx`pattern, and then clicking the right-most button to enable`RegEx`search.

![Visual Studio Code interface with a search for SQL queries across multiple Java files. The welcome page offers options to start a new file, open a folder, and view recent projects. Walkthroughs for getting started and boosting productivity are available.](https://academy.hackthebox.com/storage/modules/188/searching/0.png)

## Results

Using either one of these methods, we can quickly identify the functions that use SQL queries. We can then go through them to try and identify ones that lack input sanitization.

#### Identifying the SQL Injection in /find-user

Using`grep`we can identify the following string concatenation in`IndexController.java`:

![Terminal output showing a search command for SQL queries with "WHERE" or "VALUES" in Java files. Result: IndexController.java:71 with a SQL query selecting users where the username matches a pattern.](https://academy.hackthebox.com/storage/modules/188/charbypass/0.png)

Taking a closer look at the code, we can see the function`findUser`which is mapped to`GET`requests to`/find-user`. This function takes a request parameter`u`. If this parameter matches the`RegEx`pattern contained in the variable`p`or contains a`space`, then the error message`"Illegal Search term"`is returned, otherwise`u`is used in a`SQL`query that populates the`users`list, which is then used as a`model attribute`when rendering`find-user`.

![Visual Studio Code showing IndexController.java. The method findUser handles user search with SQL query SELECT * FROM users WHERE username LIKE ?. It includes input validation and error handling.](https://academy.hackthebox.com/storage/modules/188/charbypass/1.png)

Looking at`resources/templates/find-user.html`we can see what happens with the`users`attribute;[Thymeleaf](https://www.thymeleaf.org/)loops through the list and prints out the`Id`,`Name`,`Username`and`Description`values of the`user`object.

![HTML template for displaying user search results. Iterates over users, showing user ID, name, username, and description using Thymeleaf expressions.](https://academy.hackthebox.com/storage/modules/188/charbypass/2.png)

We will want to look into this later, as this is a clear SQL injection vulnerability.

#### Identifying the SQL Injection in /forgot

Using grep again, we come across the following line in`AuthController.java`where user-input is concatenated into a`SQL`query.

```
icantthinkofaname23@htb[/htb]`$grep-nrE'(WHERE|VALUES).*" +'../controller/AuthController.java:134:               String sql = "SELECT * FROM users WHERE email = '" + email + "'";
<SNIP>`
```

Opening up`AuthController.java`, we can see that this line happens in the`forgotPOST()`function which handles`POST`requests to`/forgot`and the`email`variable is user-input (`@RequestParam String email`) that is validated against a`RegEx`pattern before being used in the query.

Code:java```
`// AuthController.java (Lines 121-164)@PostMapping({"/forgot"})publicStringforgotPOST(@RequestParamStringemail,Modelmodel,HttpServletRequestrequest,HttpServletResponseresponse)throwsIOException{if(email.isEmpty()){response.sendRedirect("/forgot?e=Please+fill+out+all+fields");returnnull;}else{Patternp=Pattern.compile("^.*@[A-Za-z]*\\.[A-Za-z]*$");Matcherm=p.matcher(email);if(!m.matches()){response.sendRedirect("/forgot?e=Invalid+email!");returnnull;}else{try{Stringsql="SELECT * FROM users WHERE email = '"+email+"'";Useruser=(User)this.jdbcTemplate.queryForObject(sql,newBeanPropertyRowMapper(User.class));Longvar10000=user.getId();StringpasswordResetHash=DigestUtils.md5DigestAsHex((""+var10000+":"+user.getEmail()+":"+user.getPassword()).getBytes());var10000=user.getId();StringpasswordResetLink="https://bluebird.htb/reset?uid="+var10000+"&code="+passwordResetHash;logger.error("TODO- Send email with link ["+passwordResetLink+"]");response.sendRedirect("/forgot?e=Please+check+your+email+for+the+password+reset+link");returnnull;}catch(EmptyResultDataAccessExceptionvar11){response.sendRedirect("/forgot?e=Email+does+not+exist");returnnull;}catch(Exceptionvar12){StringipAddress=request.getHeader("X-FORWARDED-FOR");if(ipAddress==null){ipAddress=request.getRemoteAddr();}if(ipAddress.equals("127.0.1.1")){model.addAttribute("errorMsg",var12.getMessage());model.addAttribute("errorStackTrace",Arrays.toString(var12.getStackTrace()));}else{model.addAttribute("errorMsg","500 Internal Server Error");model.addAttribute("errorStackTrace","Something happened on our side. Please try again later.");}return"error";}}}}`
```

Note: If you don't know what`controllers`are, you can imagine them as API endpoints.

In the case of SQL errors,`Spring`writes error messages to`STDOUT`, but in this case we can see explicit exception handling was defined. In most cases we get a typical`500 Internal Server Error`response from the server, but if our client IP address matches`127.0.1.1`then it looks like a stacktrace is passed to the`Thymeleaf`template.

This is something we will want to look into later, as SQL error output can be very useful.

#### Identifying the SQL injection in /profile

Using another one of the`RegEx`patterns with`grep`, we come across the following`SQL`query. In this case`user.getEmail()`is used in the query unsafely, but we have to take a closer look to see what the value this function returns is and more importantly whether we can manipulate it or not.

```
icantthinkofaname23@htb[/htb]`$grep-nrE'.*sql.*"'.<SNIP>
./BOOT-INF/classes/com/bmdyy/bluebird/controller/ProfileController.java:40:      sql = "SELECT text, to_char(posted_at, 'dd.mm.yyyy, hh:mi') as posted_at_nice, username, name, author_id FROM posts JOIN users ON posts.author_id = users.id WHERE email = '" + user.getEmail() + "' ORDER BY posted_at DESC";
<SNIP>`
```

Looking inside`ProfileController.java`, we find that this line is within the`profile()`function which is mapped to`GET`requests to`/profile/{id}`.

Code:java```
`// ProfileController.java (Lines 28-47)@GetMapping({"/profile/{id}"})publicStringprofile(@PathVariableintid,Modelmodel,HttpServletResponseresponse)throwsIOException{Stringsql;Useruser;try{sql="SELECT username, name, description, email, id FROM users WHERE id = ?";user=(User)this.jdbcTemplate.queryForObject(sql,newObject[]{id},newBeanPropertyRowMapper(User.class));}catch(Exceptionvar8){response.sendRedirect("/");returnnull;}sql="SELECT text, to_char(posted_at, 'dd.mm.yyyy, hh:mi') as posted_at_nice, username, name, author_id FROM posts JOIN users ON posts.author_id = users.id WHERE email = '"+user.getEmail()+"' ORDER BY posted_at DESC";Listposts=this.jdbcTemplate.queryForList(sql);model.addAttribute("user",user);model.addAttribute("posts",posts);UserDetailsImpluserDetails=(UserDetailsImpl)SecurityContextHolder.getContext().getAuthentication().getPrincipal();model.addAttribute("userDetails",userDetails);return"profile";}`
```

A quick glance over this code tells us that the`User`object referenced in the vulnerable SQL query is initialized just above with the results from another query. This is good news for us if we can find a way to influence those results, so we'll take a closer look at this later.
