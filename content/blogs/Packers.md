---
title: "Packers Packing"
date: 2025-05-05
draft: false
---


Packing is like wrapping a program in layers to hide its contents. Malware authors use packers...


**Packing** is like wrapping a program in layers to hide its contents. Malware authors use packers to **compress or encrypt** the original program and add a small **unpacking stub**. The stub is a tiny piece of code that runs first. When the packed file is executed, the stub **decompresses** (or decrypts) the real malicious code into memory and then hands control to it ([courses.cs.umbc.edu](https://courses.cs.umbc.edu/undergraduate/CMSC491malware/CMSC%20449%20-%20Lec3%20-%20Hashing%20and%20Packing.pdf#:~:text=%EF%82%A7%20Compress%20original%20program%20and,into%20memory%20and%20runs%20it) ,[redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=Software%20packers%20function%20by%20compressing,a%20decoder%20stub%20for%20decompression)). This means on disk you only see a wrapper, not the actual malware. Packing makes static analysis very hard, because the real code (and its strings or import table) stays hidden until runtime.

```mathematica
Original Program (on disk) 
  +------------+-----------+
  | .text (code) | .idata  |
  | .rdata (.data)         |
  | ... (other sections)   |
  +------------------------+
  Entry point here ⬇︎

Packed Program (on disk after packing)
  +----------------------+
  | Stub (decompressor)  |  <-- contains code to unpack
  | Compressed data      |  (the original code encrypted)
  | (filler/empty space) |
  +----------------------+
  Entry point is at Stub ⬇︎

```

‍

**Step 1 – Packing:**  The packer takes the original executable (with its code, data, imports, etc.) and **compresses or encrypts** its contents. It then creates a new executable with a **new header and stub**. For example, UPX (a common packer) names its sections `UPX0`​, `UPX1`​, etc., instead of `.text`​ or `.data`​[redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=The%20next%20image%20shows%20sections,109%20Image%3A%20How%20to)​[medium.com](https://medium.com/ax1al/packing-and-obfuscation-fe6b03bbc267#:~:text=Upx%20is%20commonly%20used%20packer,3%20main%20part%20which%20are). The original entry point (OEP) of the program is replaced by the packer stub. This stub is a tiny loader program whose job is to restore the original program in memory[7orvs.github.io](https://7orvs.github.io/tutorials%20summaries/packing-notes-part1/#:~:text=A%20,to%20decrypt%20the%20packed%20file) [courses.cs.umbc.edu](https://courses.cs.umbc.edu/undergraduate/CMSC491malware/CMSC%20449%20-%20Lec3%20-%20Hashing%20and%20Packing.pdf#:~:text=%EF%82%A7%20Compress%20original%20program%20and,into%20memory%20and%20runs%20it).

‍

```mathematica
Packing process:
  (Original EXE) --pack--> [Stub + EncryptedData] --save--> (Packed EXE)
```

‍

**Step 2 – Stub Execution:**  When you run the packed executable, the operating system loads this stub into memory and begins executing it. The stub **allocates memory**, **decompresses or decrypts** the original program into that space, and **rebuilds the import table and other metadata**. In other words, it “unzips” the malicious payload in RAM. For instance, UPX’s stub will call functions like `VirtualProtect`​, `LoadLibraryA`​, and `GetProcAddress`​ to set permissions and resolve imports [medium.com](https://medium.com/ax1al/packing-and-obfuscation-fe6b03bbc267#:~:text=After%20unpacking%20the%20original%20code,the%20help%20of%20debugger) [redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=In%20this%20example%2C%20PEiD%20confirms,small%20number%20of%20text%20strings). Finally, the stub jumps to the now-unpacked original entry point (OEP), transferring control to the real malware code[7orvs.github.io](https://7orvs.github.io/tutorials%20summaries/packing-notes-part1/#:~:text=And%2C%20the%20unpacking%20stub%20performs,three%20steps) [redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=The%20next%20step%20is%20to,return%20to%20the%20program%E2%80%99s%20OEP).

‍

```mathematica
Runtime unpacking (in memory):
  [OS loads packed EXE] ---> [Stub runs]
      |
      v
  Stub does:
    - Allocate memory space 
    - Decompress original code into it 
    - Fix up Import Table (using LoadLibrary/GetProcAddress)
    - Jump to Original Entry Point

  After this:
  [Memory: Original code runs]
```

‍

Because of all this, static disassembly or decompilation on the packed file just shows the stub code and garbage data. The actual malicious logic is hidden. That’s why experts say packers  *“make static analysis difficult as the original program will be obfuscated in packed form”*​[redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=The%20use%20of%20packers%20makes,it%20remains%20in%20packed%20form).

### Dynamic Analysis and Unpacking

In contrast, **dynamic analysis** (running the malware in a controlled environment) can reveal the unpacked code. When the packed malware is executed (for example in a virtual machine or sandbox), the unpacking stub decompresses the original code into memory. At that point the real malware code and its imports exist in memory, and one can inspect them.

Practically, an analyst might set a breakpoint at the Original Entry Point. Since the stub transfers control there after unpacking, hitting that breakpoint means unpacking is done. Tools like OllyDbg or x64dbg can be used to dump the process memory after the stub finishes. Another trick is to break on memory-protection APIs (`VirtualProtect`​) that stub uses, because right after that call the real code is unpacked. Once dumped, the file in memory looks like the original (or close to it) and can be statically analyzed further[redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=The%20next%20step%20is%20to,return%20to%20the%20program%E2%80%99s%20OEP)​[7orvs.github.io](https://7orvs.github.io/tutorials%20summaries/packing-notes-part1/#:~:text=And%2C%20the%20unpacking%20stub%20performs,three%20steps).

‍

```mathematica
Runtime Flow Diagram:
Packed EXE (disk)  -->  Stub runs (allocates memory, unpacks code, fixes imports)  -->  Original code executes in memory

                                Runtime
  [Disk]               [Memory after unpack]               
  +------------+        +------------------+                  
  | Stub code  | -----> | Unpacked .text   |  (Original code) 
  | Compressed |        | Unpacked .data   |  (Data restored)  
  | data blob  |        | IAT rebuilt      |  (Imports resolved) 
  +------------+        +------------------+                  
```

‍

Dynamic analysis tools may still be tricked if the packer includes anti-debugging checks. Some packers check for debuggers/virtual machines (by looking at CPU flags, special hardware ports, or timing differences) and will behave differently if detected. Others deliberately generate code that confuses disassemblers (like overlapping instructions or self-modifying code). These techniques are more advanced layers: for example, a packer might execute a few real instructions, then jump backwards and execute them differently, foiling linear disassembly[7orvs.github.io](https://7orvs.github.io/tutorials%20summaries/packing-notes-part1/#:~:text=A%20,to%20decrypt%20the%20packed%20file).

‍

### Anti-Debug and Anti-Disassembly Tricks

Beyond simply compressing code, many malware packers also add **anti-analysis features**:

* **Debugger Detection:**  The unpacking stub might call Windows APIs like `IsDebuggerPresent()`​ or check the **PEB BeingDebugged flag**. It can deliberately trap single-step or breakpoints. For example, it might execute an illegal instruction (`INT 3`​) inside a try-catch block to see if a debugger catches it.
* **Junk and Overlapping Code:**  The stub may contain garbage bytes or misleading jumps so that static disassemblers can’t parse straight through. An example is using `JMP`​ to skip over random bytes or creating code that only makes sense when executed sequentially. This confuses tools like IDA or Ghidra.
* **Anti-VM/Sandbox Checks:**  Some packers query hardware or OS properties (e.g. disk serial numbers, MAC addresses, or CPU timestamp counters) that behave differently under virtualization. They might not unpack fully if they suspect a sandbox.
* **Self-Modification:**  The stub might decrypt or modify parts of its own code at runtime in small chunks, so even stepping through with a debugger is harder without knowing exactly where.

These tricks **only slow down** analysis; they cannot hide the code forever. Eventually, if the code runs, an analyst can trace through the unpacking stub, dump memory, and recover the payload[7orvs.github.io](https://7orvs.github.io/tutorials%20summaries/packing-notes-part1/#:~:text=And%2C%20the%20unpacking%20stub%20performs,three%20steps)​[redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=The%20next%20step%20is%20to,return%20to%20the%20program%E2%80%99s%20OEP). But they do make the process much more tedious.

### Layered Understanding

* **Beginner:**  At first glance, a packed malware file is like an **envelope** hiding a letter. You only see the envelope (stub + gibberish), not the letter inside (real code). You must open the envelope (run the unpacker) to read the letter.
* **Intermediate:**  Technically, the packer **replaces the PE header and sections**. On disk you lose the usual layout (.text, .data, .rdata). Everything goes into one or two sections. The stub then rebuilds the usual layout in memory.
* **Advanced:**  The unpacking stub is effectively doing **runtime linking**. It mimics what the Windows loader usually does for imports, but manually with `LoadLibrary/GetProcAddress`​. It also often calls `VirtualProtect`​ to change memory permissions (so the unpacked code can execute). After all fixes, it jumps to the original code’s start. At that point, the malware runs normally, but now *in memory* instead of from disk.

Throughout this process, packing **hides section names, strings, and import tables**. The original `.text`​ section becomes encrypted data. The `.rdata`​ (import) section may be minimal or gone. Analysts scanning the packed file statically see almost nothing of value: as noted,  *“few readable strings”*  and  *“imports resolved using runtime linking”*  are key signs of packing[courses.cs.umbc.edu](https://courses.cs.umbc.edu/undergraduate/CMSC491malware/Basic%20Static%20Analysis.pdf#:~:text=How%20Packers%20Work)​[redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=In%20this%20example%2C%20PEiD%20confirms,small%20number%20of%20text%20strings). Only by letting the program unpack itself (under careful observation) can one restore the original view.

In summary, packers wrap malware in a hidden layer. On disk you see only a stub that does the heavy lifting. When run, the stub unpacks the real code into memory and then lets it execute. Packing thwarts static analysis (because code, strings, imports are hidden or scrambled[courses.cs.umbc.edu](https://courses.cs.umbc.edu/undergraduate/CMSC491malware/Basic%20Static%20Analysis.pdf#:~:text=17%20%EF%82%A7%20Malware%20authors%20want,code%20%EF%81%B1%20Strings%20%EF%81%B1%20Imports)​[redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=In%20this%20example%2C%20PEiD%20confirms,small%20number%20of%20text%20strings)) and can include anti-debug tricks. Dynamic analysis is the usual way to unveil the packed content: a debugger or unpacking tool follows the stub’s actions until the malware is revealed.

**Sources:**  Packing/unpacking concepts are well documented in malware analysis resources [courses.cs.umbc.edu](https://courses.cs.umbc.edu/undergraduate/CMSC491malware/CMSC%20449%20-%20Lec3%20-%20Hashing%20and%20Packing.pdf#:~:text=%EF%82%A7%20Compress%20original%20program%20and,into%20memory%20and%20runs%20it) [7orvs.github.io](https://7orvs.github.io/tutorials%20summaries/packing-notes-part1/#:~:text=And%2C%20the%20unpacking%20stub%20performs,three%20steps) [redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=Software%20packers%20function%20by%20compressing,a%20decoder%20stub%20for%20decompression) [courses.cs.umbc.edu](https://courses.cs.umbc.edu/undergraduate/CMSC491malware/Basic%20Static%20Analysis.pdf#:~:text=17%20%EF%82%A7%20Malware%20authors%20want,code%20%EF%81%B1%20Strings%20%EF%81%B1%20Imports). These explain the stub mechanism, the effect on PE sections, and why static tools see only the unpacker, not the real code. The challenges to static analysis (hidden imports, strings, section names) are likewise noted by experts [redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=The%20next%20image%20shows%20sections,109%20Image%3A%20How%20to) [redscan.com](https://www.redscan.com/news/redscan-labs-malware-unpacking-uncover-hidden-cyber-threats/#:~:text=In%20this%20example%2C%20PEiD%20confirms,small%20number%20of%20text%20strings) [courses.cs.umbc.edu](https://courses.cs.umbc.edu/undergraduate/CMSC491malware/Basic%20Static%20Analysis.pdf#:~:text=17%20%EF%82%A7%20Malware%20authors%20want,code%20%EF%81%B1%20Strings%20%EF%81%B1%20Imports).
