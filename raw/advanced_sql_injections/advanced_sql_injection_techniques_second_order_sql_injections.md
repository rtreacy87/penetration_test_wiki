# Second-Order SQL Injection

## Introduction

`Second-order SQL injection`is a type of`SQL injection`where`user-input`is`stored`by the application and then later used in a`SQL`query unsafely. This type of vulnerability can be harder to spot, because it usually requires interacting with seperate application functionalities to store and then use the data.

## Expanding on the SQL injection in /profile

In this section we will take a closer look at the third SQLi vulnerability that we identified in the`Identifying Vulnerabilities`sections:

Code:java```
`// ProfileController.java@GetMapping({"/profile/{id}"})publicStringprofile(@PathVariableintid,Modelmodel,HttpServletResponseresponse)throwsIOException{Stringsql;Useruser;try{sql="SELECT username, name, description, email, id FROM users WHERE id = ?";user=(User)this.jdbcTemplate.queryForObject(sql,newObject[]{id},newBeanPropertyRowMapper(User.class));}catch(Exceptionvar8){response.sendRedirect("/");returnnull;}sql="SELECT text, to_char(posted_at, 'dd.mm.yyyy, hh:mi') as posted_at_nice, username, name, author_id FROM posts JOIN users ON posts.author_id = users.id WHERE email = '"+user.getEmail()+"' ORDER BY posted_at DESC";Listposts=this.jdbcTemplate.queryForList(sql);model.addAttribute("user",user);model.addAttribute("posts",posts);UserDetailsImpluserDetails=(UserDetailsImpl)SecurityContextHolder.getContext().getAuthentication().getPrincipal();model.addAttribute("userDetails",userDetails);return"profile";}`
```

In this function, two queries are made:

1. The first selects the`username`,`name`,`description`,`email`and`id`values from`users`where`id`matches`{id}`in the path. These values are then used to initialize a`User`object.
1. The second query selects`posts`made by the`user`whose`email`matches the`email`of the`User`object we just created.
Even though the`email`we use in the second query came from the database as a result of the first query, this is still unsafe since it is concatenated into the query. If we can find a way to set the value of`email`for a known`id`, then we should be able to exploit a`second-order SQL injection`.

So let's do a little bit of`input tracing`, and find out where/if we can set the value of`email`. To update a value in SQL, you have to use the`UPDATE`keyword, so we can grep for this in the project:

```
icantthinkofaname23@htb[/htb]`$grep-irnE'UPDATE.*email'com/bmdyy/bluebird/controller/ProfileController.java:70:               sql = "UPDATE users SET name = ?, description = ?, email = ?";
com/bmdyy/bluebird/controller/ProfileController.java:85:                  this.jdbcTemplate.update(sql, new Object[]{name, description, email, passwordHash, userDetails.getId()});
com/bmdyy/bluebird/controller/ProfileController.java:87:                  this.jdbcTemplate.update(sql, new Object[]{name, description, email, userDetails.getId()});`
```

The results show an`UPDATE`query which seems to include`email`. Taking a closer look, we find the line in`editProfilePOST()`which maps to`POST`requests to`/profile/edit`. This function lets us edit the user details of the user we are logged in as, including the`email`.

Code:java```
`@PostMapping({"/profile/edit"})publicvoideditProfilePOST(@RequestParamStringname,@RequestParamStringdescription,@RequestParamStringemail,@RequestParam(required=false)Stringpassword,@RequestParam(required=false)StringrepeatPassword,HttpServletResponseresponse)throwsIOException{<SNIP>sql="UPDATE users SET name = ?, description = ?, email = ?";<SNIP>sql=sql+" WHERE id = ?";<SNIP>this.jdbcTemplate.update(sql,newObject[]{name,description,email,userDetails.getId()});<SNIP>}`
```

At this point we have confirmed that we can control the value of email, and that this in turn will be passed into the vulnerable SQL query (`second-order`). One important thing to note is that in`editProfilePOST()`the`SQL`query to update the user's`email`is`parameterized`. There is no`SQL`injection vulnerability in this query.

## Exploiting Second-Order SQL Injection

With the vulnerability identified, exploiting`second-order SQL injection`is no different than regular`SQL injection`, except that setting the payload and 'running' the payload are`two`seperate requests.

First things first, we need to log into`BlueBird`and head on over to`/profile/edit`so that we can set our user's`email`.

**********![BlueBird edit profile page with fields for username, name, description, email, new password, and repeat new password. Includes "Update Details" button.](https://academy.hackthebox.com/storage/modules/188/second/0.png)Since our`email/payload`will be used in`/profile/{id}`, we need to keep in mind the`SQL`query that we will be injecting into:

Code:sql```
`SELECTtext,to_char(posted_at,'dd.mm.yyyy, hh:mi')asposted_at_nice,username,name,author_idFROMpostsJOINusersONposts.author_id=users.idWHEREemail='" + user.getEmail() + "'ORDERBYposted_atDESC`
```

Since a list of posts is returned, one option would be to use`union-based SQL injection`to easily exfiltrate data with a payload like:

Code:sql```
`'UNIONSELECT1,2,3,4,5--`
```

We can enter this payload and click`Update Details`to 'set' the payload, and then load our users`Profile`page so that it will be triggered.

![Terminal showing PostgreSQL log with a SQL error. Includes an attempted SQL injection using UNION SELECT 1,2,3,4,5--.](https://academy.hackthebox.com/storage/modules/188/second/3.png)

Unfortunately, this exact payload will result in a`SQL error`, but we can easily troubleshoot it. Taking another look at the error message in the log file we can see that`type character varying and integer cannot be matched`:

![Terminal showing PostgreSQL log with an error: "UNION types character varying and integer cannot be matched." Includes attempted SQL injection using UNION SELECT 1,2,3,4,5--.](https://academy.hackthebox.com/storage/modules/188/second/4.png)

What this means is that some of the columns are supposed to be`VARCHAR`and we tried to union with 5`INTEGER`values. There are multiple ways we can figure out which ones are supposed to be which, the easiest way being to look at the full query listed in the error message and deduce the types. The columns`text, posted_at_nice, username and name`are all likely to be`VARCHAR`whereas`author_id`is probably an`INTEGER`. We can test this theory by modifying our payload to`' UNION SELECT '1','2','3','4',5--`and trying again:

**********![BlueBird profile page for user "pentest" with sidebar navigation. Displays user details and security trends like SQL Injection. Includes "Edit Profile" button.](https://academy.hackthebox.com/storage/modules/188/second/1.png)
