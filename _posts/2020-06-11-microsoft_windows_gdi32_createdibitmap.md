---
layout: post
title: "Windows GDI CreateDIBitmap - Information Disclosure"
excerpt: "An info-leak in Windows graphics device interface (GDI)."
modified: 2020-06-11
categories: blog
tags: ["Windows GDI"]
---

# Overview

An **Information Disclosure** can be triggered by running a specially crafted program. The issue was caused by the lack of user-supplied value check before dereferencing it as a pointer in the **GDI32!CreateDIBitmap** function. An attacker can exploit this vulnerability to achieve a sensitive information disclosure.

# Crash Analysis

Initial crash:
```
ModLoad: 75ff0000 7601b000   C:\Windows\SysWOW64\IMM32.DLL
ModLoad: 77a10000 77b30000   C:\Windows\SysWOW64\MSCTF.dll
ModLoad: 77bb0000 77c6e000   C:\Windows\SysWOW64\msvcrt.dll
(19f0.1370): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=005b6d7e ebx=005b6b7e ecx=00000080 edx=00000000 esi=005b6b7e edi=005afdb8
eip=77d8e703 esp=0044fb14 ebp=0044fb1c iopl=0         nv up ei pl nz ac pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010216
ntdll!memcpy+0x33:
77d8e703 f3a5            rep movs dword ptr es:[edi],dword ptr [esi]
```

# Bug Analysis

The 4th argument is initially passed as a pointer to an array of uninitialized data which later passed as the source for `memcpy` to an allocated heap of fixed size.

```
eax=06fd67ae ebx=00000276 ecx=06fce746 edx=00000000 esi=40010c5f edi=06fce738
eip=774c1c40 esp=0090f7d0 ebp=0090f804 iopl=0         nv up ei pl nz na po nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000202
GDI32!CreateDIBitmap:
774c1c40 8bff            mov     edi,edi
0:000> dd esp
0090f7d0  0012113c 40010c5f 06fce746 00000004
0090f7e0  06fd67ae 06fce746 00000000 06fc3f78   <-- 06fd67ae is 4th arg
0090f7f0  71fe2108 7f1c7000 40010c5f 00000276
0090f800  00000020 0090f84c 00121341 00000000
0090f810  06fc3f78 06fc47f0 09ff0d71 001213c9
0090f820  001213c9 7f1c7000 00000000 00000000
0090f830  00000000 0090f818 00000000 0090f898
0090f840  00121afb 097dd075 00000000 0090f860
0:000> dd 06fd67ae
06fd67ae  ???????? ???????? ???????? ????????
06fd67be  ???????? ???????? ???????? ????????
06fd67ce  ???????? ???????? ???????? ????????
06fd67de  ???????? ???????? ???????? ????????
06fd67ee  ???????? ???????? ???????? ????????
06fd67fe  ???????? ???????? ???????? ????????
06fd680e  ???????? ???????? ???????? ????????
06fd681e  ???????? ???????? ???????? ????????
```

However, before reaching to the part where the input is copied. There is a check to test if the 3rd bit of the pointer address is set or not.

```
.text:000000018001BBBA loc_18001BBBA:                          ; CODE XREF: CreateDIBitmap+F0↑j
.text:000000018001BBBA                                         ; CreateDIBitmap+38A78↓j
.text:000000018001BBBA                 cmp     r14, 0FFFFFFFFFFFFFFFFh
.text:000000018001BBBE                 jnz     loc_18001BC49
.text:000000018001BBC4                 test    r13b, 3
.text:000000018001BBC8                 jnz     loc_18005451D
.text:000000018001BBCE                 mov     r14d, dword ptr [rbp+47h+Size]
.text:000000018001BBD2                 mov     r15, rbx
```

Once the check is satisfied, the **CreateDIBitmap** copies the uninitialized data to the allocated heap of size `0x200` and then triggers the **Invalid Pointer Read** error.

```
eax=06fcfdb8 ebx=06fd67ae ecx=06fcfdb8 edx=00000000 esi=06fcfdb8 edi=06fce746
eip=77d8e6d0 esp=0090f77c ebp=0090f7cc iopl=0         nv up ei pl nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
ntdll!memcpy:
77d8e6d0 55              push    ebp
0:000> dd esp
0090f77c  774fb352 06fcfdb8 06fd67ae 00000200
0090f78c  06fce738 40010c5f 00000276 00000000
0090f79c  03bd09c0 00000000 00000020 00000020
0090f7ac  e22d07cf 06fcfdb8 00000068 06fd67ae
0090f7bc  00000200 00000000 00000000 ffffffff
0090f7cc  0090f804 0012113c 40010c5f 06fce746
0090f7dc  00000004 06fd67ae 06fce746 00000000
0090f7ec  06fc3f78 71fe2108 7f1c7000 40010c5f
0:000> dd 06fcfdb8
06fcfdb8  003f005c 005c003f 003a0043 0055005c
06fcfdc8  00650073 00730072 0075005c 00650073
06fcfdd8  005c0072 00650044 006b0073 006f0074
06fcfde8  005c0070 00610068 006e0072 00730065
06fcfdf8  005c0073 00640067 00330069 002e0032
06fcfe08  006c0064 005c006c 00720043 00610065
06fcfe18  00650074 00490044 00690042 006d0074
06fcfe28  00700061 0052005c 006c0065 00610065
0:000> dd 06fd67ae
06fd67ae  ???????? ???????? ???????? ????????
06fd67be  ???????? ???????? ???????? ????????
06fd67ce  ???????? ???????? ???????? ????????
06fd67de  ???????? ???????? ???????? ????????
06fd67ee  ???????? ???????? ???????? ????????
06fd67fe  ???????? ???????? ???????? ????????
06fd680e  ???????? ???????? ???????? ????????
06fd681e  ???????? ???????? ???????? ????????
```

# Proof-of-Concept

To achieve infoleak, **CBM_INIT** must be set so that the system would use the data pointed by the 4th argument because we are going to fill it with the index of the color table data and the color table must not be initialized. The heap allocation must be large enough to make sure the masked pointer still points to our filled data.

```
HBITMAP hbmp = CreateDIBitmap(hdc,
	&(bmi.bmiHeader),
	CBM_INIT,
	lpBits,
	&bmi,
	DIB_RGB_COLORS);
```

Here is how it looks like on the stack:

```
eax=007efdf0 ebx=7fb69000 ecx=00000000 edx=83010c69 esi=0095fc9b edi=83010c69
eip=774c1c40 esp=007efdc8 ebp=007efe20 iopl=0         nv up ei pl nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
GDI32!CreateDIBitmap:
774c1c40 8bff            mov     edi,edi
0:000> dd esp
007efdc8  013610d9 83010c69 007efdf0 00000004   <-- 6 arguments
007efdd8  0095fc9b 007efdf0 00000000 0095a1b8
007efde8  71fe2108 71f3c42d 00000028 00000020   <-- *(007efde8+8) is the BITMAPINFO structure
007efdf8  00000020 00040001 00000000 00000b12
007efe08  00000b12 00000010 00000010 0066cccc
007efe18  00ffffff 846be04d 007efe68 013612f2   <-- *(007efe18+4) is the color table
007efe28  00000001 0095a1b8 00954790 846be005
007efe38  0136137a 0136137a 7fb69000 00000000
```