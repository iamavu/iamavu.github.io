---
title: PMA - 0x02 - Basic Techniques
published: 2025-03-01
description: 'Notes from Practical Malware Analysis Section 0x02'
image: '/04-PMA-Basic-Techniques/banner.png'
tags: [PMA, Malware, Malware Analysis, Notes]
category: 'PMA-Notes'
draft: false 
---

# Static Analysis

## Antivirus Scanning

- Running it through multiple antivirus which may have already identified the malware, although they are not perfect
- They rely on identifiable pieces of suspicious code (file signatures) and behavior/pattern matching analysis (heuristics)
- [Virustotal](https://www.virustotal.com/) is great help here as it runs the malware through multiple antivirus systems

## Hashing : Fingerprinting a malware

- Hashing is common method where you run the malware through hashing programs
- They use algorithms such as MD5 or SHA-1 to produce unique hash (fingerprint)
- Hash then can be used as label or sharing it to other analyst to identify the malware or see if it has been already identified

## Finding Strings

- Strings can be great method to find texts within the malware
- ASCII and Unicode format are used to store strings
- They store by characters in sequence and ending with a NULL terminator to indicate that string is complete
- ASCII uses 1byte per character while Unicode uses 2bytes per character

![ascii](public/04-PMA-Basic-Techniques/ascii.png)

![unicode](public/04-PMA-Basic-Techniques/unicode.png)

- Sometimes if strings program identifies a sequence of characters which end with null terminator, it might think of it as string while it could be just some CPU instruction or memory address

## Packed and Obfuscated Malware

- Obfuscated malware are the one whose execution are hidden
- Packed malware is subset of obfuscated malware where the program is compressed making it harder to analyze
- When packed program is ran, a small wrapper program, it de-compresses the packed program and then executes unpacked program
- When packed program is analyzed statically, only wrapper program can be dissected

![obfuscated](public/04-PMA-Basic-Techniques/obfuscated.png)

- Packers can be detected using software such as PEiD
- Packed program must be unpack so that we can analyze it

## Portable Executable File Format

- PE format is used by windows executables, object code and DLLs
- It contains necessary information for the Windows OS loader to manage the wrapped executable code
- PE files begin with a header that includes information about the code, type of the application, required library functions and space requirements

## Linked Library and Functions

- Imports are function that are used by program whilst they are stored in another program, such as code libraries that contain common functionality which are connected by linking
- Code libraries can be linked statically, at runtime or dynamically
- Static linking is not used often although it’s common in UNIX programs
- When code is statically linked, all the code from the library are copied to our main executable making it grow in size which makes analyzing code harder
- Runtime linking is commonly used in malwares especially when it’s obfuscated or packed
- Some linked functions can be imported without being listed in program headers like `LoadLibrary`, `LdrGetProcAddress` , `LdrLoadDll`and `GetProcAddress`
- Dynamic linking is the most common method of linking, where OS searches for all necessary linked libraries when the program is loaded
- Libraries used and called are very important for us to understand what the program does
- Functions can also be imported by ordinals making it harder for us to analyze
- Below are some common DLLs

![dll](public/04-PMA-Basic-Techniques/dll.png)

- `Ex` is a suffix used when the function is updated by Microsoft
- `A` and `W` appearing at the end is extra information about suffix which doesn’t appear in actual documentation and is just there to tell us that function accepts ASCII string and word respectively
- Like imports, there are also exports, which are functions exported by programs so that other programs can import and utilize them, these are most common in DLLs

## PE File Headers and Sections

- `.text` section contains instructions code that CPU will execute
- `.rdata` section contains information about imports and exports, storing read only data
- `.data` contains global data accessible from anywhere in the program
- `.idata` stores data about import functions, usually not present
- `.edata` stores data about export functions, usually not present
- `.pdata` present only in 64bit applications storing exception-handling information
- `.rsrc` contains other data such as icons, images, menus and strings
- `.reloc` contains information about relocation of library files

## Some Tips and Trivia

- All Delphi programs use compile time of June 19, 1992
- Virtual size (space allocated for section during loading) and raw data (how big section is on disk) should be equal (small differences are fine), if they aren’t that means it’s a packed program

# Malware Analysis in Virtual Machines

## Introduction

- One can analyze malware in either physical machine or virtual machine.
- Physical machine gives advantage of malware behaving the same was as intended though as it is on air-gapped network malware communications with internet might be hampered
- Virtual machine solves this but there is possibility that malware might behave differently on virtual machine than physical one making analysis hard

## Structure of Virtual Machine

- A virtual machine is computer within a computer, allowing complete isolation of virtual machine from host machine

![physical-machine](public/04-PMA-Basic-Techniques/physical-machine.png)

- Using host-only network is common practice in VMs for malware analysis

![host-only](public/04-PMA-Basic-Techniques/host-only.png)

- Taking snapshots is important before you analyze any malware so you can return back to original state once you are done with your work

![snapshot](public/04-PMA-Basic-Techniques/snapshot.png)

# Basic Dynamic Analysis

## Introduction

- Dynamic analysis is performed after we have exhausted our static analysis
- It allows us to observe actual behavior of the malware
- It is important to know how to run a malware if you want to perform dynamic analysis. Quite often it can be simple as double clicking the exe
- DLLs might be hard to run, there is tool called `rundll32.exe` which comes with all modern version of windows which has following syntax `rundll32.exe DLLname, Export Arguments`. Where `Export` value must be a function name or ordinal selected from exported function table in DLL which can be viewed using tools such as PEBear etc. Example syntax for both would be `rundll32.exe mal.dll, install` or `rundll32.exe mal.dll, #5` where `install` is the export function name and `#5` is the ordinal number prepended with `#`
- Malicious DLL quite often run their code in `DLLMain` (called from the DLL entry point) and as `DLLMain`  is executed when DLL is loaded, we can force DLL to load via `rundll32.exe` to get information out of it
- One can also modify the PE header of DLL and change the extension and force windows to load DLL as EXE. To modify that, wipe the `IMAGE_FILE_DLL (0x2000)` flag from the characteristics field in the `IMAGE_FILE_HEADER` , thought it might cause malware to crash  or terminate but as long as the changes cause malware to execute it’s payload, we are good to go.
- DLL malware may also be needed to install as service with following syntax `rundll32.exe mal.dll, InstallService *ServiceName`* and then to start the service `net start *ServiceName`*
- When there isn’t a export function such as `Install` or `InstallService`  in the DLL, we may need to manually install the DLL as service via either Windows `sc` command or modifying the register for unused service and then using `net start` on that service. The service entries are located in `HKLM\SYSTEM\CurrentControlSet\Services`

## Monitoring With Process Monitor

- Process monitor or procmon is powerful tool to monitor certain registry, file system, network, process and thread activity although it should not be usually use to log network activity as it is inconsistent throughout windows versions.
- Procmon can monitor all system calls as soon as it is ran making it impossible to look through all of them as they are over in thousands and it may crash our virtual machine, so it is advised to load it up, stop capturing, clear the events and capture for few minutes once you load the malware.

### Promon Display

- Lets have a look at example where malware `mm32.exe` creates a file called `mw2mmgr.txt` at sequence number `212` using `CreateFile` . The word success in result column tells that operation was successful.

![procmon-display](public/04-PMA-Basic-Techniques/procmon-display.png)

### Filtering in Procmon

- One can also filter on individual system calls such as `RegSetValue`, `CreateFile`, `WriteFile`, or other suspicious or destructive calls.
- Filtering is only for visual purposes, all data is still being recorded

![filter](public/04-PMA-Basic-Techniques/filter.png)

- Some of the important filters
    - **Registry :** By examining registry operations, you can tell how a piece of
    malware installs itself in the registry.
    - **File system :** Exploring file system interaction can show all files that the
    malware creates or configuration files it uses.
    - **Process activity :** Investigating process activity can tell you whether the
    malware spawned additional processes.
    - **Network :** Identifying network connections can show you any ports on
    which the malware is listening.
- If your malware runs at boot time, use procmon’s boot logging options to install procmon as a startup driver to capture startup events

## Viewing Process With Process Explorer

![process-explorer](public/04-PMA-Basic-Techniques/process-explorer.png)

- Process Explorer shows process in tree format listing child and its parent process.

![properties](public/04-PMA-Basic-Techniques/properties.png)

### Using the verify option

- Verify button on image tab checks if the image on disk is microsoft signed binary or not
- This process happens on disk rather than in memory, so it is rendered useless if attacker uses process replacement which involves running a process on the system and overwriting its memory space with malicious executable providing same privileges as the process it is replacing

### Comparing Strings

![compare](public/04-PMA-Basic-Techniques/compare.png)

- One way to recognize process replacement is to compare strings between the memory and disk version of executable

### Using Dependency Walker

- Process Explorer allows launching of `depends.exe` (Dependency Walker) by right clicking a process
- It also allows you to search for Find Handle or DLL which is useful when you want to know if a particular malicious DLL is being used by any running process or not

### Analyzing Malicious Documents

- One can also open malicious PDF or word documents and see if any process are being created when opening them to see if they are malicious or not

## Comparing Registry Snapshots with RegShot

![regshot](public/04-PMA-Basic-Techniques/regshot.png)

- To use regshot, first take 1st shot, run the malware and then take 2nd shot and then we can compare what changes had been done

## Faking a Network

- Malware often communicates back to their C2 server, to get this data we need to setup our VM appropriately and not make it realize that its in virtual environment

### Using ApateDNS

- ApateDNS spoofs DNS response to user specified IP address by listening on UDP 53

### Monitoring with netcat

- Lets use ApateDNS to get malware to send its request our localhost, we can then use nc for listening to connections before cutting off the malware
- Malware often use port 80 and 443 for communication as they aren’t blocked or monitored for outbound connections

### Packet Sniffing with Wireshark

![wireshark](public/04-PMA-Basic-Techniques/wireshark.png)

- To use wireshark to view contents, right click any TCP packet and click “Follow TCP stream”

![follow](public/04-PMA-Basic-Techniques/follow.png)

- To capture packets, just click Capture→ Interfaces and select the interface

### Using InetSim

- InetSim provides fake services, allowing you to analyze the network behavior of malware
- It also handles all requests given to it appropriately without throwing a 404