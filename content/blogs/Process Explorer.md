
---
title: "Process Explorer"
date: 2025-07-16
draft: false
---

**Process Explorer** is a free, powerful system Monitoring tool from Microsoft’s Sysinternals suite that goes far beyond the builtin Task Manager.

‍

![image](/blogs/process-explorer/image-20250710133028-2rbu61j.png)

To ensure Process Explorer always runs with administrative privileges:

1. **Right‑click** on `procexp64.exe`​ (or its shortcut) and select **Properties**.
2. Switch to the **Compatibility** tab.
3. Under **Settings**, check **Run this program as an administrator**.
4. Click **OK** to save.

‍

![image](/blogs/process-explorer/image-20250710140933-ywxjjlh.png)

‍

There are  more columns present here compared to task manager

![9bc95adc-2baf-4759-8fdf-ebc851f9e191](/blogs/process-explorer/9bc95adc-2baf-4759-8fdf-ebc851f9e191-20250710134603-fksr7hg.png)

```ascii
+---------------------------+-------------------------------------------------------------+
|  Column                   |  Description                                                |
+---------------------------+-------------------------------------------------------------+

|  Process Name             |  Executable’s base name (e.g. explorer.exe)                 |

|  PID                      |  Numeric ID for the process; used in logs & taskkill        |

|  User Name                |  Account running the process (SYSTEM, YourUser, etc.)       |

|  Description              |  PE header “File description” (e.g. “Windows Explorer”)     |

|  Company Name             |  PE version “Company” field (e.g. Microsoft Corporation)    |

|  Verified Signer          |  ✔ if signed by trusted authority; blank if unsigned       |

|  Version                  |  File version string (e.g. 10.0.19041.1)                    |

|  Image Path               |  Full on‑disk path (e.g. C:\Windows\System32\svchost.exe)   |

|  Image Type (64/32‑bit)   |  Indicates 64‑bit vs 32‑bit (WoW64) processes               |

|  Package Name             |  UWP app package family (e.g. Calculator_8wekyb3d8bbwe)     |

|  DPI Awareness            |  How it scales on high‑DPI screens (Unaware, Per‑Monitor)   |

|  Protection               |  Protected Process / PPL flags (antimalware, DRM, etc.)     |

|  Control Flow Guard       |  ✔ if CFG mitigation enabled in the binary                 |

|  Stack Protection         |  ✔ if /GS buffer‑overrun checks are enabled                |

|  Window Title             |  Title bar text (distinguish multiple UI windows)           |

|  Window Status            |  Minimized, Maximized, Hidden                               |

|  Session                  |  Terminal Services / RDP session ID                         |

|  Command Line             |  Full launch command with switches & paths                  |

|  Comment                  |  Free‑text notes (via Properties → Image → Comment)         |

|  Autostart Location       |  Registry or task entry where it auto‑starts                |

|  VirusTotal               |  # of VT engines flagging this file (requires VT lookup)    |

|  DEP Status               |  DEP mode (OptIn, OptOut, AlwaysOn, AlwaysOff)              |

|  Integrity Level          |  Low, Medium, High, System integrity level                  |

|  Virtualized              |  UAC registry/file I/O redirection for legacy apps          |

|  ASLR Enabled             |  ✔ if ASLR relocation applied at load time                 |

|  UI Access                |  ✔ if marked to send input to higher‑privilege windows     |

|  Enterprise Context       |  Enterprise roaming context ID (Windows 10+ corporate)      |

+---------------------------+-------------------------------------------------------------+

```

‍

Process Explorer’s **Color Selection** feature lets you assign and instantly recognize different process and object types by color

![image](/blogs/process-explorer/image-20250710140817-x9dziwo.png)

**What Each Category Means**

- **New Objects (bright green):**  Processes or threads that just started.
- **Deleted Objects (red):**  Processes or threads that are terminating.
- **Own Processes (Cyan ):**  Processes started by the current user.
- **Services (peach):**  Any `svchost.exe`​ or Windows service host.
- **Suspended Processes (gray):**  UWP or other apps that Windows has paused.
- **Packed Images (purple):**  EXEs or DLLs compressed/obfuscated with packers (potentially suspicious).
- **Relocated DLLs (brown):**  Modules that had to be rebased in memory.
-  **.NET Processes (yellow):**  Managed‑code applications running on the CLR.
- **Immersive Processes (cyan):**  Modern UWP “Store” apps.
- **Protected Processes (fuchsia):**  PP/PPL processes that Windows hard protects against tampering.

‍

**lets look at the some of them:** 

1. ![image](/blogs/process-explorer/image-20250710140848-qxgp9cn.png)

Let's look at this Cyan colour, cyan colour is used for UWP applications, these applications are the ones whome you can suspend when you minimize them.

To be more precise, cyan color is about process that are hosting the windows runtime, they use the windows runtime api, which are the ones that is used to built UWP applications. But it can be used for other purposes as well. and thats why we can see something like windows explorer as well

‍

**What is UWP?**

![image](/blogs/process-explorer/image-20250710141640-wkr3ihx.png)

[https://learn.microsoft.com/en-us/windows/uwp/get-started/universal-application-platform-guide](https://learn.microsoft.com/en-us/windows/uwp/get-started/universal-application-platform-guide)

[https://en.wikipedia.org/wiki/Universal_Windows_Platform](https://en.wikipedia.org/wiki/Universal_Windows_Platform)

‍

![image](/blogs/process-explorer/image-20250710141308-nn8g46w.png)

‍

when we minimize calculator, it gets suspended and the colour is now grey.

![image](/blogs/process-explorer/image-20250710142509-2dxbmko.png)

The Calculator application included with Windows is a Universal Windows Platform (UWP) application.

**UWP apps are designed for multiple Windows devices,**  This means the Calculator can run on PCs, tablets, and other Windows device.

‍

![image](/blogs/process-explorer/image-20250710142359-jnw5h8m.png)

But if we look at explorer.exe it is also shown here with  the cyan colour, but it doesnt mean if i minimize the window of windows explorer and it becomes suspended. It is in cyan colour because it is using the windows runtime api (**winRT**)

let's go even more precise, the api behind covers is called IsImmersiveProcess, this was created on windows 8 days when these kind of applications started to appear. And in windows 8 days those applications were called as **metro applcations** back then, they were always fullscreen, so we had that kind of immersive experience this is where the function name isImmersiveProcess is coming from, thats what these cyan color reprtesents

‍

![image](/blogs/process-explorer/image-20250710145117-fouejdq.png)

‍

**Then vs. Now**

- **Windows 8:**  All Metro apps ran true full‑screen, no window borders—hence “immersive.”
- **Windows 10/11:**  UWP apps (the Metro successors) can run in normal windows, snapped side‑by‑side, or full‑screen.

‍

**What** **​`IsImmersiveProcess`​**​ **really tests**

- It doesn’t check “is this window full screen right now?”
- It checks “does this process load the Windows Runtime (WinRT) APIs designed for Metro/UWP apps?”
- If **yes**, you get cyan, even if the app runs in a little window on your desktop. LIKE CALCULATOR

‍

‍

Explorer.exe isn’t a Store/UWP app in the sense that you installed it from the Microsoft Store. It’s still a classic Win32 program, but under the hood it now loads Windows Runtime (WinRT) components (for its modern XAML UI, notifications, search APIs, etc.).

Because **Process Explorer**’s cyan highlight is purely driven by the **IsImmersiveProcess** check (i.e. “does this process host the WinRT engine?”), Explorer.exe comes up **TRUE** and so it gets painted cyan even though it isn’t a fullscreen “immersive” app and won’t suspend when minimized. It’s simply using the same runtime layer that UWP/Store apps use.

‍

![image](/blogs/process-explorer/image-20250712103640-hb7ber4.png)

Let's Look at .Net processes

‍

2. ![image](/blogs/process-explorer/image-20250712103833-xx7y91g.png)

    **Yellow** – the .NET processes. These are processes that use the Microsoft .NET Framework

‍

> Learn more about .NET\:[ https://youtu.be/6BcPIvVfVAw?si=348hGBl3syHfoK9B](https://youtu.be/6BcPIvVfVAw?si=348hGBl3syHfoK9B)
>
> ‍

Visual studio is a .Net application(? will see later), .net applications are those which are using  the .net clr are going to be shown in <span data-type="text" style="color: var(--b3-font-color12);">yellowish</span> color. and we can see the Microsoft  Visual studio process devenv.exe in <span data-type="text" style="color: var(--b3-font-color12);">yellowish</span> color

![image](/blogs/process-explorer/image-20250712104155-xp7jv2k.png)

‍

![image](/blogs/process-explorer/image-20250712105246-1y0206b.png)

> **devenv.exe** loads clr.dll inside itself → that’s why it’s marked as a “.NET process.”

When **clr.dll** is loaded into a process, Process Explorer knows “aha this process is running .NET code,” so it shows you the  **“.NET Assemblies”**  and  **“.NET Performance”**  tabs to let you inspect which managed libraries are loaded and how the CLR is behaving at runtime.

‍

**But what is CLR?** 

https://www.youtube.com/watch?v=3WJ1mRGP4So

‍

**Double click on devenv. exe**

![image](/blogs/process-explorer/image-20250712111300-axjf3p7.png)

**For a .NET Process you will usually see  tabs called .NET assemblies and .NET performance**.

‍

 **.NET Assemblies**

The  **.NET Assemblies** tab is shown for process that use the .NET Framework.  
 With .net assemblies you can see the list of all modules present into the addressspace of the process from a .NET perspective, The term ***assemblies***  means .NET module. This tab displays all the **AppDomains** in the process, along with the names of the assemblies loaded in each.  The flags and the full path to the assembly’s executable image are also  shown.

![e5bc94ff-9f10-423c-b093-38a30227618a](/blogs/process-explorer/e5bc94ff-9f10-423c-b093-38a30227618a-20250712123228-8po2n1e.png)

‍

 **.NET Performance Tab**

Only for **managed processes** (same requirement: it must be running the .NET CLR).

.Net performance shows a bunch of performance counters as they relate to various categories which might be interesting for people trying to diagnos .net issues in process and this view is updating every seonds with new data if that is applicable.

>  An **AppDomain** (short for **Application Domain**) is a lightweight, in‑process “sandbox” that the .NET CLR uses to isolate groups of assemblies from one another.

‍

![image](/blogs/process-explorer/image-20250713145902-k9xqe2g.png)

So, for .NET processes you will alwas see these two tabs, for processes which are not .NET you wont see these two tabs.

[https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)

‍

3. ![image](/blogs/process-explorer/image-20250713134115-hh06w7l.png)

  **Why does our  computer run so many svchost.exe?** 

![image](/blogs/process-explorer/image-20250713140438-y8v6eys.png)

When you look at Process Explorer, Peach highlighted entries are **Windows Services**. In general software terms “service” can mean many things, but in Windows it refers specifically to a background process that:

- **Starts automatically** when the system boots
- **Runs under special built‑in accounts** (LocalSystem, NetworkService, or LocalService)
- Is **managed by the Service Control Manager** (SCM), which can start, stop, pause or resume services on demand

‍

**What is Svchost.exe?** 

[https://www.linkedin.com/pulse/what-svchostexe-faeza-ahammed-kotta/](https://www.linkedin.com/pulse/what-svchostexe-faeza-ahammed-kotta/)

‍

**Svc host is  a generic Process.** 

A **generic process** means a **common or shared process** used to run different tasks or services.

In the case of `svchost.exe`​:

- It’s not made for ​**one specific task**​.
- Instead, it's a **general-purpose process** that can ​**host multiple Windows services**​.

‍

If you double click a service host process in Process Explorer, you’ll see a **Services** tab displaying exactly which Windows Services are running inside it. This tab only appears for processes that actually host services.

![image](/blogs/process-explorer/image-20250713140316-du9j9qc.png)

Windows services often rely on DLLs for their functionality. Since DLLs can't run on their own, svchost.exe provides the necessary executable environment to load and execute these DLL-based services.

‍

![image](/blogs/process-explorer/image-20250713141138-qjxsd8p.png)

You will notice **many** copies of **​`svchost.exe`​**​ in the process list. Microsoft uses this single, generic executable to host dozens of built in services each implemented as a DLL because:

- **Efficiency**  
  Creating a new process for every service would waste memory (each process needs its own address space, kernel object, handle table, etc.).
- **Grouping**  
  Related services can be bundled together under one `svchost`​​ instance.

​​

**However, this design has a tradeoff**  
if one DLL hosted service crashes or throws an unhandled exception, **the entire** **​`svchost.exe`​**​ **process** (and all services inside it) will terminate together. Memory leaks or runaway services become harder to diagnose, since you can’t tell at a glance which service inside the shared host is misbehaving.

Starting with Windows 10 build 1703 (and continuing into Windows 11 on systems with ≥ 3.5 GB RAM), Microsoft moved most services into **their own dedicated process** rather than group hosts. This improves reliability if one service fails or leaks memory, it only brings down its own process, not its neighbors.

In current Windows versions you’ll still see some grouped `svchost.exe`​​ processes, but often they now host **just a single service**. This gives you the best of both worlds, memory efficient grouping where it makes sense, and process per service isolation for greater robustness.

‍

**The very few Svchost are actually hosting multiple services for example:**   

![image](/blogs/process-explorer/image-20250713145436-zn4k277.png)

​​

![image](/blogs/process-explorer/image-20250713145311-vkrdcrd.png)

‍

4. ![image](/blogs/process-explorer/image-20250714124320-cgaqj9f.png)

The protected processes are shown in magenta/fuchsia color.

[https://web.archive.org/web/20090124094639/http://download.microsoft.com/download/a/f/7/af7777e5-7dcd-4800-8a0a-b18336565f5b/process_Vista.doc](https://web.archive.org/web/20090124094639/http://download.microsoft.com/download/a/f/7/af7777e5-7dcd-4800-8a0a-b18336565f5b/process_Vista.doc)

[https://medium.com/@boutnaru/the-windows-security-journey-protected-processes-6451a5fff277](https://medium.com/@boutnaru/the-windows-security-journey-protected-processes-6451a5fff277)

‍

Under proection tab

![image](/blogs/process-explorer/image-20250714131259-6jfspso.png)

‍

![image](/blogs/process-explorer/image-20250714131407-stzomcq.png)

[https://uberagent.com/blog/uberagent-7-preview-edr-antivirus-windows-defender-protected-process-performance-monitoring/](https://uberagent.com/blog/uberagent-7-preview-edr-antivirus-windows-defender-protected-process-performance-monitoring/?utm_source=chatgpt.com)

‍

5. ![image](/blogs/process-explorer/image-20250714133816-b6yvkxi.png)

‍

When we start a program, a process gets created, and it is shown in green colour indicating  a process has started

![image](/blogs/process-explorer/image-20250714133731-xfoalo7.png)

‍

And when a process gets terminated it is shown in Red.

![image](/blogs/process-explorer/image-20250714134010-97vky9m.png)

‍

**Tree View**

![image](/blogs/process-explorer/image-20250714134304-b4w5tjj.png)

The Process column of the window lists all the running processes in a  tree structure demonstrating the parent-child relationship of the  processes. For example, all the **svchost.exe** are child of **services.exe**. It also shows the icons of all the running processes.

‍

You can click the ▶/▼ icon next to a parent to show or hide its children and Double click on any of the child processes to see their threads, strings etc.

![image](/blogs/process-explorer/image-20250714135602-xfxc9v9.png)

Window's parent - child tree is **only** about who **created** whom, not about who **controls** whom at runtime. By default:

- When Process A spawns Process B, it simply calls `CreateProcess`​.
- After that, **A and B run completely independently**.
- If A crashes or is terminated, B keeps running just as before.

‍

**Explorer.exe**

if we see Some processes like Explorer.exe has no parent shown in process explorer

![image](/blogs/process-explorer/image-20250715133451-3x6yamp.png)

‍

![image](/blogs/process-explorer/image-20250715133358-mxyj5xn.png)

```mathematica
Parent: <Non-existent Process> (2140)
```

- Explorer was created by PID 2140
- But that PID doesn’t exist anymore (it's terminated)

‍

**What is Explorer.exe Process?** 

<u>*Explorer.exe  is the default Windows “GUI" shell*</u>. A **shell** is simply the layer that sits between you (the user) and the operating system’s core. It interprets your clicks and commands into OS actions, and presents back windows, menus, text, or graphics.

![image](/blogs/process-explorer/image-20250715134015-awqzn1i.png)

It’s the program that reads the **Shell** registry key under `Winlogon`​ and launches your desktop environment.

**GUI Shell (explorer.exe)**

- Provides desktop, taskbar, Start menu, File Explorer windows, system tray icons.
- You interact with it by clicking icons, dragging files, using menus.

‍

Once explorer.exe starts, **any process launched by the user via the Start Menu, Run box, desktop icons, etc.**  will appear as explorer.exe’s child.

![image](/blogs/process-explorer/image-20250715135019-qyafiyl.png)

```ascii
explorer.exe (parent)
 ├── msedge.exe
 ├── notepad.exe
 ├── OneDrive.exe	
 └── procexp64.exe
```

‍

**Userinit.exe (Parent of Explorer.exe)** 

```mathematica
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
 Userinit = C:\Windows\System32\userinit.exe --- Path
```

![image](/blogs/process-explorer/image-20250715135541-9b17j9u.png)

[https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-2000-server/cc939862(v=technet.10)?redirectedfrom=MSDN](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-2000-server/cc939862(v=technet.10)?redirectedfrom=MSDN)

> Specifies the programs that Winlogon runs when a user logs on. By  default, Winlogon runs Userinit.exe, which runs logon scripts,  reestablishes network connections, and then starts Explorer.exe, the  Windows user interface.

‍

### **1.**  **​`Winlogon.exe`​**​

- This is the system component that manages **user authentication** and logon events.
- Once login is successful, **it looks at the registry key**:

```mathematica
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
→ Userinit
```

‍

**It finds:**

```mathematica
C:\Windows\system32\userinit.exe,
```

‍

###  **2.**  **​`Userinit.exe`​**​

This is the **program specified by Winlogon**. It performs 3 main actions:

1. **Runs logon scripts** (if configured via Group Policy)
2. **Reconnects network drives** (e.g., mapped drives like `Z:\`​)

    > [ More on Network drive ](https://www.techtarget.com/whatis/definition/network-drive)
    >
3. **Starts the shell** (usually `explorer.exe`​)

‍

### 3. **​`Explorer.exe`​**​

- Started **by** **​`userinit.exe`​**​, not by `winlogon.exe`​​ directly.
- This is your **Windows desktop shell** — desktop, taskbar, Start menu, etc.

‍

**Lower Pane**

![image](/blogs/process-explorer/image-20250715193234-cb0zy17.png)

This section shows **all the DLLs**  that are currently **loaded into the memory space of** **explorer.exe**.

‍
