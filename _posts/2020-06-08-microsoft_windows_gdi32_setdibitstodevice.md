---
layout: post
title: "Windows GDI SetDIBitsToDevice - Information Disclosure"
excerpt: "An info-leak in Windows graphics device interface (GDI)."
modified: 2020-06-08
categories: blog
tags: ["Windows GDI"]
---

# Overview

An **Information Disclosure** can be triggered by running a specially crafted program. The issue was caused by the lack of user-supplied value check before dereferencing it as a pointer in the **GDI32!SetDIBitsToDevice** function. An attacker can exploit this vulnerability to achieve a sensitive information disclosure.

# Crash Analysis

Initial crash:
```
0:000> g
ModLoad: 742a0000 742cb000   C:\Windows\SysWOW64\IMM32.DLL
ModLoad: 768d0000 769f0000   C:\Windows\SysWOW64\MSCTF.dll
ModLoad: 74060000 7411e000   C:\Windows\SysWOW64\msvcrt.dll
(2570.2010): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=00808276 ebx=00000001 ecx=00000080 edx=00000000 esi=00808076 edi=00828db8
eip=76f9e7e3 esp=007cf31c ebp=007cf324 iopl=0         nv up ei pl nz ac pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010216
ntdll!memcpy+0x33:
76f9e7e3 f3a5            rep movs dword ptr es:[edi],dword ptr [esi]
```

# Bug Analysis

Before the user-supplied pointer is dereferenced, a chunk of memory of user-controlled size is allocated by the **RtlAllocateHeap** function. However, there is a bit check to ensure that certain flag is enabled but this can be easily bypassed by masking the address of the user pointer with `0x3`. Thus shifted the address which may not be suitable in some conditions.

```
.text:000000018001CC8A                 mov     rax, gs:30h
.text:000000018001CC93                 and     dword ptr [rax+808h], 0
.text:000000018001CC9A                 lea     r8, [rsp+158h+Size]
.text:000000018001CCA2                 mov     edx, r12d
.text:000000018001CCA5                 mov     rcx, r14
.text:000000018001CCA8                 call    hrBitmapScanSize
.text:000000018001CCAD                 test    eax, eax
.text:000000018001CCAF                 js      loc_180054B9B
.text:000000018001CCB5                 mov     r14d, 1
.text:000000018001CCBB                 mov     esi, r14d
.text:000000018001CCBE                 test    byte ptr [rsp+158h+lpvBits], 3   <-- bit check
.text:000000018001CCC6                 jnz     loc_18001CDFC

.text:000000018001CDFC loc_18001CDFC:                          ; CODE XREF: SetDIBitsToDevice+146↑j
.text:000000018001CDFC                 mov     edi, dword ptr [rsp+158h+Size]
.text:000000018001CE03                 mov     rcx, gs:60h
.text:000000018001CE0C                 mov     r8d, edi
.text:000000018001CE0F                 xor     edx, edx
.text:000000018001CE11                 mov     rcx, [rcx+30h]
.text:000000018001CE15                 call    cs:__imp_RtlAllocateHeap
.text:000000018001CE1B                 mov     [rsp+158h+var_70], rax
.text:000000018001CE23                 mov     [rsp+158h+var_78], rax
.text:000000018001CE2B                 test    rax, rax
.text:000000018001CE2E                 jz      loc_18001CCCC
.text:000000018001CE34
.text:000000018001CE34 loc_18001CE34:                          ; DATA XREF: .rdata:0000000180150574↓o
.text:000000018001CE34 ;   __try { // __except at loc_18001CE5C ; Size
.text:000000018001CE34                 mov     r8d, edi
.text:000000018001CE37                 mov     rdx, [rsp+158h+lpvBits] ; Src
.text:000000018001CE3F                 mov     rcx, rax        ; Dst
.text:000000018001CE42                 call    memcpy_0
.text:000000018001CE47                 mov     rax, [rsp+158h+var_70]
.text:000000018001CE4F                 mov     [rsp+158h+lpvBits], rax
.text:000000018001CE57                 jmp     loc_18001CCCC
```

This is how it looks like once the **GDI32!SetDIBitsToDevice** is hit, all the arguments passed can be found on the stack.

```
0:000> g
ModLoad: 742a0000 742cb000   C:\Windows\SysWOW64\IMM32.DLL
ModLoad: 768d0000 769f0000   C:\Windows\SysWOW64\MSCTF.dll
ModLoad: 74060000 7411e000   C:\Windows\SysWOW64\msvcrt.dll
Breakpoint 0 hit
*** WARNING: Unable to verify checksum for SetDIBitsToDevice.exe
eax=00e7fe18 ebx=7faac000 ecx=00000000 edx=99010a8d esi=0758efdb edi=99010a8d
eip=76e91f50 esp=00e7fddc ebp=00e7fe48 iopl=0         nv up ei pl nz na po nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000202
GDI32!SetDIBitsToDevice:
76e91f50 6a68            push    68h
0:000> dd esp L13
00e7fddc  002c10e6 99010a8d 00000000 00000000   <-- *(00e7fddc+4) is the starting of the arguments
00e7fdec  00000020 00000020 00000000 00000000
00e7fdfc  00000000 00000020 0758efdb 00e7fe18   <-- these two addresses is lpvBits and a pointer to BITMAPINFO structure
00e7fe0c  00000000 07589de8 73f92108 00000028   <-- *(00e7fe0c) is DIR_RGB_COLORS
00e7fe1c  00000020 00000020 00040001
0:000> dd 00e7fe18
00e7fe18  00000028 00000020 00000020 00040001   <-- BITMAPINFO structure
00e7fe28  00000000 00000b12 00000b12 00000010
00e7fe38  00000010 0066cccc 00ffffff 8a3557da   <-- 0x00ffffff is the BITMAPINFO.RGBQUAD array
00e7fe48  00e7fe90 002c12ea 00000001 07589de8
00e7fe58  07584370 8a355702 002c1372 002c1372
00e7fe68  7faac000 00000000 00000000 00000000
00e7fe78  00e7fe5c 00000000 00e7fedc 002c1acb
00e7fe88  8afe8f62 00000000 00e7fea4 74773744
```

The memory right after the **BITMAPINFO.RGBQUAD** is where the color table data is stored and we can use `lpvBits` pointer to leak any address within this address space. There are many methods to achieve info-leak and one of them is by using **COLORREF**.

```
00e7fe38                             8a3557da
00e7fe48  00e7fe90 002c12ea 00000001 07589de8
00e7fe58  07584370 8a355702 002c1372 002c1372
00e7fe68  7faac000 00000000 00000000 00000000
00e7fe78  00e7fe5c 00000000 00e7fedc 002c1acb
00e7fe88  8afe8f62 00000000 00e7fea4 74773744
```