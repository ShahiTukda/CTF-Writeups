## URL ENCODING



In this level I had to send an HTTP request to `/challenge/server` and
get a flag back. Reading the server's source, I found two requirements,
requests had to come from an `nc` client, and the target endpoint was
`access submission hack`, with literal spaces in it, which meant it
needed to be URL-encoded as `access%20submission%20hack` before sending.


My first attempt was typing the request directly into an interactive
`nc` session.


```bash
>> nc -v challenge.localhost 80
GET /access%20submission%20hack HTTP/1.1
Host: challenge.localhost:80
```


This came back as a clean `404 Not Found` from Werkzeug, a normal,
well formed HTTP response, not a malformed request error. That told me
the request syntax itself wasn't the problem. The server just wasn't
seeing the path I intended. Typing multi line raw HTTP by hand into an
interactive session gives no real visibility into the exact bytes being
sent, so I switched to building the request explicitly with `printf`,
piped straight into `nc`.


```bash
>> printf "GET /access%20submission%20hack HTTP/1.1\r\nHost: challenge.localhost:80\r\n\r\n" | nc -v challenge.localhost 80
```


This came back with a `400 Bad Request`, and the error message quoted
back exactly what the server had received:


"GET /access ubmission 0x0p+0ck HTTP/1.1"


That garbled text was the actual bug, and it had nothing to do with the
server or `nc` at all, it was `printf` itself. `printf`'s format string
treats any `%` followed by characters as the start of a conversion
specifier, the same as `%s` or `%d` would be in C. So `%20s` wasn't
being sent as the literal text "%20s", `printf` was reading it as
"format a string, padded to width 20," consuming the `s` immediately
after "20" as the conversion type, which is exactly why "submission"
came out as "ubmission", the leading `s` got eaten by the specifier
instead of being printed. `%20h` got mangled in its own way for the same
underlying reason. A URL encoded string is full of literal `%`
characters, and `printf` doesn't know or care that I meant them as text,
not formatting directives.


The fix was escaping the percent signs so `printf` treats them as
literal characters instead of specifiers, `%%` is how you print one
literal `%` in a printf format string:


```bash
>> printf "GET /access%%20submission%%20hack HTTP/1.1\r\nHost: challenge.localhost:80\r\n\r\n" | nc -v challenge.localhost 80
```


With every `%20` written as `%%20`, `printf` emitted the request exactly
as intended, the server matched the route, and I got the flag.
