---
title: Making Medusa : My First CrackMe - Part 0x01
published: 2025-07-19
description: 'Making my first crackme challenge in C'
image: '/05-Making-Medusa-My-First-CrackMe-Part-0x01/banner.png'
tags: [Reversing, CrackMe]
category: 'Reversing'
draft: false
---

## Table Of Contents

- [main.c](#mainc)
- [pseudo-C](#pseudo-c)
- [disassembly](#disassembly)
- [layering](#layering)
- [problemo](#problemo)
- [what-we-learnt](#what-we-learnt)

I had just started learning C and after completing few basics, I was looking for my first project to make. After thinking around, I landed on making a CrackMe challenge in C.
Goal would be to start small, compile the binary, view the disassembly, view the pseudo-C code in IDA-Free and co-relate everything and move to adding more complexities.

GGs, that sounds fun!
Lets code, shall we?

### main.c
First I wrote a simple `main.c` program---
```c
#include  <stdint.h>
#include  <sys/mman.h>
#include  <string.h>

int  main()
{
	uint8_t  code  []  =  {0xB8,  0x42,  0x00,  0x00, 0x00,  0xC3}; // mov eax, 0x42; ret

	void  *mem  =  mmap(NULL,  1024,  PROT_READ  |  PROT_WRITE  |  PROT_EXEC,  MAP_ANON  |  MAP_PRIVATE,  -1,  0); // create a protected executable anonymous private memory region

	memcpy(mem,  code,  sizeof(code)); // copy the code to that region

	int  (*func)()  =  mem; // cast a function pointer and point it to mem

	int  result  =  func(); // execute the func() function and store the result in result variable

	return  result; // return the result which should be 66
}
```
Lets compile the program (`gcc -m32 -fno-stack-protector -z execstack -no-pie -fno-pic main.c -o main`) and view its pseudo-C code

### pseudo-C
```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int (*v3)(void); // function pointer that takes no argument

  v3 = (int (*)(void))mmap(0, 1024u, 7, 0x22, -1, 0); // calls mmap to allocate 1024 bytes of memory with read, write, and execute permissions
  *(_DWORD *)v3 = 0x42B8; // store 0x42B8 in memory
  *((_WORD *)v3 + 2) = 0xC300; // store next bytes (0xC300) with offset of 4 (2 WORDs)
  return v3(); // calls the function executing the code
}
```

### disassembly
and now its disassembly
```asm
; int __cdecl main(int argc, const char **argv, const char **envp)
public main
main proc near

; variables
var_1A= dword ptr -1Ah
var_16= word ptr -16h
var_14= dword ptr -14h
var_10= dword ptr -10h
var_C= dword ptr -0Ch
var_4= dword ptr -4
argc= dword ptr  8
argv= dword ptr  0Ch
envp= dword ptr  10h

; __unwind {
; stack frame setup
lea     ecx, [esp+4]
and     esp, 0FFFFFFF0h
push    dword ptr [ecx-4]
push    ebp
mov     ebp, esp
push    ecx
sub     esp, 24h

; call mmap to allocate executable memory and storing return pointer in var_C
mov     [ebp+var_1A], 42B8h
mov     [ebp+var_16], 0C300h
sub     esp, 8
push    0               ; offset
push    0FFFFFFFFh      ; fd
push    22h ; '"'       ; flags
push    7               ; prot
push    400h            ; len
push    0               ; addr
call    _mmap
add     esp, 20h
mov     [ebp+var_C], eax

; copy machine code to allocated memory
mov     eax, [ebp+var_C]
mov     edx, [ebp+var_1A]
mov     [eax], edx
movzx   edx, [ebp+var_16]
mov     [eax+4], dx

; prepare and call the function
mov     eax, [ebp+var_C]
mov     [ebp+var_10], eax
mov     eax, [ebp+var_10]
call    eax

; return the result and clean the stack
mov     [ebp+var_14], eax
mov     eax, [ebp+var_14]
mov     ecx, [ebp+var_4]
leave
lea     esp, [ecx-4]
retn
; } // starts at 8049166
main endp

_text ends
```

### layering
Now that we have seen code in all three forms, let's add some layers. I wrote another file `validate.c`---
```c
#include  <stdio.h>
#include  <string.h>
int  validate(const  char  *input);  
int  main()
{
	char  input[64];
	scanf("%63s",  input);
	if  (validate(input))
	{
	printf("Correct!\n");
	}
	else
	{
	printf("Wrong!\n");
	}
	return  0;
}  

int  validate(const  char  *input)
{
	const  char  *flag  =  "pwning-since-1337";
	int  i  =  0;
	for  ( ; ;  i++)
	{
		unsigned  char  a  =  (unsigned  char)input[i];
		unsigned  char  b  =  (unsigned  char)flag[i];
		if  (a  !=  b)
		{
			return  0;
		}
		if  (a  ==  0)
		{
			return  1;
		}
	}
}
```
I compiled it again and this time we extract just the raw instruction bytes of validate part of the code `objdump -M intel -d validate | awk '/<validate>:/,/^$/' | \
awk '/^[[:space:]]*[0-9a-f]+:/ {for(i=2;i<=10;i++) if($i ~ /^[0-9a-f][0-9a-f]$/) printf "0x%s, ", $i} END {print ""}'` (thanks chatGPT).

which now we will be XORing with a key (`0x1337`) using python---
```py
def  xor_encrypt(buf:  bytearray,  key:  int)  ->  None:
	key_bytes =  [key &  0xFF,  (key >>  8)  &  0xFF]
	for i in  range(len(buf)):
	buf[i]  ^=  key_bytes[i %  2]

validate_bytes =  [
0x55,  0x89,  0xe5,  0x83,  0xec,  0x10,  0xc7,  0x45,  0xf8,  0x1d,  0xa0,  0x04,
0x08,  0xc7,  0x45,  0xfc,  0x00,  0x00,  0x00,  0x00,  0x8b,  0x55,  0xfc,  0x8b,
0x45,  0x08,  0x01,  0xd0,  0x0f,  0xb6,  0x00,  0x88,  0x45,  0xf7,  0x8b,  0x55,
0xfc,  0x8b,  0x45,  0xf8,  0x01,  0xd0,  0x0f,  0xb6,  0x00,  0x88,  0x45,  0xf6,
0x0f,  0xb6,  0x45,  0xf7,  0x3a,  0x45,  0xf6,  0x74,  0x07,  0xb8,  0x00,  0x00,
0x00,  0x00,  0xeb,  0x13,  0x80,  0x7d,  0xf7,  0x00,  0x75,  0x07,  0xb8,  0x01,
0x00,  0x00,  0x00,  0xeb,  0x06,  0x83,  0x45,  0xfc,  0x01,  0xeb,  0xc1, 0xc9,
0xc3

]

data =  bytearray(validate_bytes)

xor_encrypt(data,  0x1337)

print("Encrypted:",  ', '.join(f'0x{b:02x}'  for b in data))
```

Now we update our C code to look something like this---
```c
#include  <stdint.h>
#include  <sys/mman.h>
#include  <string.h>
#include  <stdio.h>

void  xor_decrypt(uint8_t  *buf,  size_t  len,  uint16_t  key);

int  main()
{
	char  input[64];
	printf("WELCOME TRAVELLER, SPEAK THY SHAN'T BE STONED: ");
	fflush(stdout);
	if  (scanf("%63s",  input)  !=  1)
	{
		return  1;
	}
	
	uint8_t  code[]  =  {0x62,  0x9a,  0xd2,  0x90,  0xdb,  0x03,  0xf0,  0x56,  0xcf,  0x0e,  0x97,  0x17,  0x3f,  0xd4,  0x72,  0xef,  0x37,  0x13,  0x37,  0x13,  0xbc,  0x46,  0xcb,  0x98,  0x72,  0x1b,  0x36,  0xc3,  0x38,  0xa5,  0x37,  0x9b,  0x72,  0xe4,  0xbc,  0x46,  0xcb,  0x98,  0x72,  0xeb,  0x36,  0xc3,  0x38,  0xa5,  0x37,  0x9b,  0x72,  0xe5,  0x38,  0xa5,  0x72,  0xe4,  0x0d,  0x56,  0xc1,  0x67,  0x30,  0xab,  0x37,  0x13,  0x37,  0x13,  0xdc,  0x00,  0xb7,  0x6e,  0xc0,  0x13,  0x42,  0x14,  0x8f,  0x12,  0x37,  0x13,  0x37,  0xf8,  0x31,  0x90,  0x72,  0xef,  0x36,  0xf8,  0xf6,  0xda,  0xf4}; // encrypted xor
	
	void  *mem  =  mmap(NULL,  1024,  PROT_READ  |  PROT_WRITE  |  PROT_EXEC,  MAP_ANON  |  MAP_PRIVATE,  -1,  0); // create a protected executable anonymous private memory region
	
	memcpy(mem,  code,  sizeof(code)); // copy the code to that region
	
	xor_decrypt((uint8_t  *)mem,  sizeof(code),  0x1337); // decrypt mem in runtime (use sizeof code as we only need to decrypt that many bytes)
	int  (*validate_func)(const  char  *)  =  mem; // cast a function pointer and point it to mem
	
	int  ok  =  validate_func(input);
	
	puts(ok  ?  "YOU ARE SAVED TRAVELLER, YOU MAY PROCEED!"  :  "YOU GOT STONED BY THE MEDUSA!");
}

void  xor_decrypt(uint8_t  *buf,  size_t  len,  uint16_t  key)
{
	uint8_t  key_bytes[2];
	key_bytes[0]  =  key  &  0xFF; // lower byte
	key_bytes[1]  =  (key  >>  8)  &  0xFF; // upper byte
	for  (size_t  i  =  0;  i  <  len;  i++)
	{
		buf[i]  ^=  key_bytes[i  %  2]; // alternate between lower and upper byte
	}
}
```

### problemo
But there comes a problem, no matter what I entered, wrong flag or right flag, It would always give me `"YOU GOT STONED BY THE MEDUSA! D:"`
What went wrong, after pondering and tinkering I realised that the flag `pwning-since-1337` would be stored in `.rodata` and there would be no way to access it in `validate`'s function.

We need to write self contained function which has the `pwning-since-1337` itself.
So we just make an local array, easy-peasy-lemon-squeezy!

Here is our updated `validate-self-contained.c`---
```c
#include  <stdio.h>
#include  <string.h>

int  validate(const  char  *input);

int  main()
{
	char  input[64];
	scanf("%63s",  input);
	
	if  (validate(input))
	{
		printf("Correct!\n");
	}
	else
	{
		printf("Wrong!\n");
	}
	
	return  0;
}
  
int  validate(const  char  *input)
{
// Store the flag as a local array, not as a pointer to a string literal
const  unsigned  char  flag[]  =  {
'p',  'w',  'n',  'i',  'n',  'g',  '-',  's',  'i',  'n',  'c',  'e',  '-',  '1',  '3',  '3',  '7',  0};

int  i  =  0;
while  (1)
{
	unsigned  char  a  =  (unsigned  char)input[i];
	unsigned  char  b  =  flag[i];
	if  (a  !=  b)
	{
		return  0;
	}
		
	if  (a  ==  0)
	{
		return  1;
	}		
	i++;
}
}
```
Compile. Extract. XOR. Same yadda yadda process
and lets hit run!
![working-image](public/05-Making-Medusa-My-First-CrackMe/pwning.png)
yipeee, it is working!

We now have a simple working [CrackMe](https://github.com/iamavu/Medusa).


### what we learnt
- we can create a executable region in memory from which we can execute code
- how to alternatively use key's both bytes to XOR 
- the data or our flag was stored in `.rodata` hence we needed to make it local

In next post, we will be adding more layers to this, see you soon pwners : D

