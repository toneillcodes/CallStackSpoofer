# CallStackSpoofer

## Enhancements & Architectural Updates

This fork of the **VulcanRaven** project introduces enhancements to the original PoC, transitioning from a static, version-locked implementation to a **dynamic, portable evasion engine**. The primary focus is on build-agnostic stability and modularity.

## Key Features
### 1. Dynamic RVA Resolution (EAT Parsing)
The original project relied on hardcoded offsets (Magic Numbers) tied to specific Windows builds. If target DLLs were updated (e.g., via Windows Update), the offsets would break.
*   **Improvement:** Implemented a custom **Export Address Table (EAT) Parser** (`GetRvaFromName`).
*   **Impact:** The tool now resolves function locations at runtime. This makes the spoofer build-agnostic and significantly more stable across different versions of Windows 10 and 11.

### 2. Blueprint-Based Stack Profiling
Separated the definition of a call stack from the logic that constructs it.
*   **Improvement:** Introduced the `StackProfileEntry` structure. This allows users to define a target process's call stack using high-level metadata (Module Name + Function Name + Relative Offset) rather than raw hex deltas.
*   **Modularity:** New profiles (e.g., `explorer.exe` or `msedge.exe`) can be added by simply defining a new blueprint vector.

### 3. Automated Stack Construction Engine
Developed a factory function, `BuildDynamicStack`, to automate the transformation of a Blueprint into a functional `StackFrame` vector.
*   **Automation:** This engine handles module lookup, on-demand library loading, and final RVA calculations automatically.
*   **Reliability:** Included fallback loading mechanisms to ensure the spoofed thread always has a valid environment to execute within.

## Comparison
### Comparison at a Glance
| Feature | Original Implementation | This Version |
| :--- | :--- | :--- |
| **Offset Stability** | **Build-Dependent:** Hardcoded from Image Base | **Build-Agnostic:** Dynamic via EAT Parsing |
| **Configuration** | Manual `StackFrame` arrays | Modular `StackProfileEntry` Blueprints |
| **Maintenance** | High risk of crash on OS update | High reliability via runtime resolution |

## Usage
### LSASS Proof-of-Concept
The project now supports dynamic building of the `svchost` profile. 
The PoC will attempt to identify the LSASS process and open a handle to generate a sysmon event.  
Trigger the dynamic resolution with the `--svchost` option:
```bash
# Spoof using the dynamically resolved svchost blueprint
> VulcanRaven.exe --svchost


                             $$\
                             $$ |
        $$\    $$\ $$\   $$\ $$ | $$$$$$$\ $$$$$$\  $$$$$$$\         $$$$$$\  $$$$$$\ $$\    $$\  $$$$$$\  $$$$$$$\
        \$$\  $$  |$$ |  $$ |$$ |$$  _____|\____$$\ $$  __$$\       $$  __$$\ \____$$\\$$\  $$  |$$  __$$\ $$  __$$\
         \$$\$$  / $$ |  $$ |$$ |$$ /      $$$$$$$ |$$ |  $$ |      $$ |  \__|$$$$$$$ |\$$\$$  / $$$$$$$$ |$$ |  $$ |
          \$$$  /  $$ |  $$ |$$ |$$ |     $$  __$$ |$$ |  $$ |      $$ |     $$  __$$ | \$$$  /  $$   ____|$$ |  $$ |
           \$  /   \$$$$$$  |$$ |\$$$$$$$\\$$$$$$$ |$$ |  $$ |      $$ |     \$$$$$$$ |  \$  /   \$$$$$$$\ $$ |  $$ |
            \_/     \______/ \__| \_______|\_______|\__|  \__|      \__|      \_______|   \_/     \_______|\__|  \__|

                                       Call Stack Spoofer            William Burgess @joehowwolf

[+] Resolved C:\Windows\System32\kernelbase.dll!CtrlRoutine to RVA: CB892
[+] Resolved C:\Windows\System32\ntdll.dll!TpReleaseCleanupGroupMembers to RVA: D8C00
[+] Resolved C:\Windows\System32\kernel32.dll!BaseThreadInitThunk to RVA: 2E8D4
[+] Resolved C:\Windows\System32\ntdll.dll!RtlUserThreadStart to RVA: 8C531
[+] Initialising fake call stack...
[+] Created suspended thread
[+] Initialising spoofed thread state...
[+] Resuming suspended thread...
[+] Sleeping for 5 seconds...
[+] VEH Exception Handler called
[+] Re-directing spoofed thread to RtlExitUserThread
[+] Successfully obtained handle to lsass with spoofed callstack: 0000000000000110
[+] Check SysMon event logs to view spoofed callstack: Applications and Services --> Microsoft --> Windows --> Sysmon

>
```

## Future Improvements
### Dynamic Entry Resolution


# Original Readme
This repository demonstrates a PoC implementation to spoof arbitrary call stacks when making system calls. For a full technical walkthrough please see
the accompanying blog post here: https://labs.withsecure.com/blog/spoofing-call-stacks-to-confuse-edrs.

By default it contains three sample call stacks to mimic, which can be selected via supplying either `--wmi`, `--rpc`, or `--svchost`, as demonstrated below:

![readmeexample](https://user-images.githubusercontent.com/108275364/176182191-23cf273c-f154-411c-a1cf-428a9be323b9.PNG)

These call stacks were obtained by running SysMon with process access events enabled and searching for events where lsass was the target of the handle operation.

NB As a word of caution, this PoC was tested on the following Windows build:
 - 10.0.19044.1706 (21h2)

It has not been tested on any other versions and offsets may obviously vary (and hence break) on different Windows builds.

If you are having trouble, one technique to debug errors is to find the process generating OpenProcess events in SysMon and attach to it in 
WinDbg. Once attached, run `bp ntdll!NtOpenProcess` and when bp is hit run `knf`.

This will display output similar to the below, which will contain full symbol resolution and the correct stack utilisation space (indicated by the 'Memory' column):
```
0:003> knf
 #   Memory  Child-SP          RetAddr               Call Site
00           00000037`f01fe300 00007ffd`221d2ea6     ntdll!NtOpenProcess+0x12
01         8 00000037`f01fe308 00007ffd`1ffee959     KERNELBASE!ProcessIdToSessionId+0x96
02        80 00000037`f01fe388 00007ffd`23b99633     lsm!RpcOpenEnum+0x129
03       430 00000037`f01fe7b8 00007ffd`23b33711     RPCRT4!Invoke+0x73
04        40 00000037`f01fe7f8 00007ffd`23bfd77b     RPCRT4!Ndr64UnmarshallHandle+0xe1
05        70 00000037`f01fe868 00007ffd`23b7d2ac     RPCRT4!Ndr64StubWorker+0xb0b
06       6c0 00000037`f01fef28 00007ffd`23b7a408     RPCRT4!NdrServerCallAll+0x3c
07        50 00000037`f01fef78 00007ffd`23b5a266     RPCRT4!DispatchToStubInCNoAvrf+0x18
08        50 00000037`f01fefc8 00007ffd`23b59bb8     RPCRT4!RPC_INTERFACE::DispatchToStubWorker+0x1a6
09        e0 00000037`f01ff0a8 00007ffd`23b68a0f     RPCRT4!RPC_INTERFACE::DispatchToStub+0xf8
0a        70 00000037`f01ff118 00007ffd`23b67e18     RPCRT4!LRPC_SCALL::DispatchRequest+0x31f
0b        d0 00000037`f01ff1e8 00007ffd`23b67401     RPCRT4!LRPC_SCALL::HandleRequest+0x7f8
0c       110 00000037`f01ff2f8 00007ffd`23b66e6e     RPCRT4!LRPC_ADDRESS::HandleRequest+0x341
0d        a0 00000037`f01ff398 00007ffd`23b6b542     RPCRT4!LRPC_ADDRESS::ProcessIO+0x89e
0e       140 00000037`f01ff4d8 00007ffd`24ab0330     RPCRT4!LrpcIoComplete+0xc2
0f        a0 00000037`f01ff578 00007ffd`24ae2f26     ntdll!TppAlpcpExecuteCallback+0x260
10        80 00000037`f01ff5f8 00007ffd`23387034     ntdll!TppWorkerThread+0x456
11       300 00000037`f01ff8f8 00007ffd`24ae2651     KERNEL32!BaseThreadInitThunk+0x14
12        30 00000037`f01ff928 00000000`00000000     ntdll!RtlUserThreadStart+0x21
```
 As a note, the total stack space used by the current call site is indicated in the row below (e.g. ntdll!NtOpenProcess only takes up 8 bytes). As stated previously, a complete technical walkthrough is covered in the blog linked at the start of this readme which will explain this (and concepts like Child-SP) in more detail. The values generated by windbg can then be used to correlate with what is being returned by CalculateFunctionStackSize() in the event of any problems.

## Sysmon for LSASS Handles
Load Sysmon with the `lsassmontor.xml` configuration file.
```
C:\Tools\sysinternals>Sysmon64.exe -i c:\users\Administrator\Desktop\mystuff\VulcanRaven\lsassmonitor.xml


System Monitor v15.15 - System activity monitor
By Mark Russinovich and Thomas Garnier
Copyright (C) 2014-2024 Microsoft Corporation
Using libxml2. libxml2 is Copyright (C) 1998-2012 Daniel Veillard. All Rights Reserved.
Sysinternals - www.sysinternals.com

Loading configuration file with schema version 4.30
Sysmon schema version: 4.90
Configuration file validated.
Sysmon64 installed.
SysmonDrv installed.
Starting SysmonDrv.
SysmonDrv started.
Starting Sysmon64..
Sysmon64 started.

C:\Tools\sysinternals>
```

# Related Work
Thanks to the unicorn_pe project (https://github.com/hzqst/unicorn_pe) for example code in parsing UNWIND_CODEs.
