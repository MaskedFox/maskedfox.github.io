---
title: "How to use GDB when Doing Dynamic Reversing"
layout: post
date: 2019-01-22 16:45
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
description: Basics of doing Dynamic Reversing with GDB
---

     #          **How to use GDB when Doing Dynamic Reversing**

For the last year i been doing reversing challenges and of them contained a buffer overflow. Anyways, on my last challenge which was a real life binary, i found myself not knowing how to use gdb-multi  doing a remote debugging and it super bother me because nexti or next didnt work as expect so i decided to do a quick review on it. I first started watching this [video from Pentester Academy](https://www.youtube.com/watch?v=RF7DF4kfs1E&list=PLwkhI3Ao_JDuCzfurQrti8_wtWnfNoPEy&index=2&t=18s)
and after i typed the code and tried the buffer overflow myself i found using step and next, i guess in real life sometimes Next or some commands wont work as expect it since we dont have any symbols etc. So this is what i learned (again). =D

####     **My GDB Cheatsheet:**

**Commands:**

`Step in: si`

`next in: ni`

###`si` its to get inside a function, for example if i do` si` on `0x4011f6 <main+13> call getInput <0x4011b9>`, i would get ### into the funciton, example of using si`on the function above:

```
   0x4011b9        push   ebp
   0x4011ba      mov    ebp, esp
   0x4011bc      push   ebx
   0x4011bd      sub    esp, 8
   0x4011c0      call   __x86.get_pc_thunk.bx <0x4010c0>

   0x4011c5     add    ebx, 0x2e3b
   0x4011cb     lea    eax, [ebp - 0xc]
   0x4011ce     push   eax
   0x4011cf     call   gets@plt <0x401040>

   0x4011d4     add    esp, 4
   0x4011d7     lea    eax, [ebp - 0xc]

```

However, at this point we dont want to use `si`anymore, what you want to use is` ni`

So you can go through each line without getting into a function again, unless thats what you want to do.

So another `Command `that is cool is`return` or `**finish**`

`return` forces the current function to return inmediately, passing the giving value

*finish* continues until the current function returns

A pretty good Cheatsheet on GDB is this [one](https://darkdust.net/files/GDB%20Cheat%20Sheet.pdf)
