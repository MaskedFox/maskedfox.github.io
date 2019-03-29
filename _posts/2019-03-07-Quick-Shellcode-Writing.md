---
title: "x86 a Super quick Guide to Shellcode Writing:"
layout: post
date: 2019-03-07 09:56:00
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
description: Basics of Shellcode writing
---


# **Writing Shellcode**

## **Steps:**

1.  #### **Write the code in C and create executable**
    
2.  #### **Disassemble the executable to look at the Assembly equivalent**
    
3.  #### **Remove unnecesary code**
    
4.  #### **Rewrite the final code in Assembly**
    
5.  #### **Use Objdump to figure out the final shellcode**
    

## **Steps with examples:**

-   #### Write the code in C and create executable
    
    ```c
    #include 
    
    main(void)
    {
        exit(0);
    }
    ```
    

-   #### Disassemble the executable to look at the Assembly equivalent
    
    ```nasm
    (gdb) disas main
    Dump of assembler code for function main:
       0x08049ab5 <+0>:    lea    ecx,[esp+0x4]
       0x08049ab9 <+4>:    and    esp,0xfffffff0
       0x08049abc <+7>:    push   DWORD PTR [ecx-0x4]
       0x08049abf <+10>:    push   ebp
       0x08049ac0 <+11>:    mov    ebp,esp
       0x08049ac2 <+13>:    push   ebx
       0x08049ac3 <+14>:    push   ecx
       0x08049ac4 <+15>:    call   0x8049ada <__x86.get_pc_thunk.ax>
       0x08049ac9 <+20>:    add    eax,0x91537
       0x08049ace <+25>:    sub    esp,0xc
       0x08049ad1 <+28>:    push   0x0
       0x08049ad3 <+30>:    mov    ebx,eax
       0x08049ad5 <+32>:    call   0x804fc90 
    End of assembler dump.
    (gdb) disas _exit
    Dump of assembler code for function _exit:
       0x0806c3c3 <+0>:     mov    ebx,DWORD PTR [esp+0x4]
       0x0806c3c7 <+4>:     mov    eax,0xfc
       0x0806c3cc <+9>:     call   DWORD PTR gs:0x10
       0x0806c3d3 <+16>: mov    eax,0x1
       0x0806c3d8 <+21>: int    0x80
       0x0806c3da <+23>: hlt
    ```
    
    **Here you can find the list with system calls that matches the above ^**
    
    ```root@mfox:/usr/include/i386-linux-gnu/asm# vim unistd_32.h```
    
    root@mfox:/usr/include/i386-linux-gnu/asm# vim unistd_32.h
    
    ```
    
    Ok, now that we know that 0xfc = 252 and 0x1 = 1 and if we look at the system calls
    
    We realize that 252 is exit_group and 1 is exit
    
    This means we are good using system call #1
    

-   #### Remove unnecesary code
    
    ```nasm
     mov ebx,DWORD PTR [esp+0x4]
     mov eax,0x1
     int 0x80
    ```
    

-   #### Rewrite the final code in Assembly
    
    vim exitShellCode.s
    
    ```nasm
    .text
    .globl _start
    
            _start:
                    movl $20, %ebx
                    movl $1, eax
                    int $0x80
    ```
    
    as -o exitShellcode.o exitShellCode.s
    
    ld -0 exitshellcode exitshellcode.o
    
    
-   #### **Use objdump to figure out the final shellcode**
    
    objdump -d exitShellcode
    

```nasm
root@mfox:~/Documents# objdump -d exitShellCode

exitShellCode:     file format elf32-i386


Disassembly of section .text:

08049000 <_start>:
 8049000:	bb 14 00 00 00       	mov    $0x14,%ebx
 8049005:	b8 01 00 00 00       	mov    $0x1,%eax
 804900a:	cd 80                	int    $0x80
```

```

#include <stdio.h>

char shellcode[] = "\xbb\x14\x00\x00\x00"
 "\xb8\x01\x00\x00\x00"
 "\xcd\x80"

int main(void)
{
 int *ret;
    /* defines a variable ret which is a pointer to an int. */
 
 ret = (int *)&ret +2;
    /* makes the ret variable point to an address on the stack which is located at a size 2 int away from it's own address. 
    This is presumably the address on the stack where the return address of main() has been stored. */

 (*ret) = (int)shellcode;
    /* assigns the address of the shellcode to the return address of the main 
     function. Thus when main() will exit, it will execute this shellcode instead of exiting normally. */
}
```

```
root@mfox:~/Documents# gcc -ggdb -mpreferred-stack-boundary=2 -m32 -fno-stack-protector -z execstack exitShellcode.c -o exitShellcode
```

Next well execute our Exit program and let's see what happens:

```
root@mfox:~/Documents# 
root@mfox:~/Documents# 
root@mfox:~/Documents# ./exitShellcode 
root@mfox:~/Documents# 
root@mfox:~/Documents# ./exitShellcode 
root@mfox:~/Documents# 
root@mfox:~/Documents# 

```
Voila, nothing happens because it exits =)

Now let's investigate what actually happens debbuging our program with GDB:

```bash
root@mfox:~/Documents# gcc -ggdb -mpreferred-stack-boundary=2 -m32 -fno-stack-protector -z execstack  shellcode.c -o shellcode
```

First Lets disassemble main:

```bash
root@mfox:~/Documents# gdb -q ./exitShellcode
Reading symbols from ./shellcode...done.
(gdb) disas main
Dump of assembler code for function main:
   0x00001189 <+0>:	push   ebp
   0x0000118a <+1>:	mov    ebp,esp
   0x0000118c <+3>:	sub    esp,0x4
   0x0000118f <+6>:	call   0x1185 <__x86.get_pc_thunk.dx>
   0x00001194 <+11>:	add    edx,0x2e6c
   0x0000119a <+17>:	lea    eax,[ebp-0x4]
   0x0000119d <+20>:	add    eax,0x8
   0x000011a0 <+23>:	mov    DWORD PTR [ebp-0x4],eax
   0x000011a3 <+26>:	mov    eax,DWORD PTR [ebp-0x4]
   0x000011a6 <+29>:	lea    edx,[edx+0x18]
   0x000011ac <+35>:	mov    DWORD PTR [eax],edx
   0x000011ae <+37>:	mov    eax,0x0
   0x000011b3 <+42>:	leave  
   0x000011b4 <+43>:	ret    
End of assembler dump.
(gdb) list
1	#include 
2	
3	char shellcode[] = "\xbb\x00\x00\x00\x00"
4			   "\xb8\x01\x00\x00\x00"
5			   "\xcd\x80";
6	
7	int main(void)
8	{
9		int *ret; 
10	    /* defines a variable ret which is a pointer to an int. */
(gdb) 
11	
12		ret = (int *)&ret +2;
13	    	/* makes the ret variable point to an address on the stack which is located at 		a size 2 int away from it's own address. 
14	    	This is presumably the address on the stack where the return address of main() 		has been stored. */
15	
16		(*ret) = (int)shellcode;
17	    	/* assigns the address of the shellcode to the return address of the main 
18		 function.
19	     	Thus when main() will exit, it will execute this shellcode instead of exiting 
20		normally. */
(gdb) 
21	
22	}
23	


```

Now let's break on line 12 and take a look at ESP ( Top of the Stack or Stack Pointer).

```bash
(gdb) b 12
Breakpoint 1 at 0x119a: file exitShellcode.c, line 12.
(gdb) r
Starting program: /root/Documents/exitShellcode
 Breakpoint 1, main () at exitShellcode.c:12
12 ret = (int *)&ret +2;
(gdb) x/8xw $esp
0xbffff2b4: 0xb7fbf000 0x00000000 0xb7dffb41 0x00000001
0xbffff2c4: 0xbffff354 0xbffff35c 0xbffff2e4 0x00000001

```

Cool so the above is the Top of the Stack, but what is the address `0xb7dffb41` ?

Let's disassemble it and find out!

```bash
(gdb) disas 0xb7dffb41
Dump of assembler code for function __libc_start_main:
 0xb7dffa50 <+0>: call 0xb7f1cc59 <__x86.get_pc_thunk.ax>
 0xb7dffa55 <+5>: add eax,0x1bf5ab
 0xb7dffa5a <+10>: push ebp
 0xb7dffa5b <+11>: xor edx,edx
 0xb7dffa5d <+13>: push edi
 0xb7dffa5e <+14>: push esi
 0xb7dffa5f <+15>: push ebx
 0xb7dffa60 <+16>: mov edi,eax
 0xb7dffa62 <+18>: sub esp,0x4c
 0xb7dffa65 <+21>: mov ecx,DWORD PTR [edi-0x7c]
 0xb7dffa6b <+27>: mov DWORD PTR [esp+0x8],eax
 0xb7dffa6f <+31>: mov eax,DWORD PTR [esp+0x74]
 0xb7dffa73 <+35>: test ecx,ecx
 0xb7dffa75 <+37>: je 0xb7dffa80 <__libc_start_main+48>
 0xb7dffa77 <+39>: mov edi,DWORD PTR [ecx]
 --Type  for more, q to quit, c to continue without paging--q
Quit
```

Interesting enough we find out the above address is pointing to`libc_start_main` which is responsible for starting the whole program and calling the main function =0 !

Cool!

So lets print our `ret pointer` and the check the Top of the stack address

```bash
(gdb) print /x ret
$1 = 0xb7fbf000
(gdb) x/8xw $esp
0xbffff2b4: 0xb7fbf000 0x00000000 0xb7dffb41 0x00000001
0xbffff2c4: 0xbffff354 0xbffff35c 0xbffff2e4 0x00000001

```

Right now basec on the Top of the Stack above we should know the following:

0xb7fbf000 = ret pointer

0x00000000 = EBP

0xb7dffb41 = Return

Now that we review the above lets "step" (s) in GDB to the next instruction:

```bash
(gdb) s
16		(*ret) = (int)shellcode;
(gdb) x/8xw $esp
0xbffff2b4:	0xbffff2bc	0x00000000	0xb7dffb41	0x00000001
0xbffff2c4:	0xbffff354	0xbffff35c	0xbffff2e4	0x00000001
(gdb) s
22	}
(gdb) x/8xw $esp
0xbffff2b4:	0xbffff2bc	0x00000000	0x00404018	0x00000001
0xbffff2c4:	0xbffff354	0xbffff35c	0xbffff2e4	0x00000001
(gdb) print &shellcode
$5 = (char (*)[13]) 0x404018 
```
Voila! again our Shellcode where the return address is, which means that it returns our shellcode =)

In case you want to double check this is our Shellcode, disassemble the address. Let's do it!

```
(gdb) disas 0x00404018
Dump of assembler code for function shellcode:
   0x00404018 <+0>:	mov    ebx,0x0
   0x0040401d <+5>:	mov    eax,0x1
   0x00404022 <+10>:	int    0x80
   0x00404024 <+12>:	add    BYTE PTR [eax],al
End of assembler dump.

```

I ll start a new blog post for the second part on How to write our own Shellcode, even though we can get it from a place like
shell-storm.com, its always good to know how to write your own in order to customize ;)
