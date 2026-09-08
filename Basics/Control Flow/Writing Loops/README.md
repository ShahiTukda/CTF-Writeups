## WRITING LOOPS



In this level I had to write a program that computes the length of
`argv[1]` and returns it as the exit code, inspecting the string one byte
at a time and using a backward jump to repeat the loop.


My first attempt read the string byte by byte and had the correct jump
instructions, but it still wasn't producing the right result.


After going back over the logic, I realized the program had everything it
needed except a way to actually count the characters, it was walking
through the string correctly but never kept track of how many bytes it
had seen.


To fix this, I set an `rdx` register to `0` before the loop starts, and
incremented it on every iteration where the current byte wasn't the null
terminator:


```asm
mov rdx, 0
mov rdi, [rsp+16]
loop:
cmp BYTE PTR [rdi], 0
je success
inc rdx
inc rdi
jmp loop
```


`rdx` only increments when the loop *doesn't* hit the terminator, so by
the time it jumps to `success`, `rdx` holds exactly the number of
characters read — the string's length. Ran `/challenge/check` and got
the flag.
