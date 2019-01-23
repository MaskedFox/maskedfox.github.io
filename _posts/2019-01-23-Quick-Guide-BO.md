---
title: "x86 a Super quick Guide to BO:"
layout: post
date: 2019-01-23 18:02
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
star: true
category: blog
author: MaskedFox
description: Basics of BUffer Overflow for x86, more like a cheatsheet =p
---


#      **x86 a Super quick Guide to BO:**

##     Basics of x86 Stack:

####     The Stack grows from High to Low and if your buffer was 8 bytes. The stack will look like this:

`(High Memory)`

`|RET (if nothing then 4 bytes) |`

`|EBP (if nothing then 4 bytes) |`

`|ESP (8 bytes) |`

`(Low Memory)`

##       Basics of Buffer Overflows(BO):

####   This is suppose to be a reminder of Basic BO, think of it  as a Cheat Sheet:

**Vulnerable code:**

  pwndbg> disas main
  Dump of assembler code for function main:
  0x004011e9 <+0>: push ebp
  0x004011ea <+1>: mov ebp,esp
  0x004011ec <+3>: call 0x401202 <__x86.get_pc_thunk.ax>
  0x004011f1 <+8>: add eax,0x2e0f
  0x004011f6 <+13>: call 0x4011b9 <getInput>` We can see there is Function call "`getInput`"
  0x004011fb <+18>: mov eax,0x0
  0x00401200 <+23>: pop ebp
  0x00401201 <+24>: ret 
  End of assembler dump.

####   We should check this out, disassamble the function called "`getInput`"
  
  pwndbg> disas getInput
  Dump of assembler code for function getInput:
  0x004011b9 <+0>: push ebp
  0x004011ba <+1>: mov ebp,esp
` 0x004011bc <+3>: push ebx`
` 0x004011bd <+4>: sub esp,0x8` This is how we know our Buffer is 8 bytes, because ESP is 8 bytes
` 0x004011c0 <+7>: call 0x4010c0 <__x86.get_pc_thunk.bx>`
` 0x004011c5 <+12>: add ebx,0x2e3b`
` 0x004011cb <+18>: lea eax,[ebp-0xc]`
` 0x004011ce <+21>: push eax`
` 0x004011cf <+22>: call 0x401040 <gets@plt>` Something gets call (8 bytes)
` 0x004011d4 <+27>: add esp,0x4`
` 0x004011d7 <+30>: lea eax,[ebp-0xc]`
` 0x004011da <+33>: push eax`
` 0x004011db <+34>: call 0x401050 <puts@plt>` Something gets print
` 0x004011e0 <+39>: add esp,0x4`
` 0x004011e3 <+42>: nop`
` 0x004011e4 <+43>: mov ebx,DWORD PTR [ebp-0x4]`
` 0x004011e7 <+46>: leave `
` 0x004011e8 <+47>: ret ` returns to Main
`End of assembler dump.`

####   All right now, if right at the beggining before we break into "`getInput`" & "`gets`"

Let's examine ESP:

`pwndbg> x/8xw $esp`
`0xbffff234: 0x00000000 0x00401219 0x00000000 0xbffff248`
`0xbffff244: 0x004011fb 0x00000000 0xb7dea9a1 0x00000002`

####   Now let me explain:

`0xbffff234: 0x00000000 0x00401219 <-- This is ESP 8 bytes`

####   if you fill all 8 bytes and keep going until here:

`0xbffff234: 0x00000000 0x00401219 0x00000000 <-- then you ll have ESP + EBP`

####   but you wont have any control of it yet, you need to take control of `Return`

####     Essentially in order to BO you need to fill Return with whatever ("C"s), but if you want to take control of it you need shellcode.

####   You can see where `Return` is, here:

`0xbffff234: 0x00000000 0x00401219 0x00000000 0xbffff248 <-- the Last four bytes`

####   Lets do one final review again:

`pwndbg> x/8xw $esp`
`0xbffff234: 0x00000000 0x00401219 (8 bytes of ESP) 0x00000000(4 bytes of EBP) 0xbffff248 (4 bytes of Return ;)`
`0xbffff244: 0x004011fb 0x00000000 0xb7dea9a1 0x00000002`

####   Now let me show you how i fill each register with 8 bytes of "A"s and 4 bytes of "B"s and finally another 4 bytes of "C"s

####   `# Here you can see how i filled ESP with "A"s, 0x41 = A`

`pwndbg> x/8xw $esp`

`0xbffff234: 0x41414141 0x41414141 0x00000000 0xbffff248`
`0xbffff244: 0x004011fb 0x00000000 0xb7dea9a1 0x00000002`

####   Here you can see how i keep filling ESP so that my "B"s goes to EBP, 0x42 = B

`pwndbg> x/8xw $esp`
`0xbffff234: 0x41414141 0x41414141 0x42424242 0xbffff200`
`0xbffff244: 0x004011fb 0x00000000 0xb7dea9a1 0x00000002`

And finally here we BO Return with "C"s, 0x43 = C

`pwndbg> x/8xw $esp`
`0xbffff234: 0x41414141 0x41414141 0x42424242 0x43434343`
`0xbffff244: 0x00401100 0x00000000 0xb7dea9a1 0x00000002`

Now in order to take control of it you need Shellcode, but that i ll leave for another blog post.

Until then =)
