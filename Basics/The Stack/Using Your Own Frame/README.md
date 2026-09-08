## USING YOUR OWN FRAME



In this level I had to write a function that reserves a 256-byte scratch
table on the stack, clears it to zero, then scans a buffer (pointer in
`rdi`, length in `rsi`) and marks a slot for every distinct byte value it
sees. The final step counts how many slots ended up marked and returns
that count in `rax`.


My first attempt stalled because I didn't have clear enough footing on
how to actually walk a buffer byte by byte and translate each byte's
*value* into a table *index*, that mapping is the whole trick of the
challenge, and without it clear in my head the loop logic kept falling
apart. After going back over the fundamentals, I settled on a structure
with three separate loops:


- `loop_1` — walk all 256 slots of the table and zero them out
  
- `loop_2` — walk the input buffer, and for each byte read, mark the
  corresponding slot in the table as `1`
  
- `loop_3` — walk the table one more time, counting how many slots ended
  up marked, accumulating that count into `rax`
  

One specific bug along the way: my boundary check in `loop_2`,
`cmp rcx, rsi`, wasn't stopping the loop where it should have. The issue
was an off-by-one — the last valid byte in the buffer sits at index
`rsi - 1`, not `rsi`, so comparing my index directly against `rsi` let
the loop run one iteration too far. I fixed it by doing `dec rsi` once,
before the loop starts, so the comparison lines up with the buffer's
actual last valid index instead of one past it.


Took a few hours of genuinely working through the logic rather than
pattern matching my way in, but got there and the flag came out clean.
