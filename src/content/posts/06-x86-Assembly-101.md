---
title: "x86 Assembly 101"
published: 2025-09-23
description: "Beginner's Introduction to x86 Assembly"
image: '/06-x86-Assembly-101/banner.png'
tags: [Reversing, Assembly, x86, Guides, Fundamentals]
category: 'Guides'
draft: false
---

# x86 Assembly

## Endianness

Endianness is how you store the value for e.g. - `0x12345678` into RAM.

Little Endian - LSB(Least Significant Byte) or Lowest end would be stored at lowest address, i.e. the value would look like this — `0x78 0x56 0x34 0x12`

Big Endian - MSB(Most Significant Byte) or  Big end would be stored at the lowest address i.e, the value would look like this — `0x12 0x34 0x56 0x78`

Endianness only applies to memory storage of values and not to registers (it will be always in big endian) and only to bytes not to bits

## Sizes

Data in 32 bit assembly is bits, bytes, words, and dwords as follows—

- Bit can be 0 or 1
- Byte are 8 bits put together ranging between  0 and 255; signed (two’s-complement) `-128..127`
- Word is two bytes put together with max value of 65535; signed `-32768..32767`
- Dword is two words(D stands for double), max value being  4294967295.;signed `-2,147,483,648..2,147,483,64`

**Notes / tips**

- “Word” on x86 is **always 16 bits** (even in 32/64-bit modes). The 32-bit term is “doubleword/dword”; the 64-bit term is “quadword/qword”.
- Alignment matters: many ABIs prefer 4-byte stack alignment on x86 (and some libraries/SSE code expect 16-byte alignment at call sites).
- Sign vs zero extension appear often: `MOVSX`/`MOVZX`, `CBW/CWDE/CDQE`, `CWD/CDQ/CQO` (see below).

## Registers

A register is small storage space  which are much faster to access usual ones.

### General Purpose Register (GPR)

There are 8 of them as follows

- EAX → Extended Accumulator Register
- EBX → Extended Base Register
- ECX → Extended Counter Register
- EDX → Extended Data Register
- ESI → Extended Source Index
- EDI → Extended Destination Index
- EBP → Extended Base Pointer
- ESP → Extended Stack Pointer

All the GPR are 32 bit in size but a subset of them can be referred as follows—

| 32 bits | 16 bits | 8 bit |
| --- | --- | --- |
| EAX | AX | AH/AL |
| EBX | BX | BH/BL |
| ECX | CX | CH/CL |
| EDX | DX | DH/DL |
| ESI | SI |  |
| EDI | DI |  |
| EBP | BP |  |
| ESP | SP |  |

AX to SP are the 16 bit register used to reference the 16 least significant bits in their equivalent 32 bit registers.

The 8 bit register reference the higher and lower eight bits of the 16 bit registers

**EIP** – Extended Instruction Pointer is a **register** which points to next instruction to be executed.

**Notes / tips**

- Partial registers alias: writing `AL` only changes the low 8 bits of `AX/EAX`; writing `AH` touches bits 8–15; writing `AX` changes the low 16 of `EAX`.
- On older µarchs, mixing partial and full register writes could cause stalls; zero-idioms like `XOR EAX,EAX` are preferred to clear.
- There are other important architectural registers: **EFLAGS** (status), control registers **CR0–CR4**, debug registers **DR0–DR7**.

### Segment Registers

Segment Registers are used to make segmental distinctions in the binary, the hex value `0x90` can either be instruction or a data value. CPU knows which one it is because of segment registers.

**Details**

- The visible segment registers are **CS, DS, SS, ES, FS, GS**.
- In common 32-bit **flat** protected-mode (modern OSes), segment bases are set so that linear = virtual addresses, so you rarely manipulate segments directly. `FS`/`GS` are commonly used for TLS/TEB/PEB.
- Instruction fetch uses **CS**, data loads typically use **DS/SS** (stack uses **SS**). You can override with segment override prefixes (`FS:`, etc).

### Status Flag Registers

Flags are tiny bit values either set (1) or not set (0). Flags are stored in special flag register.

Some commonly used ones are

- Z → zero flag, set when the result of last operation is zero
- S → signed flag, set to determine if the value should be intercepted as signed or unsigned
- O → overflow flag, set when the result of last operation switches the most significant bit from either F to 0 or 0 to F
- C → carry flag, set when the result of the last operation changes the most significant bit

## Segments and Offsets

There are must have four segments in any program

- `.text` → stores program code
- `.data` → stores global data
- `.stack` → stores local variables, function arguments and much more
- `.heap` → extendable memory segment which programs use as per their need

### Stack

The stack is the part of memory where a program stores a local variables and function arguments for later use. 
It is LIFO (Last In First Out) data structure i.e. when something is added to stack, it is added at the top of stack and when something is removed from the stack, it is remove from the stack.

Stack grows backwards i.e. from highest memory address to lowest memory address.

ESP is stack pointer register that always points to the top of the stack and when something new is added at top of stack, ESP is *decremented*  because stack grows backwards

**Notes / tips**

- `PUSH` stores at `ESP-4` then updates `ESP`; `POP` reads at `ESP` then increments `ESP`.
- Many ABIs require stack alignment on call entry (Win32: 4-byte; some libraries/SSE code assume 16-byte).
- Use stack for temporaries/spills; prefer registers for hot paths.

### Stack Frames

The EBP is the base pointer, every function has its own stack frame. The base is the beginning of the stack frame. When function is called it creates its own stack frame which is marked out by EBP

**Typical prologue/epilogue**

```asm
push ebp
mov  ebp, esp
sub  esp, local_size
; ...
leave        ; mov esp, ebp / pop ebp
ret  argsz   ; optional immediate cleans args (stdcall)

```

**Notes / tips**

- Frame-pointer omission (FPO): compilers may use `EBP` as a GPR and address locals off `ESP`.
- `ENTER`/`LEAVE` exist but compilers favor `push/mov/sub` + `leave`.

### Heap

Heap is memory space where process can allocate memory when it needs it.

Each process has one heap and it is shared among the different threads. Heap is a linked-list data structure i.e. each item only knows the position of the immediate items before and after it.

**Notes / tips**

- Real allocators are more complex (bins, arenas, freelists). Windows uses the NT heap/Low-fragmentation heap; glibc uses ptmalloc; others: jemalloc, tcmalloc.
- Heap is per **process**; threads share it. Use care with concurrency and free-after-use.

## Instructions

Intel instructions vary in size from one to fourteen bytes. The opcodes(short for operation code) is mandatory for them all and can be combined to created advanced instructions.

Instructions may have upto 3 operators. Instructions containing `[]` means at certain memory offset.
Bytes are saved in reverse order, known as little endian representation i.e. most significant it of every byte is the most left bit

### NOP

Stands for no-operation, does literally nothing, takes no registers, no values.
Used for padding or aligning bytes or to delay time

Is an alias mnemonic for `XCHG EAX, EAX` which just exchanges register with itself doing nothing.

### Arithmetic Operations

**ADD  :**  

add dest, src

Destination and source can be either register, a memory reference (anything surrounded by `[]` is address reference). The source can also be an immediate number. Note that both destination and source cannot be a memory a reference at the same time, but both however can be registers.

**SUB :**  

sub dest, src

Works similar to add instruction

**DIV/IDIV :** 

div divisor

The dividend is always eax and that is also were the result of the operation is stored. The rest value is stored in edx. 

IDIV is same as DIV but signed division.

**MUL/IMUL :** 

mul value

mul dest, value, value

mul, dest, value

mul/imul (unsigned/signed) multiply either eax with a value or they multiply two values and put them in destination register or they multiply a register with a value.

**LEA :**
lea reg, [mem computes effective address; use as free add/shift (`lea eax, [ecx*4+edx+8]`).

### Bitwise Operations

**AND :** and dest, src

**OR :** or dest, src

**XOR :** xor dest, src

**NOT :** not eax

In bitwise two pieces of data are being compared bit by bit and depending on the operation, the outcome is either a 0 or a 1.

**Shifts / rotates**

- `shl/sal`, `shr`, `sar` (arith right keeps sign), `rol`, `ror`, `rcl`, `rcr`
- Count in `CL` or immediate (1). Shifts set `CF` to the last shifted-out bit.

### Branching

**J* :** j* address (*where * jmp type instruction)*

In assembly branching is made through the use of jumps and flags. A jump is just an instruction that under certain circumstances will point the EIP to another portion of code (much like `goto` in C).
Flags are tiny one bit values which can be 1 or 0. Most instructions set one or more flags.

**ADD and SUB** can set all the Z, S, O, C flags .

**AND** always clears O and C flags but sets  Z and S according to results.

Depending on flags set, a jump will either happen or not.

A lot of times you will see an instruction called **CMP**  being used before jump. CMP is the ideal pre-branch instruction as it can set all status flags and is really fast.

**CMP :** cmp dest, src

But its not a compulsion for CMP to be used, XOR also occurs frequently

### Data Moving

**MOV :** mov dest, src

**MOVSB :** movsb dest, src

**MOVZX :** movzx dest, src

MOV copies data from source to destination. Both source and destination can be register or register to memory but not memory to memory

MOVSX copies data from source to destination while preserving the sign.

MOVZX copies data from source to destination while filling upper bits in destination register with 0s

### Loops

loop example

```asm
mov ecx, 5 ; ecx, the extended counter register

_proc:
dec ecx    ; decrements ecx
loop _proc ; loops back to _proc
```

rep example (similar to loop but specifically designed to handle strings

```asm
mov esi, str1
mov edi, str2
mov ecx, 10h
rep cmps
```

String to be compared here are loaded in ESI and EDI and then comparison is performed for 16 bytes (10h), If at some point source and destination is not equal, a flag will be set and operation will be aborted.

**Notes / tips**

- `LOOP` is generally slower than `dec ecx / jnz` on modern CPUs; prefer explicit branches.
- For `rep movs*`/`stos*`, some CPUs have specialized microcode for large copies/sets.

### Stack Management

**POP :** pop dest

**PUSH :** push var/reg

The POP instruction pops a value or memory address (which is also a value) from the stack and stores it in destination and also increments ESP to point to new top of the stack.

PUSH pushes new value on stack and decrements ESP to the point to new top

**Also**

- `PUSHAD/POPAD` push/pop all 32-bit GPRs (deprecated in 64-bit mode, but available in 32-bit).
- `PUSHF/POPF` save/restore flags.

### Functions

**CALL :** call _func

**RET :** RET / RET num

CALL is like jump with several differences.  A jump instruction loads an address into EIP and continues execution from there. A CALL though stores current EIP On stack, with expectation to reload it once the callee(the called function) is done

When CALL functions occurs following steps occur →

1. EIP is stored on the stack ; this is done by CALL instruction
2. EBP is stored on the stack 
3. EBP is made to point to ESP ; an abstraction of calling convention
4. ESP is decremented to, among several things, contain the local variables of _func
5. EIP is loaded with the address of _func

When  callee has finished executing, the caller’s EBP is popped back into the EBP. 

Then the RET instruction removes the stack-frame of the callee by incrementing the ESP and pop the old saved EOP into EOP so that execution can continue where it left of.

Returns values are stored in EAX.

### Interrupts, Debugger Traps

**INT :** int num (num represents the interrupt handler)

Interrupts are used to tell the CPU to halt the execution of a thread. They can be hardware based, software based or exception based. 
When the INT instruction is hit, the execution is moved to an exception handler, which is defined by num.

When a software breakpoints is a set in debugger, the instruction where breakpoint is supposed to be hit is exchanged by INT3 (0xCC). Then the control is handed to debugger and the trap flag is set and CPU will execute one instruction at a time.

## Calling Conventions

### stdcall

In stdcall, function arguments are passed from right to left and the callee is in the charge of cleaning up the stack. Return values are stored in EAX. The stdcall is combination of pascal and cdecl

### cdecl

The cdecl (short for c declaration) originates from. Main difference between stdcall and cdecl is that the caller is responsible for cleaning up the stack 

### pascal

It originates from pascal programming language. The main difference between stdcall and pascall is that parameters are pushed from left to right.

### fastcall

It is non-standardized calling convention. It is usually recognized through the way it sends function arguments. While all above conventions use the stack to store the function arguments, fastcall loads them into registers.

### Quick tips

- Endianness: little-endian affects **byte** layout in memory, not bit order inside a byte.
- Use `CLD` before string ops if you rely on forward direction (some code may have set `DF`).
- Compare type matters: use `JA/JB` for **unsigned** and `JG/JL` for **signed** after `CMP`.
- Before `IDIV`, set up `EDX:EAX` correctly: `CDQ` sign-extends `EAX` into `EDX`.
- Zeroing a register: `XOR reg, reg` or `SUB reg, reg` both set it to 0; `XOR` is preferred (zero-idiom).

> References: [Sensepost Blog](https://sensepost.com/blogstatic/2014/01/SensePost_crash_course_in_x86_assembly-.pdf) and [OST2 x86-64 Assembly](https://apps.p.ost2.fyi/learning/course/course-v1:OpenSecurityTraining2+Arch1001_x86-64_Asm+2021_v1/home)

