# COMMAND INJECTION


In this level I was given a server file in `/challenge` and had to spin it up in a second terminal while injecting a command into it from a client. Reading through the server's source, the vulnerability was plain enough: whatever I passed in as a `target` parameter got dropped straight into a template, `ls -l {target}`, and executed as a shell command. The catch was that the challenge had explicitly blocked every obvious character used to chain a second command onto the first, semicolons, ampersands, pipes, backticks, all filtered. Except for one that I had to find myself: the newline.

My first several attempts all failed for the same underlying reason, even though each one looked like a slightly different idea at the time. I tried sending `\ncat /flag` as the target, expecting the `\n` to be read as a newline and split my payload into two separate commands. The server's debug output showed exactly why this didn't work: `command='ls -l \\ncat /flag'`. That double backslash in the debug print is the tell, it means the string actually contained the two literal characters `\` and `n`, not a real newline control byte. Typing the two characters backslash-then-n into a URL doesn't magically become an escape sequence somewhere downstream; nothing in this pipeline was parsing my input for escape sequences, so `\n` just sat there as two ordinary characters inside a single `ls` argument, going nowhere near a
second command. Wrapping it in quotes, adding spaces around it, moving it to different positions in the string, none of that changed the core problem, because I was still sending literal text instead of an actual control byte.

At one point I tried sending just the letter `n` on its own, with the backslash dropped entirely. This wasn't really expected to work, it was more of a diagnostic step, to check whether the backslash character itself was being silently stripped out by the server's filter before it
even reached the command, which would have meant all my earlier attempts were functionally identical to just sending "n" the whole time. That told me the backslash was surviving fine; the problem wasn't filtering, it was that I'd been sending the wrong thing all along.

The actual breakthrough was realizing I needed to stop trying to type an escape sequence and instead send the real byte directly. A newline is ASCII `0x0A`, and in a URL that byte gets represented as the percent-encoded sequence `%0a`. Sending `target=+%0a+cat+/flag` produced a debug line that looked almost identical to my earlier attempts but was critically
different: `command='ls -l \n cat /flag'`, single backslash this time, meaning an actual newline character had landed inside the string, not two literal text characters pretending to be one.

Why that single byte was enough to fully bypass the filter comes down to how a POSIX shell treats a newline: functionally, it's a command separator, exactly like a semicolon. `ls -l\ncat /flag`, once that `\n` is a real control byte rather than text, is executed by the shell
exactly as if two separate lines had been typed and each one run in sequence, identical in effect to `ls -l; cat /flag`. The challenge had carefully filtered every character most people reach for first when chaining shell commands, but a raw newline does the exact same job and
had apparently been left off the list entirely.

Ran it, the server logged a clean `200`, and `cat /flag` executed right alongside the intended `ls -l`, printing the flag.
