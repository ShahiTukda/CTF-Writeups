## BROKEN FUNCTION



In this module, across the past few levels, I got a lot more comfortable
with GDB, inspecting registers, examining memory, setting breakpoints,
and most usefully, writing full GDB scripts.


My first real use of scripting was on a SUID binary: break right before
its `read` call, then print the updated stack value and copy-paste it in
by hand to get the flag.


The level after that required skipping the whole `scanf` step entirely,
so the flag would print right after typing `continue`, no copy-pasting
needed. I set:


```
set *((uint64_t *) ($rbp-0x10)) = *((uint64_t *) ($rbp-0x18))
```


where `rbp-0x10` is my input slot and `rbp-0x18` is the value it checks
against, forcing the comparison to always succeed. I also pointed `rip`
to `main+695` to skip the `scanf` call outright. This worked, and it even
printed "Here is your flag:", but the actual flag never showed. The
cause was that my `catch syscall read` was firing on *every* read,
including the one that reads `/flag` off disk, and my patching logic was
stepping on that read too. The fix was adding an `if` check on the file
descriptor in `rdi` right before the read I actually wanted to affect,
that scoped the patch to only the read I was targeting and left the
`/flag` read untouched.


After that, since this was a SUID binary all along, I realized every
prior level in this module could've just been solved with
`call (void)win()`. But on the final level, `win` itself was broken.


Disassembling it showed:


```asm
mov QWORD PTR [rbp-0x8], 0x0
mov rax, QWORD PTR [rbp-0x8]
mov eax, DWORD PTR [rax]
```


That first line sets `[rbp-0x8]` to `0`, making it a null pointer. The
third line then dereferences that null pointer directly — an instant
segfault. I didn't catch this on my first pass through the disassembly
(I was too focused on the error-handling calls nearby, which turned out
to be irrelevant), but got there naturally once I looked closer.


I set a breakpoint right after the null gets written, before it's read
back into `rax`, and first tried:


```gdb
set *((uint64_t *) ($rbp-0x8)) = *((uint64_t *) $rsp)
```


This was wrong — the `*(...)` dereferences `$rsp`, reading whatever
garbage value happened to be sitting on the stack at that moment, rather
than giving me an address at all. What I actually needed was the address
itself:


```gdb
set *((uint64_t *) ($rbp-0x8)) = $rsp
```


This fixed the segfault — `call (void)win()` no longer crashed — but it
still didn't produce the flag. The reason: `call` in GDB is a "dummy
call," and it expects to run to completion and return control cleanly.
If any breakpoint fires during that dummy call — even one whose commands
just say `continue` — GDB abandons the call outright, because a
breakpoint firing mid-dummy-call breaks its ability to unwind properly
back to where I started.


The fix was `jump` instead of `call`. `jump` doesn't invoke anything the
way `call` does — it just relocates `$pc` to the given address and
resumes normal execution from there, so breakpoints behave exactly like
they would during any other `continue`. Running my script and then
`jump win` executed the now-patched `win` function normally and printed
the flag.
