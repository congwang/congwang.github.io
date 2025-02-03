---
layout: post
title: "CSAW CTF writeup — A Tour of x86"
date: 2018-09-18 04:20:33 +0000
categories: blog
---

### CSAW CTF writeup — A Tour of x86

The series of x86 assembly challenges in CSAW CTF are interesting, because it wraps with a very tiny i386 OS! How I miss the good old days of hand writing i386 boot assembly!

Part 1 is fairly elementary, I don’t want to waste a word on it. Part 2 is very easy too, the only thing I want to mention is that I used the following Qemu debugging trick to single step to where the OS stops and it clearly shows it pauses at the hlt instruction. You should know how to proceed to getting the flag!

```
hlt
```

```
qemu-system-x86_64 -serial stdio -d guest_errors -drive format=raw,file=tacOS.bin -s -S -d in_asm -singlestep
```

Now, let’s talk about part 3.

This part is slightly harder because it requires to write actual assembly code. Don’t worry, although it is embedded to the OS bootstrap code, you nearly don’t need to know anything about i386 interrupts or I/O ports. More importantly, at the time of the execution of our code, x86_64 is already properly setup, we can just write normal x86_64 assembly!

The only thing confusing is about the 0x1f character, it is a “protocol” for displaying characters via VGA, it means displaying a white-color character on a blue background. 0x00b8000 is the starting memory address of VGA video memory. All the rest are trivial, here is my NASM code:

```
bits 64
```

```
call get_ipget_ip:    pop rsi
```

```
add rsi, 0x23    mov rdx, 0x00b8000
```

```
print_char_loop:    cmp byte [rsi], 0    je done
```

```
mov byte bl, [rsi]    mov byte [rdx], bl    inc rdx    inc rsi    mov byte [rdx], 0x1f    inc rdx    jmp print_char_loop
```

```
done:    hlt    hlt
```

Compile it:

```
nasm -f bin -o printflag.bin printflag.asm
```

Translate it into hex:

```
xxd -p < printflag.bin | tr -d '\n'
```

Finally, feed them to the remote:

```
#!/usr/bin/env python
```

```
import subprocessfrom pwn import *
```

```
context.log_level = 'debug'
```

```
r = remote("rev.chal.csaw.io", 9004)r.recvline()r.sendline("e8000000005e4883c623ba00800b00803e0074128a1e881a48ffc248ffc6c6021f48ffc2ebe9f4f4")ret = r.recvline().rstrip('\n')port = ret.split(" ")[-1]subprocess.call(["vncviewer", "rev.chal.csaw.io:"+port])r.close()
```

In fact, I am sure we can put everything into a single pwntools Python script, I just didn’t want to spend much time on digging pwntools doc.