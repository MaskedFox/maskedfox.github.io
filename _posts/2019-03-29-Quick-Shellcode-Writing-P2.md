---
title: "x86 a Super quick Guide to Shellcode Writing Part 2:"
layout: post
date: 2019-03-29 12:19:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
- gdb
- reversing
- debugging
- assembly
- linux
- exploit
- buffer overflow
- shellcode
star: true
category: blog
author: MaskedFox
description: Basics of Shellcode writing Part 2
---


# **Writing Shellcode Part 2**

##       **Disassembling Execve:**

**man execve**

```bash
EXECVE(2)                  Linux Programmer's Manual                 EXECVE(2)

NAME
       execve - execute program

SYNOPSIS
       #include 

       int execve(const char *filename, char *const argv[],
                  char *const envp[]);

DESCRIPTION
       execve()  executes the program pointed to by filename.  This causes the
       program that is currently being run by the calling process  to  be  re‐
       placed  with  a  new  program,  with newly initialized stack, heap, and
       (initialized and uninitialized) data segments.
```

```c
/***************
 *@Title: shell.c
 *
 *(Disassembling Execve)
 *
 * *************/
#include 
#include 

int main()
{
	char *args[2];

 	args[0] = "/bin/bash";
	args[1] = 0x0;
	execve(args[0], args, 0x0);
	
	exit(0);
}

```

Let's compile the above program and run it

1.  **compile it**
    
    ```c
    gcc -ggdb -mpreferred-stack-boundary=2 -m32 -fno-stack-protector -z execstack shell.c -o shell
    ```
    
2.  **run it**
    
    ```c
    
    root@mfox:~/Documents# ./shell
    root@mfox:/root/Documents# 
    ```
    

Voila! We got ourselves a shell

Lets look at what happen when we disas the main function of our program:

```bash
root@mfox:~/Documents# gdb -q ./shell
Reading symbols from ./shell...done.
(gdb) disas main
Dump of assembler code for function main:
   0x000011a9 <+0>:	push   ebp
   0x000011aa <+1>:	mov    ebp,esp
   0x000011ac <+3>:	push   ebx
   0x000011ad <+4>:	sub    esp,0x8
   0x000011b0 <+7>:	call   0x10b0 <__x86.get_pc_thunk.bx>
   0x000011b5 <+12>:	add    ebx,0x2e4b
   0x000011bb <+18>:	lea    eax,[ebx-0x1ff8]
   0x000011c1 <+24>:	mov    DWORD PTR [ebp-0xc],eax
   0x000011c4 <+27>:	mov    DWORD PTR [ebp-0x8],0x0
   0x000011cb <+34>:	mov    eax,DWORD PTR [ebp-0xc]
   0x000011ce <+37>:	push   0x0
   0x000011d0 <+39>:	lea    edx,[ebp-0xc]
   0x000011d3 <+42>:	push   edx
   0x000011d4 <+43>:	push   eax
   0x000011d5 <+44>:	call   0x1050 
   0x000011da <+49>:	add    esp,0xc
   0x000011dd <+52>:	push   0x0
   0x000011df <+54>:	call   0x1030 
End of assembler dump.

```

Now, lets see the above from the stack point of view: Remeber the Stack growth from High to Low memory as noted here: https://stackoverflow.com/questions/4560720/why-does-the-stack-address-grow-towards-decreasing-memory-addresses

```bash
******************    High Memory                    
*     (4 Bytes)  *
******************
*    EBP old     *
******************
*    0x0 = Null  * 
******************    EAX = p(/bin/bash)
*    p(/bin/bash)*
******************
*    0x0 = NULL  * 
******************    EDX = /bin/bash
*    EAX         *
******************
*    EDX         *
******************    Low Memory
```

Cool, Now that we understand better whats happening with our code, then we can build our Assembly:

```nasm
.data
    Bash:
        .asciz "/bin/bash"
        
    Nulll:
        .int 0;
        
    AddrToBash:
        .int 0
        
    Null2:
        .int 0
        
.text
    .globl _start
    
_start:
    # Execve routine
    
    movl $Bash, AddrToBash
    movl $11, %eax
    movl $Bash, %ebx
    movl $AddrToBash, %ecx
    movl $Null2, %edx
    int $0x80
    
    # Exit Routine
    
    Exit:
    movl $10, %ebx
    movl $1, %eax
    int $0x80 
```

Ok, lets review the above ^:

```
$11 is the syscall to execve
$Bash is the pointer to "/bin/bash"
$AddrToBash is the pointer to variable of Arguments
$Null is just a Null or 0x0
```

Cool, now lets assemble the program above:

```bash
root@mfox:~/Documents# as -ggstabs -o exec.o  exec.s
```

Now let's link it and Lets run it:

```bash
root@mfox:~/Documents# ld -o exec exec.o
root@mfox:~/Documents# ./exec 
root@mfox:/root/Documents# 

```

Cool it works!, lets see the opcodes for the above program:

```bash
root@mfox:~/Documents# objdump -d exec

exec:     file format elf32-i386


Disassembly of section .text:

08049000 <_start>:
 8049000:	c7 05 0e a0 04 08 00 	movl   $0x804a000,0x804a00e
 8049007:	a0 04 08 
 804900a:	b8 0b 00 00 00       	mov    $0xb,%eax
 804900f:	bb 00 a0 04 08       	mov    $0x804a000,%ebx
 8049014:	b9 0e a0 04 08       	mov    $0x804a00e,%ecx
 8049019:	ba 12 a0 04 08       	mov    $0x804a012,%edx
 804901e:	cd 80                	int    $0x80

08049020 :
 8049020:	bb 0a 00 00 00       	mov    $0xa,%ebx
 8049025:	b8 01 00 00 00       	mov    $0x1,%eax
 804902a:	cd 80                	int    $0x80

```

**#Important Resources:**

These serious of blogs come from the amazing tutotials from Security Tube: https://youtu.be/e2jQgOVkJPM
