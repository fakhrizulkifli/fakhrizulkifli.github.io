---
layout: post
title: "Suricata - ReadErfFile Stack-based Buffer Overflow"
date: 2020-04-03
categories:
    - blog
keywords: "suricata, endace dag,  stack overflow, overflow, buffer overflow"
---

Another day another bug. The bug can be triggered by a specially crafted ERF file. The Extensible Record Format (ERF) is a trace file produced by the Endace DAG cards and can be consumed by Suricata and Wireshark. In this case, the bug is in the Suricata's `ReadErfFile` function.

The payload can be crafted based on their packet format documentation [here](https://www.endace.com/erf-extensible-record-format-types.pdf) and [here](https://wiki.wireshark.org/ERF)

Proof-of-Crash:
```
#0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:51
#1  0x00007ffff5783801 in __GI_abort () at abort.c:79
#2  0x00007ffff57cc5e5 in __libc_message (action=(do_abort | do_backtrace), fmt=0x7ffff58f9796 "%s", fmt=0x7ffff58f9796 "%s", 
    action=(do_abort | do_backtrace)) at ../sysdeps/posix/libc_fatal.c:181
#3  0x00007ffff57cc92a in __GI___libc_fatal (
    message=message@entry=0x7ffff58fb3b8 "Fatal error: glibc detected an invalid stdio handle\n")
    at ../sysdeps/posix/libc_fatal.c:191
#4  0x00007ffff57cd1d7 in _IO_vtable_check () at vtables.c:72
#5  0x00007ffff57ce76a in IO_validate_vtable (vtable=0x4141414141414141) at libioP.h:876
#6  __GI__IO_file_xsgetn (fp=0x7fffec268b20, data=<optimized out>, n=18446744073709551614) at fileops.c:1364
#7  0x00007ffff57c23c1 in __GI__IO_fread (buf=<optimized out>, size=18446744073709551614, count=count@entry=1, fp=0x7fffec268b20)
    at iofread.c:38
#8  0x000055555570adc0 in fread (__stream=<optimized out>, __n=1, __size=<optimized out>, __ptr=<optimized out>)
    at /usr/include/x86_64-linux-gnu/bits/stdio2.h:294
#9  ReadErfRecord (p=p@entry=0x7fffec268180, data=data@entry=0x7fffec268d50, tv=<optimized out>) at source-erf-file.c:170
#10 0x000055555570b346 in ReceiveErfFileLoop (tv=<optimized out>, data=0x7fffec268d50, slot=<optimized out>) at source-erf-file.c:137
#11 0x0000555555731aa4 in TmThreadsSlotPktAcqLoop (td=0x555556c09b10) at tm-threads.c:300
#12 0x00007ffff69816db in start_thread (arg=0x7ffff38b0700) at pthread_create.c:463
#13 0x00007ffff586488f in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```