---
layout: post
title: "Foxit Reader 10 - NULL Pointer Dereference Remote Code Execution"
excerpt: "A non-trivial pdf reader pointer reference leads to a possible RCE."
modified: 2020-05-25
categories: blog
tags: ["Foxit Reader 10"]
---

# Overview

A **NULL Pointer Dereference** can be triggered by opening a specially crafted PDF file using **Foxit Reader** version 10.0.0.35798. The issue was caused by an unchecked pointer in **FoxitReader!safe_vsnprintf** function. An attacker can trigger a read from address zero which could leads to a code execution or a denial of service.

# Crash Analysis

```
(1c38.1e00): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=00000000 ebx=00000000 ecx=06758a14 edx=0a0d0000 esi=32b98fd0 edi=06758a34
eip=027fed2e esp=0675899c ebp=06758a0c iopl=0         nv up ei pl nz ac pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00210216
FoxitReader!safe_vsnprintf+0x9c4ffe:
027fed2e 8b10            mov     edx,dword ptr [eax]  ds:002b:00000000=????????
```

# Bug Analysis

So this is where the bug was initially triggered. The `eax` is directly obtained after a call to **FoxitReader!safe_vsnprintf+0x9c1b30** function. There is no check whether the pointer is NULL.

```
// where null ptr was deref'ed
027fed10 0f8d3d010000    jge     FoxitReader!safe_vsnprintf+0x9c5123 (027fee53)
027fed16 c7450800000000  mov     dword ptr [ebp+8],0
027fed1d 8bce            mov     ecx,esi
027fed1f c645fc09        mov     byte ptr [ebp-4],9

// directly jump here, skipped the above path
027fed23 e838cbffff      call    FoxitReader!safe_vsnprintf+0x9c1b30 (027fb860)
027fed28 8d4d08          lea     ecx,[ebp+8]
027fed2b 51              push    ecx
027fed2c 6a00            push    0
027fed2e 8b10            mov     edx,dword ptr [eax]  ds:002b:00000000=????????
0216ed30 8bc8            mov     ecx,eax
0216ed32 ff5208          call    dword ptr [edx+8]
```

The `eax` was a returned value from a call to a function pointer. The function was referenced at a fixed memory address which is **0x04372174**.

```
// call FoxitReader!safe_vsnprintf+0x9e3610, where got checks all over the place
027fb917 8d4de8          lea     ecx,[ebp-18h]
027fb91a 51              push    ecx
027fb91b 8bc8            mov     ecx,eax
027fb91d 8b10            mov     edx,dword ptr [eax]
027fb91f ff5210          call    dword ptr [edx+10h]  ds:002b:04372174=0281d340     ; call FoxitReader!safe_vsnprintf+0x9e3610
027fb922 8bf0            mov     esi,eax 	; store null to esi
027fb924 8d4df0          lea     ecx,[ebp-10h]
027fb927 e8447ca2ff      call    FoxitReader!safe_vsnprintf+0x3e9840 (02223570)
027fb92c 8bc6            mov     eax,esi 	; make sure eax is still null
027fb92e 8b4df4          mov     ecx,dword ptr [ebp-0Ch]
027fb931 64890d00000000  mov     dword ptr fs:[0],ecx
027fb938 59              pop     ecx
027fb939 5f              pop     edi
027fb93a 5e              pop     esi
027fb93b 8be5            mov     esp,ebp
027fb93d 5d              pop     ebp
027fb93e c3              ret
```

Inside the **FoxitReader!safe_vsnprintf+0x9e3610**, we can see that the `eax` register is set to NULL.

```
// FoxitReader!safe_vsnprintf+0x9e3610:
0281d340 55              push    ebp
0281d341 8bec            mov     ebp,esp
0281d343 6aff            push    0FFFFFFFFh
0281d345 68f284f203      push    offset FoxitReader!CFXJSE_Arguments::GetValue+0x15b5622 (03f284f2)
0281d34a 64a100000000    mov     eax,dword ptr fs:[00000000h]
0281d350 50              push    eax
0281d351 83ec24          sub     esp,24h
0281d354 53              push    ebx

// then set eax to null
0281d449 8b7508          mov     esi,dword ptr [ebp+8]
0281d44c 837e0402        cmp     dword ptr [esi+4],2
0281d450 7d20            jge     FoxitReader!safe_vsnprintf+0x9e3742 (0281d472)
0281d452 33c0            xor     eax,eax 	; set to null
0281d454 8b4df4          mov     ecx,dword ptr [ebp-0Ch]
0281d457 64890d00000000  mov     dword ptr fs:[0],ecx
0281d45e 59              pop     ecx
0281d45f 5f              pop     edi
0281d460 5e              pop     esi
0281d461 5b              pop     ebx
0281d462 8be5            mov     esp,ebp
0281d464 5d              pop     ebp
0281d465 c20400          ret     4
```

There is also a small possibility to gain control of the memory address at `edx+10h` by manipulating the allocated heap but I have not found any tricks to do that yet. So I will just leave it here for the readers.

```
0:000> !heap -p -a eax
    address 2f448fc0 found in
    _DPH_HEAP_ROOT @ b0e1000
    in busy allocation (  DPH_HEAP_BLOCK:         UserAddr         UserSize -         VirtAddr         VirtSize)
                                2f440958:         2f448fc0               40 -         2f448000             2000
          ? FoxitReader!google::LogMessage::kMaxLogMessageLen+30e198
    71cfab70 verifier!AVrfDebugPageHeapAllocate+0x00000240
    775b909b ntdll!RtlDebugAllocateHeap+0x00000039
    7750bbad ntdll!RtlpAllocateHeap+0x000000ed
    7750b0cf ntdll!RtlpAllocateHeapInternal+0x0000022f
    7750ae8e ntdll!RtlAllocateHeap+0x0000003e
    0394bb00 FoxitReader!CFXJSE_Arguments::GetValue+0x00fd8c30
    0222857b FoxitReader!safe_vsnprintf+0x003ee84b
    02228b06 FoxitReader!safe_vsnprintf+0x003eedd6
    02228723 FoxitReader!safe_vsnprintf+0x003ee9f3
    0281b677 FoxitReader!safe_vsnprintf+0x009e1947
    02808458 FoxitReader!safe_vsnprintf+0x009ce728
    027fc78f FoxitReader!safe_vsnprintf+0x009c2a5f
    02827dd3 FoxitReader!safe_vsnprintf+0x009ee0a3
    02825fe7 FoxitReader!safe_vsnprintf+0x009ec2b7
    0282659c FoxitReader!safe_vsnprintf+0x009ec86c
    02824c64 FoxitReader!safe_vsnprintf+0x009eaf34
    0282571c FoxitReader!safe_vsnprintf+0x009eb9ec
    028611f3 FoxitReader!safe_vsnprintf+0x00a274c3
    00a11396 FoxitReader!std::basic_ostream<char,std::char_traits<char> >::put+0x00039366
    00a10f7a FoxitReader!std::basic_ostream<char,std::char_traits<char> >::put+0x00038f4a
    009c9adf FoxitReader!std::basic_ostream<char,std::char_traits<char> >::operator<<+0x000eab8f
    009e1c8f FoxitReader!std::basic_ostream<char,std::char_traits<char> >::put+0x00009c5f
    009e1f45 FoxitReader!std::basic_ostream<char,std::char_traits<char> >::put+0x00009f15
    03678916 FoxitReader!CFXJSE_Arguments::GetValue+0x00d05a46
    036786f8 FoxitReader!CFXJSE_Arguments::GetValue+0x00d05828
    03680fa9 FoxitReader!CFXJSE_Arguments::GetValue+0x00d0e0d9
    0369a6b1 FoxitReader!CFXJSE_Arguments::GetValue+0x00d277e1
    00bd4eb7 FoxitReader!std::basic_ios<char,std::char_traits<char> >::fill+0x00028cd7
    009ec531 FoxitReader!std::basic_ostream<char,std::char_traits<char> >::put+0x00014501
    0367371f FoxitReader!CFXJSE_Arguments::GetValue+0x00d0084f
    0368106b FoxitReader!CFXJSE_Arguments::GetValue+0x00d0e19b
    0369a6f4 FoxitReader!CFXJSE_Arguments::GetValue+0x00d27824
```