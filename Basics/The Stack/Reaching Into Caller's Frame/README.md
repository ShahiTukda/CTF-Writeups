## REACHING INTO CALLER'S FRAME



In this level I had to build a shared library with a `solve` function that
gets called from a main program. The flag is physically copied onto the
stack at `[rsp+0x40]`, and the job was to read it from there and get
`write` to print it out.


My first attempt used:

```
mov rsi, [rsp+0x40]
```

which didn't work. My first theory was an offset problem, since `main`
calls `solve` through a `call` instruction, the return address gets
pushed onto the stack first, pushing `rsp` down by 8 bytes. So the flag's
actual position, relative to `solve`'s own `rsp`, should have shifted
from `0x40` to `0x48`. I changed the offset accordingly. Still didn't
work.


That's when it clicked that the real problem wasn't the offset at all —
it was the instruction itself. `mov rsi, [rsp+0x40]` doesn't hand `write`
a pointer to the flag; it *dereferences* that address and loads whatever
8 bytes are sitting there straight into `rsi`. Since the flag is
physically stored at that stack location rather than pointed to from
there, those 8 bytes are just the first 8 characters of the flag itself,
misread as a garbage address. `write` was trying to read from wherever
that "address" happened to point, not from the flag.


What I actually needed was the address of that stack location, not its
contents — so I switched to `lea`:

```
lea rsi, [rsp+0x40]
```

This printed the flag, but the first 8 bytes were missing. That was the
tell that my earlier `0x48` theory had been wrong the whole time — the
challenge's `[rsp+0x40]` was already correct as given, computed from
inside `solve`'s frame after the `call` had already happened. My "fix"
for the return address was solving a problem that didn't exist, and it
only became visible as its own bug once the real `mov`/`lea` issue was
fixed underneath it. Switching back to `0x40` with `lea` gave the
complete flag.
