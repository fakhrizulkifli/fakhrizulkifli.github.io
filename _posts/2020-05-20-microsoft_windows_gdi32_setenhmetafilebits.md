---
layout: post
title: "Windows GDI SetEnhMetaFileBits - Out-of-Bounds Read"
excerpt: "An info-leak in Windows graphics device interface (GDI)."
modified: 2020-05-20
categories: blog
tags: ["Windows GDI"]
---

# Overview

An **Out-of-Bounds Read** can triggered by running a specially crafted program. The issue was caused by a lack of bounds check and initialization in the **GDI32!SetEnhMetaFileBits** function. An attacker can trigger an uninitialized memory read within the allocated address.

# Crash Analysis

Initial crash:
```
ModLoad: 75ff0000 7601b000   C:\Windows\SysWOW64\IMM32.DLL
ModLoad: 77a10000 77b30000   C:\Windows\SysWOW64\MSCTF.dll
ModLoad: 77bb0000 77c6e000   C:\Windows\SysWOW64\msvcrt.dll
(4a4.1844): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
*** WARNING: Unable to verify checksum for SetEnhMetaFileBits.exe
eax=800fdf70 ebx=00000000 ecx=00000000 edx=00000000 esi=77527e70 edi=800fdf70
eip=775283dc esp=00b4f968 ebp=00b4f970 iopl=0         nv up ei ng nz ac pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010296
GDI32!SetEnhMetaFileBits+0x1c:
775283dc 8b4230          mov     eax,dword ptr [edx+30h] ds:002b:00000030=????????
```

# Bug Analysis

This is how it was initially triggered by my harness. We can clearly see that the 2nd argument is NULL.
```
eax=800fdf70 ebx=00000000 ecx=00000000 edx=00ff8360 esi=77527e70 edi=800fdf70
eip=775283c0 esp=00d2f730 ebp=00d2f748 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
GDI32!SetEnhMetaFileBits:
775283c0 8bff            mov     edi,edi
0:000> dd esp
00d2f730  00c210cb 800fdf70 00000000 00ff3fb8
00d2f740  71fe2108 7eb77000 00d2f790 00c212b5
00d2f750  1f460d5d 00ff3fb8 00ff5ed0 5b6e5a06
00d2f760  00c2133d 00c2133d 7eb77000 00000000
00d2f770  00000000 00000000 00d2f75c 00000000
00d2f780  00d2f7dc 00c21a6b 5b7e88d6 00000000
00d2f790  00d2f7a4 75eb3744 7eb77000 75eb3720
00d2f7a0  33b18477 00d2f7ec 77d79e54 7eb77000
```

We can confirm with the disassembly code below that there is no initialization to the allocated memory.

```
.text:0000000180040089                 test    r13b, 1
.text:000000018004008D                 jnz     loc_18004019F
.text:0000000180040093                 mov     edx, [rsi+30h]  ; uBytes
.text:0000000180040096                 xor     ecx, ecx        ; uFlags
.text:0000000180040098                 call    cs:__imp_LocalAlloc
.text:000000018004009E                 mov     [rbx+20h], rax
.text:00000001800400A2                 test    rax, rax
.text:00000001800400A5                 jz      loc_18006C72D
.text:00000001800400AB                 mov     r8d, [rsi+30h]  ; Size
.text:00000001800400AF                 mov     rdx, rsi        ; Src
.text:00000001800400B2                 mov     rcx, rax        ; Dst
.text:00000001800400B5                 call    memcpy_0
.text:00000001800400BA                 mov     ecx, [rsi+30h]
.text:00000001800400BD                 mov     rax, [rbx+20h]
.text:00000001800400C1                 mov     [rbx+30h], rax
.text:00000001800400C5                 mov     [rbx+38h], rcx
```

With a little bit of modification to the harness. The user can control the `Size` passed to the `ntdll!memcpy` and thus control how much bytes to be leaked. So from here we can find a way to leak the uninitialized data but it is trivial so I will leave it as an exercise to the readers :).

Here is how it looks like once everything is set in the right way. Have fun!

```
eax=00000000 ebx=00fd8fe8 ecx=00000001 edx=00fd8808 esi=77527e70 edi=800fdf70
eip=775283c0 esp=00d7f7fc ebp=00d7f814 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
GDI32!SetEnhMetaFileBits:
775283c0 8bff            mov     edi,edi
0:000> dd esp
00d7f7fc  009c10d8 800fdf70 00fd8fe8 00fd3fb8
00d7f80c  71fe2108 7f1c5000 00d7f85c 009c12cb
00d7f81c  6b4610c4 00fd3fb8 00fd5108 42630ad1
00d7f82c  009c1353 009c1353 7f1c5000 00000000
00d7f83c  00000000 00000000 00d7f828 00000000
00d7f84c  00d7f8a8 009c1aab 4228d595 00000000
00d7f85c  00d7f870 75eb3744 7f1c5000 75eb3720
00d7f86c  56109131 00d7f8b8 77d79e54 7f1c5000
0:000> g
(d90.394): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=00000000 ebx=00fd8fe8 ecx=006c6006 edx=00000000 esi=02af0ffc edi=04609034
eip=77d8e8e3 esp=00d7f794 ebp=00d7f79c iopl=0         nv dn ei pl nz ac pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010616
ntdll!memcpy+0x213:
77d8e8e3 f3a5            rep movs dword ptr es:[edi],dword ptr [esi]
0:000> kv
 # ChildEBP RetAddr  Args to Child              
00 00d7f79c 774feed1 02af1020 00fd8fe8 41414141 ntdll!memcpy+0x213
01 00d7f7d4 775283f7 00000000 00000000 00000000 GDI32!pmfAllocMF+0x36197
02 00d7f7f8 009c10d8 800fdf70 00fd8fe8 00fd3fb8 GDI32!SetEnhMetaFileBits+0x37 (FPO: [2,0,0])
03 00d7f814 009c12cb 6b4610c4 00fd3fb8 00fd5108 SetEnhMetaFileBits!main+0x98 (FPO: [Non-Fpo]) (CONV: cdecl) [c:\users\user\desktop\harness\gdi32.dll\setenhmetafilebits\setenhmetafilebits\setenhmetafilebits.cpp @ 31] 
04 (Inline) -------- -------- -------- -------- SetEnhMetaFileBits!invoke_main+0x1c (Inline Function @ 009c12cb) (CONV: cdecl) [d:\agent\_work\3\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78] 
05 00d7f85c 75eb3744 7f1c5000 75eb3720 56109131 SetEnhMetaFileBits!__scrt_common_main_seh+0xfa (FPO: [Non-Fpo]) (CONV: cdecl) [d:\agent\_work\3\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288] 
06 00d7f870 77d79e54 7f1c5000 961a92d3 00000000 KERNEL32!BaseThreadInitThunk+0x24 (FPO: [Non-Fpo])
07 00d7f8b8 77d79e1f ffffffff 77d9d6d2 00000000 ntdll!__RtlUserThreadStart+0x2f (FPO: [Non-Fpo])
08 00d7f8c8 00000000 009c1353 7f1c5000 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [Non-Fpo])
```