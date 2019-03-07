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
```
root@mfox:~/Documents# objdump -d exitShellCode

exitShellCode:     file format elf32-i386


Disassembly of section .text:

08049000 <_start>:
 8049000:	bb 14 00 00 00       	mov    $0x14,%ebx
 8049005:	b8 01 00 00 00       	mov    $0x1,%eax
 804900a:	cd 80                	int    $0x80

```
