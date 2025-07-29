# Perplexed writeup

**Author:** 0xdf-sec  

## Challenge Overview

This challenge presented us with an ELF binary that required a correct password. The goal was to reverse-engineer the binary, understand how the password was validated, and extract the correct input required to pass the check.

## 1. Initial Reconnaissance

### 1.1 Disassembling with objdump

Since we have no source code, all information must come from the binary file itself. Let's disassemble it:

```bash
objdump -d perplexed > perplexed_disass.txt
```

In the assembly code, we found a particularly interesting function: `check()`

```assembly
0000000000401156 <check>:
 401156: 55                   push   %rbp
 401157: 48 89 e5             mov    %rsp,%rbp
 40115a: 53                   push   %rbx
 40115b: 48 83 ec 58          sub    $0x58,%rsp
 40115f: 48 89 7d a8          mov    %rdi,-0x58(%rbp)
 401163: 48 8b 45 a8          mov    -0x58(%rbp),%rax
 401167: 48 89 c7             mov    %rax,%rdi
 40116a: e8 d1 fe ff ff       call   401040 <strlen@plt>
 40116f: 48 83 f8 1b          cmp    $0x1b,%rax
 401173: 74 0a                je     40117f <check+0x29>
 401175: b8 01 00 00 00       mov    $0x1,%eax
 40117a: e9 20 01 00 00       jmp    40129f <check+0x149>
 40117f: 48 b8 e1 a7 1e f8 75 movabs $0x617b2375f81ea7e1,%rax
 401186: 23 7b 61 
 401189: 48 ba b9 9d fc 5a 5b movabs $0xd269df5b5afc9db9,%rdx
 401190: df 69 d2 
 401193: 48 89 45 b0          mov    %rax,-0x50(%rbp)
 401197: 48 89 55 b8          mov    %rdx,-0x48(%rbp)
 40119b: 48 b8 d2 fe 1b ed f4 movabs $0xf467edf4ed1bfed2,%rax
 ...
```

### 1.2 Analysis of the 'check' Function

**STEP 1: Length Verification**
Before any validation, the function ensures that user input is exactly 27 bytes long:

```assembly
0x40116f <check+0019> cmp    $0x1b,%rax    ; Compare input length with 27
0x401173 <check+001d> je     0x40117f      ; Continue if equal, else fail
```

- `%rax` holds the return value of `strlen(password)`, which must be `0x1b` (27 in decimal)
- If the length is not 27, the function exits immediately

**STEP 2: Constant Initialization**
The function initializes three 64-bit constants in memory for later validation:

```assembly
0x40117f <check+0029> movabs $0x617b2375f81ea7e1,%rax
0x401189 <check+0033> movabs $0xd269df5b5afc9db9,%rdx
0x40119b <check+0045> movabs $0xf467edf4ed1bfed2,%rax
```

These three constants are stored in little-endian order:
- `c1`: `0x617b2375f81ea7e1`
- `c2`: `0xd269df5b5afc9db9`
- `c3`: `0xf467edf4ed1bfed2`

### 1.3 Critical Discovery: Misaligned Memory Write

**The Key Insight:**
There's a crucial misaligned memory write that's easy to miss! In a typical scenario with three 64-bit constants, we'd expect sequential storage:

```
+-----------------------------------+
| Address Offset | Value (Expected) |
+-----------------------------------+
|   0x00–0x07    |   c1  (8 bytes)  |
|   0x08–0x0F    |   c2  (8 bytes)  |
|   0x10–0x17    |   c3  (8 bytes)  |
+-----------------------------------+
```

**However, this is NOT what happens in the check function.**

The third constant (`c3`) is written starting at offset 15 (`0x0F`) instead of offset 16 (`0x10`). This causes it to overwrite the last byte of `c2`:

```
+-------------------------------------------+
|  Address Offset  |     Value (Actual)     |
+-------------------------------------------+
|     0x00–0x07    |      c1 (8 bytes)      |
|     0x08–0x0E    |  First 7 bytes of c2   |
|    0x0F–0x16     |      c3 (8 bytes)      |
|    0x17–0x1A     |  Uninitialized (0x00)  |
+-------------------------------------------+
```

The last byte of `c2` is lost and replaced by the first byte of `c3`.

**This is extremely important** because if we assume the constants were stored normally, we'll derive the wrong password.

The actual memory layout is:

```
+-----------------------------------------+
|    Bytes    |         Hex Value         |
+-----------------------------------------+
|     0–7     |  e1 a7 1e f8 75 23 7b 61  |
|     8–14    |  b9 9d fc 5a 5b df 69     |
|    15–22    |  d2 fe 1b ed f4 ed 67 f4  |
|    23–26    |        00 00 00 00        |
+-----------------------------------------+
```

## 2. Reconstructing the Flag

With this crucial understanding, we can recover the real values and reconstruct the password:

```python
import struct

# The three constants (from the disassembled code)
c1 = 0x617b2375f81ea7e1
c2 = 0xd269df5b5afc9db9
c3 = 0xf467edf4ed1bfed2

# Pack each constant in little-endian format (each 8 bytes)
c1_bytes = struct.pack("<Q", c1)  # 8 bytes: bytes 0–7
c2_bytes = struct.pack("<Q", c2)  # 8 bytes: but only first 7 will be used
c3_bytes = struct.pack("<Q", c3)  # 8 bytes: stored starting at byte 15

# Construct the secret buffer according to the overlapping layout:
# • Bytes 0–7: c1_bytes (8 bytes)
# • Bytes 8–14: first 7 bytes of c2_bytes
# • Bytes 15–22: c3_bytes (8 bytes)
# • Bytes 23–26: 4 bytes of 0x00 (not explicitly set in the binary)
secret = c1_bytes + c2_bytes[:7] + c3_bytes + b'\x00'*4
assert len(secret) == 27

# Convert the 27-byte secret into a bitstring
bits = "".join(f"{byte:08b}" for byte in secret)
# Only use the first 189 bits (27×7)
bits = bits[:189]

# Reconstruct the 27-byte password by splitting into 27 groups of 7 bits
password = ""
for i in range(27):
    chunk = bits[i*7:(i+1)*7]
    char_val = int(chunk, 2)
    password += chr(char_val)

# Use repr() to show any non-printable characters (like the final null byte)
print(repr(password))
```

## 3. Execution and Result

Running the script:

```bash
$ python perplexed_script.py
'picoCTF{0n3_bi7_4t_a_7im3}\x00'
```
