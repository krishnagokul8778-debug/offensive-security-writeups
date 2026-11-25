# ezpz — PentestGarage (Reversing, Hard, 300)

**Flag:** `ctf{__redacted__}` *(solved; value removed for public repo)*

**Target:** `172.31.45.248:30023` (lab instance, per-session)
**Binary:** `ezpz` — ELF 64-bit PIE, x86-64, dynamically linked, C++, **not stripped**.
BuildID `d30611fc4fee52e422f5ec38fe9548db5d4bc00a`.

## Summary
A custom **stack-based bytecode VM**. The bytecode program is *not* user-supplied — it's
embedded in `.data` (symbol `program`, 127 x uint16). `.init_array` runs `setup()`
(unbuffers stdio, sets `SIGALRM -> bye()`/`exit`, `alarm(30)`), then `main` builds a
`vector<unsigned short>` from `program` and runs `VM::execute()`.

The VM does a hardcoded license/password check, then opens and prints `/flag`.
Category says "memory corruption" but it's really pure reversing of the VM.

## VM internals (recovered from symbols + disasm)
Object layout: `stack<unsigned char>` (offset 0), program `vector<uint16>` @0x50,
PC iterator @0x68, `mem = new unsigned char[0x100]` @0x70. Ctor crashes if
`program.size() > 0x100`.

Instruction = 16-bit: low byte = opcode, high byte = operand `arg`.
Opcode jump table @0x6050:

| op | name        | behavior |
|----|-------------|----------|
| 0  | bye+add     | `bye(arg)` -> `exit(0xff)` |
| 1  | push        | push `arg` |
| 2  | pop         | pop |
| 3  | jmp         | PC = program.begin()+arg |
| 4  | read        | pop a,b -> `read(fd=arg, mem+a, b)`; push retval |
| 5  | write       | pop a,b -> `write(fd=arg, mem+a, b)`; push retval |
| 6  | stm         | pop v -> `mem[arg]=v` |
| 7  | ldm         | push `mem[arg]` |
| 8  | jne         | pop x,y; if x!=y -> jmp arg |
| 9  | xr          | xor |
| 10 | add         | add |
| 11 | open        | pop flags -> `open(mem+arg, flags)`; push fd |
| 12 | jl / 13 jg / 14 je | signed compare + conditional jmp |

`read`/`write` cap the transfer so `offset+count <= 255` (keeps it inside the 256-byte buf).

## Embedded program flow
1. Builds `"Enter password: "` in `mem[0..15]`, `write(1, mem+0, 16)`.
2. `read(0, mem+0x80, 16)` — user input at offset `0x80`.
3. Compare loop: `ldm[0x80+i]; push c; jne 102` for each char. Any mismatch jumps to
   index 102 = `bye`/exit. Expected bytes: `L a m O t 9 F h`.
4. On success: `write(1, "Correct: ", 9)`, store `"/flag\0"` in `mem[0..5]`,
   `open("/flag", 0)`, `read(fd=3, mem+0, 32)`, `write(1, mem+0, n)`.

## Password
`LamOt9Fh`

## Exploit
```python
import socket, time
s = socket.socket(); s.settimeout(10)
s.connect(("172.31.45.248", 30023))
s.recv(4096)                      # "Enter password: "
s.sendall(b"LamOt9Fh" + b"\x00"*8)  # pad to 16 so read() returns in one shot
time.sleep(1)
print(s.recv(4096).decode())
```
Output: `Correct: ctf{__redacted__}`

## Tooling note (Windows analysis)
No objdump/readelf/strings available; used Python `capstone` + `pyelftools`
(`pip install capstone pyelftools`) to parse the ELF, resolve PLT stubs via `.rela.plt`,
walk the opcode jump table, and decode the embedded bytecode.
