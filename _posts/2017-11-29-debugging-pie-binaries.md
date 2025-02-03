---
layout: post
title: "Debugging PIE binaries"
date: 2017-11-29 05:04:48 +0000
categories: blog
---

### Debugging PIE binaries

During HITCON CTF, it is my first time to debug Position Independent Executable binaries, I did have some troubles.

First, check it with checksec, it almost enables all binary-level protections…

```
% checksec -f ./re_easy_to_say-4d171ed2949ad2e9fcb5350c71aa80ecRELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH FORTIFY Fortified Fortifiable  FILEFull RELRO      Canary found      NX enabled    PIE enabled     No RPATH   No RUNPATH   Yes 1  5 ./re_easy_to_say-4d171ed2949ad2e9fcb5350c71aa80ec
```

When exploiting binaries, setting break point is a very important step to watch the control flow. So the question is how to set break point on a PIE binary?

If we try to run it with raw gdb, there is no luck, we can never predict the run-time address of any instruction. All the addresses we saw by objdump are purely relative ones. How could we solve this?

```
gdb
```

```
objdump
```

My idea is to set a break point before we start main(), I was lucky enough to find __libc_start_main() in the binary I played and set a break point there. However, it is not always the case, symbols could be stripped off too. What can we do when we even don’t have any symbols?

```
__libc_start_main()
```

Some Stack Overflow answer suggests to set break point at _start :

```
_start
```

```
(gdb) b _startFunction "_start" not defined.Make breakpoint pending on future shared library load? (y or [n]) yBreakpoint 1 (_start) pending.(gdb) rStarting program: .../hitconf2017/re_easy_to_say-4d171ed2949ad2e9fcb5350c71aa80ec Missing separate debuginfos, use: zypper install glibc-debuginfo-2.22-8.4.x86_64
```

```
Breakpoint 1, 0x00007ffff7ddd1f0 in _start () from /lib64/ld-linux-x86-64.so.2
```

This is clearly not what we want, it breaks in the dynamic loader not at the very first instruction in the executable!

And break *0 doesn’t work with my gdb either… Sigh.

```
break *0
```

After playing with several other tools, now I find two nice solutions for this:

```
entry-break
```

```
entry-break               -- Tries to find best entry point and sets a temporary breakpoint on it. The command will test for                             well-known symbols for entry points, such as `main`, `_main`, `__libc_start_main`, etc. defined by                             the setting `entrypoint_symbols`. (alias: start)
```

It breaks at exact the first instruction:

```
gef> x/10i $rip=> 0x555555554990: xor    ebp,ebp   0x555555554992: mov    r9,rdx   0x555555554995: pop    rsi   0x555555554996: mov    rdx,rsp   0x555555554999: and    rsp,0xfffffffffffffff0   0x55555555499d: push   rax   0x55555555499e: push   rsp   0x55555555499f: lea    r8,[rip+0x57a]        # 0x555555554f20   0x5555555549a6: lea    rcx,[rip+0x503]        # 0x555555554eb0   0x5555555549ad: lea    rdi,[rip+0x3be]        # 0x555555554d72
```

2. Use radare2 debugger

We can run analysis before running it, and the addresses are known to us after analysis:

```
% r2 -d ./re_easy_to_say-4d171ed2949ad2e9fcb5350c71aa80ecProcess with PID 21364 started...= attach 21364 21364bin.baddr 0x55d4b3955000Using 0x55d4b3955000asm.bits 64 -- Everything up-to-date.[0x7f9e4664f1f0]> aaa[x] Analyze all flags starting with sym. and entry0 (aa)TODO: esil-vm not initialized[Cannot determine xref search boundariesr references (aar)[x] Analyze len bytes of instructions for references (aar)[x] Analyze function calls (aac)[x] Use -AA or aaaa to perform additional experimental analysis.[x] Constructing a function name for fcn.* and sym.func.* functions (aan)ptrace (PT_ATTACH): Operation not permitted= attach 21364 21364[0x7f9e4664f1f0]> sf main[0x55d4b3955d72]> pdf/ (fcn) main 310|   main ();|           ; var int local_2028h @ rbp-0x2028|           ; var int local_2020h @ rbp-0x2020|           ; var int local_2018h @ rbp-0x2018|           ; var int local_2010h @ rbp-0x2010|           ; var int local_8h @ rbp-0x8|              ; DATA XREF from 0x55d4b39559ad (entry0)|           0x55d4b3955d72      55             push rbp...[0x55d4b3955d72]> db 0x55d4b3955d72[0x55d4b3955d72]> dchit breakpoint at: 55d4b3955d72
```

Hope this helps people like me who ever got stuck with PIE binaries.

Happy hacking!