---
layout: post
title: Pharo VM GC Crash Summary
---

This blog post will be updated with the current status of my investigations and attempts to resolve this issue.

As people may be returning to this page multiple times it is largely in the reverse of the typical order, i.e. the latest information is at the top, introductory information is near the bottom, in particular the Symptoms section.


### Lines of Investigation

There appear to be a few things happening leading up to the crash:

- The `receiver` and `rcvr/clsr` pointers in the Frame aren't being updated when an object is moved during scavenging and compaction.
- `framePointer` is pointing to a Frame that is located in a free stack page.

### Rejected hypotheses

- FreeType in Pharo has caused multiple problems, but the crashes can be reproduced with minimal Pharo images that don't include FreeType.
- FFI can also be problematic, but nothing is using FFI during the tests, and the base image shows no inconsistencies.


### Symptoms

Crash dumps normally show that a garbage collection of some sort is being performed at the time of the crash, e.g.

```
C stack backtrace & registers:
	rax 0x180000013000034 rbx 0x00000000 rcx 0x180000013000034 rdx 0x00000005
	rdi 0x180000013000034 rsi 0x00000000 rbp 0x7ffdc5a70200 rsp 0x7ffdc5a70200
	r8  0x00000000 r9  0x00000022 r10 0xffffffde r11 0x00000000
	r12 0x004198a0 r13 0x7ffdc5adb010 r14 0x00000000 r15 0x00000000
	rip 0x00476b7c
*vmdbg/lib/pharo/5.0-201911160620/pharo(isMarked+0xc)[0x476b7c]
vmdbg/lib/pharo/5.0-201911160620/pharo[0x41afa4]
vmdbg/lib/pharo/5.0-201911160620/pharo[0x41e0d8]
/lib/x86_64-linux-gnu/libpthread.so.0(+0x12890)[0x7f1b6c953890]
vmdbg/lib/pharo/5.0-201911160620/pharo(isMarked+0xc)[0x476b7c]
vmdbg/lib/pharo/5.0-201911160620/pharo(markAndTrace+0x3f8)[0x4871a8]
vmdbg/lib/pharo/5.0-201911160620/pharo[0x481651]
vmdbg/lib/pharo/5.0-201911160620/pharo(fullGC+0xea)[0x4805fa]
vmdbg/lib/pharo/5.0-201911160620/pharo[0x4c9f65]
vmdbg/lib/pharo/5.0-201911160620/pharo[0x4add41]
vmdbg/lib/pharo/5.0-201911160620/pharo(interpret+0xa661)[0x42a6d1]
vmdbg/lib/pharo/5.0-201911160620/pharo[0x43e032]
vmdbg/lib/pharo/5.0-201911160620/pharo(interpret+0x140)[0x4201b0]
vmdbg/lib/pharo/5.0-201911160620/pharo(main+0x2d6)[0x41d8d6]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xe7)[0x7f1b6c571b97]
vmdbg/lib/pharo/5.0-201911160620/pharo(_start+0x2a)[0x4198ca]
[0x0]


Smalltalk stack dump:
    0x7ffdc5ad1230 I SessionManager>launchSnapshot:andQuit: 0x2b08860: a(n) SessionManager
    0x7ffdc5ad12a0 I [] in SessionManager>snapshot:andQuit: 0x2b08860: a(n) SessionManager
    0x7ffdc5ad12e0 I [] in INVALID RECEIVER>newProcess 0x2789478 is in new space
```

Two attributes tend to increase the chance of this type of crash happening:

1. A large Pharo image
2. The image has been saved many times.


### Related Links

- https://github.com/OpenSmalltalk/opensmalltalk-vm/issues/444
- https://github.com/OpenSmalltalk/opensmalltalk-vm/issues/391
- https://github.com/pharo-project/pharo/issues/5163