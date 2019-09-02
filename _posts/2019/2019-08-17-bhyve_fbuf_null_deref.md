---
layout: post
title: "BSD Hypervisor - pci_fbuf NULL Pointer Dereference"
date: 2019-08-17
categories:
    - blog
keywords: "bhyve, bsd, null deref, hypervisor"
---

NOTE: This also works in [xhyve hypervisor](https://github.com/machyve/xhyve){:target="_blank"}.

Affected code:
```
 0x00000000002317ec <+108>:	mov $0x206519,%edi
 0x00000000002317f1 <+113>:	xor %eax,%eax
 0x00000000002317f3 <+115>:	mov %r9d,%esi
 0x00000000002317f6 <+118>:	callq 0x24aa00 <printf@plt>
 0x00000000002317fb <+123>:	mov 0xc8(%rbx),%rax
=> 0x0000000000231802 <+130>:	cmpl $0x0,(%rax)
 0x0000000000231805 <+133>:	movzwl 0xc(%rbx),%eax
 0x0000000000231809 <+137>:	je 0x231830 <pci_fbuf_write+176>
 0x000000000023180b <+139>:	test %ax,%ax
 0x000000000023180e <+142>:	je 0x23185d <pci_fbuf_write+221>
 0x0000000000231810 <+144>:	cmpw $0x0,0xe(%rbx)
 0x0000000000231815 <+149>:	je 0x23185d <pci_fbuf_write+221>
```

Proof-of-Crash:
```
Thread 13 "vcpu 2" received signal SIGSEGV, Segmentation fault.
[Switching to LWP 100571 of process 4094]
0x0000000000231802 in pci_fbuf_write (ctx=<optimized out>, vcpu=<optimized out>, pi=<optimized out>, baridx=<optimized out>, offset=<optimized out>, size=<optimized out>,
 value=1953719668) at /usr/src/usr.sbin/bhyve/pci_fbuf.c:166
166	if (!sc->gc_image->vgamode && sc->memregs.width == 0 &&
(gdb) p sc->gc_image
$1 = (struct bhyvegc_image *) 0x0
(gdb) bt
#0 0x0000000000231802 in pci_fbuf_write (ctx=<optimized out>, vcpu=<optimized out>, pi=<optimized out>, baridx=<optimized out>, offset=<optimized out>, size=<optimized out>,
 value=1953719668) at /usr/src/usr.sbin/bhyve/pci_fbuf.c:166
#1 0x0000000000230522 in pci_emul_mem_handler (ctx=0x80028e080, vcpu=2, dir=<optimized out>, addr=<optimized out>, size=4, val=0x7fffde9f2d40, arg1=0x800a9aa00, arg2=0)
 at /usr/src/usr.sbin/bhyve/pci_emul.c:415
#2 0x0000000000224ae4 in mem_write (ctx=0x80028e080, vcpu=2, gpa=3, wval=1953719668, size=0, arg=0x0) at /usr/src/usr.sbin/bhyve/mem.c:162
#3 0x00000000002486ae in emulate_mov (vm=<optimized out>, vcpuid=<optimized out>, gpa=<optimized out>, vie=<optimized out>, memread=<optimized out>,
 memwrite=<optimized out>, arg=<optimized out>) at /usr/src/sys/amd64/vmm/vmm_instruction_emul.c:517
#4 vmm_emulate_instruction (vm=<optimized out>, vcpuid=<optimized out>, gpa=<optimized out>, vie=<optimized out>, paging=<optimized out>, memread=<optimized out>,
 memwrite=0x224ab0 <mem_write>, memarg=0x800aad100) at /usr/src/sys/amd64/vmm/vmm_instruction_emul.c:1510
#5 0x000000000022448f in emulate_mem_cb (ctx=0x80028e080, vcpu=2, paddr=34370857472, mr=0x0, arg=<optimized out>) at /usr/src/usr.sbin/bhyve/mem.c:238
#6 0x00000000002243da in access_memory (ctx=0x80028e080, vcpu=2, paddr=3221241856, cb=0x224470 <emulate_mem_cb>, arg=0x7fffde9f2ec8) at /usr/src/usr.sbin/bhyve/mem.c:215
#7 0x00000000002242b9 in emulate_mem (ctx=0x80028e080, vcpu=2, paddr=34370857472, vie=<optimized out>, paging=<optimized out>) at /usr/src/usr.sbin/bhyve/mem.c:251
#8 0x000000000021bc75 in vmexit_inst_emul (ctx=0x80028e080, vmexit=0x24f780 <vmexit+272>, pvcpu=<optimized out>) at /usr/src/usr.sbin/bhyve/bhyverun.c:630
#9 0x000000000021b6da in vm_loop (ctx=0x80028e080, vcpu=2, startrip=<optimized out>) at /usr/src/usr.sbin/bhyve/bhyverun.c:748
#10 0x000000000021a969 in fbsdrun_start_thread (param=0x24e050 <mt_vmm_info+48>) at /usr/src/usr.sbin/bhyve/bhyverun.c:353
#11 0x000000080061b776 in ?? () from /lib/libthr.so.3
#12 0x0000000000000000 in ?? ()
Backtrace stopped: Cannot access memory at address 0x7fffde9f3000

(gdb) i registers
rax 0x0 0
rbx 0x800aec000 34371190784
rcx 0x3 3
rdx 0x800a9aa00 34370857472
rsi 0x2 2
rdi 0x80028e080 34362417280
rbp 0x7fffde9f2cd0 0x7fffde9f2cd0
rsp 0x7fffde9f2cc0 0x7fffde9f2cc0
r8 0x0 0
r9 0x4 4
r10 0x231780 2299776
r11 0x7fffde9f2d40 140736928361792
r12 0x0 0
r13 0x80028e080 34362417280
r14 0x800a9aa00 34370857472
r15 0x24b430 2405424
rip 0x231802 0x231802 <pci_fbuf_write+130>
eflags 0x10297 [ CF PF AF SF IF RF ]
cs 0x43 67
ss 0x3b 59
ds <unavailable>
es <unavailable>
fs <unavailable>
gs <unavailable>
fs_base 0x800ac98d0 34371049680
gs_base 0x0 0
```

---
References:
1. https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=240272