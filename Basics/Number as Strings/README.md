## SUM THEM ALL



In this level I had to link my `atoi.so` and `itoa.so` shared libraries
(built over the last couple of levels, both handle positive and negative
values correctly) into an executable that sums `argv[1]` through
`argv[argc-1]` and prints the result.


The plan was straightforward: loop through the arguments, run each one
through `atoi`, keep a running sum in a register, then pass that sum into
`itoa` and write the result to stdout.


The structure was right and the loop correctly pulled and summed every
argument, but the program still wasn't working. The bug was in how I
called `itoa`, I never gave it a real buffer to write into. `rsi` was
still holding whatever value was left over from earlier in the loop
instead of pointing at valid writable memory, so `itoa` happily wrote its
digits out to a garbage address instead of somewhere I could actually
read from.


The fix was to actually allocate somewhere for it to write:


```
sub rsp, 32      ; carve out 32 bytes of real stack space
mov rsi, rsp      ; point rsi at that buffer before calling itoa
```


Then I reused that same address, `mov rsi, rsp` again — as the buffer
argument for the `write` syscall, so it read from exactly where `itoa`
had just written the digits. Ran it, and `/challenge/check` gave me the
flag.
