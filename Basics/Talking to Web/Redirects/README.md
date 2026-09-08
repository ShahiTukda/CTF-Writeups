## REDIRECTS



In this level the goal was straightforward on paper. Build a web server
that redirects an incoming request to another server. For some reason
x86-64 assembly is genuinely the language I feel most comfortable
building a web server in at this point, so that's what I used, the same
socket/bind/listen/accept skeleton from my earlier web server project,
this time just writing a redirect response instead of serving content.


For the actual redirect, I hand-packed the HTTP response bytes directly
into 8-byte chunks, pushed them onto the stack in reverse order so they'd
land in the right sequence in memory, then pointed `rsi` at the top of
the stack as the buffer for `write`.


```asm
mov rax, 0x0a0d0a0d30383a74
push rax
mov rax, 0x736f686c61636f6c
push rax
mov rax, 0x2e65676e656c6c61
push rax
mov rax, 0x68632f2f3a707474
push rax
mov rax, 0x68203a6e6f697461
push rax
mov rax, 0x636f4c200a0d7974
push rax
mov rax, 0x6c6e656e616d7265
push rax
mov rax, 0x502031303320312e
push rax
mov rax, 0x312f505454480000
push rax
mov rdi, r14
mov rsi, rsp
mov rdx, 73
mov rax, 1
syscall
```


This ran without crashing, but the target server's logs showed nothing
but 404s.


```
127.0.0.1 - - [24/Aug/2026 14:29:27] "GET / HTTP/1.1" 404 -
127.0.0.1 - - [24/Aug/2026 14:29:27] "GET /favicon.ico HTTP/1.1" 404 -
```


The bytes were being sent fine, the actual bug was upstream of the
assembly entirely. I'd assumed the redirect should point at the target
server's root (`/`), but the server I was supposed to redirect to didn't
expose anything at root at all, it only had a specific endpoint,
`/authenticate`. The `Location` header I'd hand-encoded was pointing
somewhere that was never going to return anything but a 404, no matter
how correct the byte-packing was.


Once I knew the real target, I re-encoded the response with the correct
`Location: http://challenge.localhost:80/authenticate` and adjusted the
length to match.


```asm
mov rax, 0x0a0d0a0d65746163
push rax
mov rax, 0x69746e6568747561
push rax
mov rax, 0x2f30383a74736f68
push rax
mov rax, 0x6c61636f6c2e6567
push rax
mov rax, 0x6e656c6c6168632f
push rax
mov rax, 0x2f3a70747468203a
push rax
mov rax, 0x6e6f697461636f4c
push rax
mov rax, 0x0a0d796c746e656e
push rax
mov rax, 0x616d726550206465
push rax
mov rax, 0x766f4d2031303320
push rax
mov rax, 0x312e312f50545448
push rax
mov rdi, r14
lea rsi, [rsp]
mov rdx, 88
mov rax, 1
syscall
```


Ran my redirect server alongside the port:80 target server in one
terminal, and the client in a third. This time the logs showed.


127.0.0.1 - - [24/Aug/2026 15:05:53] "GET /authenticate HTTP/1.1" 200 -
127.0.0.1 - - [24/Aug/2026 15:05:53] "GET /favicon.ico HTTP/1.1" 404 -


200 instead of 404, and the flag came through.
