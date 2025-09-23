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

Endianness describes how a multi-byte integer like `0x12345678` is laid out **in memory** (lowest address first).

* **Little-endian** (x86): least-significant byte at the lowest address
  memory bytes: `78 56 34 12`
* **Big-endian**: most-significant byte at the lowest address
  memory bytes: `12 34 56 78`

**Important clarifications**

* Endianness is about **byte order in memory**. Registers hold abstract values; when you move part of a register to memory, the stored bytes follow the machine’s endianness.
* Endianness concerns **bytes**, not the order of **bits** within a byte.

---

## Sizes

In 32-bit x86, common integer sizes are:

* **bit**: `0` or `1`
* **byte** (8 bits): unsigned `0..255`, signed (two’s-complement) `−128..127`
* **word** (16 bits): unsigned `0..65535`, signed `−32768..32767`
* **dword** / **doubleword** (32 bits): unsigned `0..4294967295`, signed `−2147483648..2147483647`

**Notes / tips**

* On x86, a **word is always 16 bits** (even in 32/64-bit modes). 32-bit = **doubleword**; 64-bit = **quadword**.
* Alignment matters: many 32-bit ABIs prefer **4-byte** stack alignment; some libraries/SSE code assume **16-byte** alignment at call sites.
* Sign/zero-extension show up frequently:

  * `MOVSX`/`MOVZX` (load smaller → larger with sign/zero fill)
  * `CBW`/`CWDE` (sign-extend AL→AX, AX→EAX)
  * `CWD`/`CDQ` (sign-extend AX→DX\:AX, EAX→EDX\:EAX)

---

## Registers

A **register** is a small, fast storage cell inside the CPU.

### General-Purpose Registers (GPRs)

The classic 32-bit set:

* `EAX` (accumulator) `EBX` (base) `ECX` (count) `EDX` (data)
* `ESI` (source index) `EDI` (dest index)
* `EBP` (frame/base pointer) `ESP` (stack pointer)

Sub-registers:

| 32 bits | 16 bits | 8 bits  |
| ------- | ------- | ------- |
| EAX     | AX      | AH / AL |
| EBX     | BX      | BH / BL |
| ECX     | CX      | CH / CL |
| EDX     | DX      | DH / DL |
| ESI     | SI      | —       |
| EDI     | DI      | —       |
| EBP     | BP      | —       |
| ESP     | SP      | —       |

* `AX..SP` are the **low 16** bits of the corresponding `E*` register.
* `AH/AL` access the high/low 8 bits of `AX` (similar for `BH/BL`, etc).

Other architectural registers:

* **`EIP`** – instruction pointer (address of next instruction)
* **`EFLAGS`** – status/control flags (see below)
* Control registers **`CR0..CR4`**, debug registers **`DR0..DR7`**, etc.

**Notes / tips**

* Partial register aliasing: writing `AL` changes only bits 7..0 of `EAX`; writing `AX` changes bits 15..0.
* On older µarchs, mixing partial and full writes could cause stalls; use zero-idioms like `xor eax,eax` to clear efficiently.

### Segment Registers

Visible segment registers: **`CS`, `DS`, `SS`, `ES`, `FS`, `GS`**.

* In typical 32-bit **flat** protected mode (modern OSes), segment bases are set so that linear = virtual addresses; you rarely manipulate them.
* **`CS`** is used for instruction fetch; **`DS`/`SS`** for most data/stack. You can override with prefixes (`FS:`, `GS:`, etc).
* `FS`/`GS` are commonly used for thread-local data (e.g., TEB/PEB on Windows).

> The same byte value (e.g., `0x90`) can be “code” or “data” depending on **how** it’s accessed (instruction fetch via `CS:EIP` vs load/store via data segments), not because the byte itself carries a tag.

### Status Flag Registers

Key `EFLAGS` bits (set/cleared by many instructions):

* **ZF** (Zero Flag): result == 0
* **SF** (Sign Flag): most-significant bit of result (interpreting sign)
* **OF** (Overflow Flag): signed overflow on add/sub/arith
* **CF** (Carry Flag): carry/borrow out for unsigned arith, last bit shifted/rotated out
* Others: **PF** (parity), **AF** (aux carry), **DF** (direction), **TF** (trap), etc.

---

## Segments and Offsets

Typical **sections/segments** in a program image:

* **`.text`** – code
* **`.data`** – initialized globals
* **`.bss`** – zero-initialized globals (often implicit)
* **stack** – call frames, locals, return addresses
* **heap** – dynamically allocated memory

(Exact naming/layout varies by OS/toolchain; conceptually these regions exist in every process.)

### Stack

A LIFO region used for call frames, locals, and temporaries.

* Grows **downward** (toward lower addresses) on x86.
* **`ESP`** points to the top (last pushed item is at `[ESP]`).

**Mechanics**

* `PUSH X`: store X at `ESP-4`, then `ESP := ESP-4`
* `POP  R`: load from `[ESP]` into `R`, then `ESP := ESP+4`

**Notes / tips**

* Many ABIs(Application Binary Interface) require stack alignment at call boundaries (Win32: 4-byte; some code assumes 16-byte).
* Prefer registers in hot paths; use the stack for spills/temporaries.

### Stack Frames

A function’s frame is typically anchored by **`EBP`**.

**Prologue / Epilogue (typical)**

```asm
push ebp
mov  ebp, esp
sub  esp, local_size
; ... body ...
leave          ; mov esp, ebp / pop ebp
ret  argsz     ; optional immediate cleans args (stdcall)
```

**Notes / tips**

* Compilers may omit the frame pointer (FPO) and address locals relative to `ESP`.
* `ENTER`/`LEAVE` exist; `push/mov/sub` + `leave` is preferred.

### Heap

Process-wide dynamic memory managed by an **allocator**.

* Real allocators use bins/arenas/freelists; not just a single linked list.
* The **heap is per process**; all threads share it. Free only what you allocated; avoid use-after-free/data races.

---

## Instructions

x86 instructions are **variable-length** (1–15 bytes). An instruction consists of opcode + optional prefixes, ModR/M, SIB, displacement, and immediate.

* At most **two** explicit memory operands are **not** allowed for ALU ops: x86 generally forbids **mem-to-mem** arithmetic; use a register as one side.
* Little-endian governs how multi-byte immediates/displacements are encoded and how memory operands are stored.

### NOP

**No-operation**; occupies a cycle/byte slot, affects nothing.

* Preferred encoding: `NOP` (`0x90`).
* Historically can be expressed as `XCHG EAX,EAX`, but modern assemblers encode proper NOPs (and multi-byte NOPs for alignment).

### Arithmetic Operations

> Some notations:
> r/mX = register or memory operand (X bits)
> immX = immediate value (X bits)
> rX = register operand (X bits)

**ADD**

```asm
add dest, src   ; dest := dest + src
```

* Sets `CF`, `OF`, `SF`, `ZF`, `AF`, `PF` as appropriate.
* `dest` may be reg/mem; `src` may be reg/mem/imm (but not mem+mem).

**SUB**

```asm
sub dest, src   ; dest := dest - src
```

* Similar flags behavior to `ADD`.

**DIV / IDIV** (unsigned / signed divide)

```asm
; 32-bit:
; dividend in EDX:EAX (hi:lo)
div  r/m32      ; EAX := quotient, EDX := remainder
idiv r/m32      ; signed; set up EDX:EAX with CDQ
```

* Before `IDIV`, use `CDQ` to sign-extend `EAX` into `EDX`.
* Division by zero or quotient overflow raises an exception.

**MUL / IMUL** (unsigned / signed multiply)

Forms:

* One-operand:

  ```asm
  mul  r/m32         ; EDX:EAX := EAX * r/m32
  imul r/m32         ; signed
  ```
* Two-operand:

  ```asm
  imul r32, r/m32    ; r32 := r32 * r/m32
  ```
* Three-operand:

  ```asm
  imul r32, r/m32, imm8/imm32  ; r32 := (r/m32) * imm
  ```

**LEA** (Load Effective Address)

```asm
lea reg, [base + index*scale + disp]
```

* Computes the address expression; **does not** touch memory; flags unaffected. Often used as a free add/shift: e.g., `lea eax, [ecx*4 + edx + 8]`.

### Bitwise Operations

```asm
and dest, src
or  dest, src
xor dest, src
not dest        ; unary
```

* `AND/OR/XOR` set `ZF/SF` from the result and **clear `CF` and `OF`**. `NOT` affects no flags.

**Shifts / Rotates**

* `SHL`/`SAL` (logical/arithmetic left), `SHR` (logical right), `SAR` (arithmetic right), `ROL`, `ROR`, `RCL`, `RCR`
* Count is `1`, imm8, or in `CL`. `CF` gets the last shifted-out bit; `OF` defined for count = 1 (`SHL/SAL/SHR/SAR`).

### Branching

Unconditional/conditional jumps change `EIP` based on flags.

* Set flags with `CMP dest, src` (`dest - src`) or `TEST` for bitwise checks.
* Conditional jumps (selected set):

  * Unsigned: `JA/JNBE` (>, no carry/zero), `JB/JNAE` (<, carry), `JAE/JNB` (>=), `JBE/JNA` (<=)
  * Signed: `JG/JNLE` ( > ), `JL/JNGE` ( < ), `JGE/JNL`, `JLE/JNG`
  * Equality: `JE/JZ`, `JNE/JNZ`
  * Others: `JC/JNC`, `JO/JNO`, `JS/JNS`, `JP/JPE`, `JNP/JPO`

**Notes / tips**

* `XOR reg,reg` sets `ZF=1` iff `reg` was zero, but it **clobbers** the operand; for comparisons, prefer `CMP`/`TEST`.
* Falling back to `dec ecx / jnz` is typically faster than `LOOP` on modern CPUs.

### Data Moving

```asm
mov   dest, src     ; register <-> register/memory/immediate
movsx dest, src     ; sign-extend
movzx dest, src     ; zero-extend
```

String forms (implicit operands):

* `MOVSB/MOVSW/MOVSD` move byte/word/dword from `[ESI]` → `[EDI]`, then `ESI += ±size`, `EDI += ±size` based on `DF`.
* `CMPSB/CMPSW/CMPSD` compare `[ESI]` vs `[EDI]`; `REPE/REPZ` and `REPNE/REPNZ` repeat while equal/unequal.


### Loops

Counter-based loop:

```asm
mov  ecx, 5
.proc:
  ; ... body ...
  dec  ecx
  jnz  .proc
```

String/rep example:

```asm
cld                 ; ensure forward
mov esi, str1
mov edi, str2
mov ecx, 16
repe cmpsb          ; stop on mismatch or ECX==0
```

**Notes / tips**

* `LOOP` exists (`ecx-- ; jnz target`) but is usually slower than explicit `dec/jnz`.
* For big copies/sets, `REP MOVS*`/`STOS*` may use optimized microcode paths.

### Stack Management

```asm
push r/m32
pop  r/m32
```

* `PUSHAD/POPAD` push/pop all GPRs (32-bit only; deprecated in 64-bit).
* `PUSHF/POPF` save/restore `EFLAGS`.

Constraints: `push`/`pop` work with regs/mem (no immediates for `pop`; `push imm` is fine).

### Functions

**CALL / RET**

* `CALL target`: pushes the **return address** (next `EIP`) to the stack, then jumps.
* Typical callee prologue/epilogue uses `EBP` to anchor the frame (see Stack Frames).
* `RET` pops the return address into `EIP`. `RET imm16` additionally **adjusts the stack** by `imm16` bytes (callee cleanup).

Return values: scalars typically in **`EAX`** (wider returns may use `EDX:EAX`).

### Interrupts, Debugger Traps

```asm
int  imm8      ; software interrupt
int3            ; 0xCC, breakpoint
```

* `INT n` vectors through the IDT to a handler.
* Debuggers set breakpoints by patching an instruction byte to `INT3` (`0xCC`).
* Single-step uses the **Trap Flag (`TF`)**; after each instruction, a debug exception (#DB) is raised.

---

## Calling Conventions

### stdcall

* Args pushed **right→left** on the stack.
* **Callee** cleans the stack (`ret argbytes`).
* Return in `EAX`.

### cdecl

* Args pushed **right→left**.
* **Caller** cleans the stack (`ret`).
* Variadic functions require caller cleanup; return in `EAX`.

### pascal

* Args pushed **left→right**.
* Typically callee cleanup; return in `EAX`. (Historic; uncommon today.)

### fastcall

* Non-standardized family.
* On 32-bit Windows `__fastcall`: pass first two args in **`ECX`** and **`EDX`**, rest on stack; callee usually cleans.
* Other toolchains define different fastcall ABIs; always check your compiler/OS docs.

### Quick tips

* Endianness: affects **byte layout in memory**, not the order of bits.
* Use `CLD` before string ops if you require forward progression; some code may have set `DF`.
* After `CMP`: choose jumps by **unsigned** (`JA/JB/...`) vs **signed** (`JG/JL/...`) intent.
* Before `IDIV`, prepare `EDX:EAX` correctly: `CDQ` sign-extends `EAX` into `EDX`.
* Zeroing registers: `xor reg,reg` (zero-idiom) is efficient and breaks dependencies; `sub reg,reg` also works but is less canonical.


> References: [Sensepost Blog](https://sensepost.com/blogstatic/2014/01/SensePost_crash_course_in_x86_assembly-.pdf) and [OST2 x86-64 Assembly](https://apps.p.ost2.fyi/learning/course/course-v1:OpenSecurityTraining2+Arch1001_x86-64_Asm+2021_v1/home)

