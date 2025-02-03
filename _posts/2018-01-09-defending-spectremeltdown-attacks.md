---
layout: post
title: "Defending Spectre/Meltdown attacks"
date: 2018-01-09 18:54:08 +0000
categories: blog
---

### Defending Spectre/Meltdown attacks

After explaining how these attacks work, now it is time to see how Linux kernel defends these attacks before Intel could patch its CPUs.

The Meltdown attack looks more scary than Spectre, because: a) it paves a way to cross the solid user-kernel boundary which we rely on fundamentally in our unified virtual memory space; b) at least we can still disable kernel JIT to mitigate Spectre attack quickly, while there is no mitigation for Meltdown essentially.

Some fundamental change is needed at OS level to defend Meltdown attack — Page Table Isolation (PTI). PTI fundamentally changes the x86_64 virtual memory layout, it is a big change that impacts a lot of x86_64 system call related things.

As we know, traditionally Linux use the following virtual memory layout on x86_64:

Kernel-space is mapped into each process and shares a same virtual memory space with user-space, which means there is only one set of 4-level page tables for one process. Given this, when kernel switches from one process to another, it has to switch the whole page tables too, TLB has to be flushed, this is why context switching is so expensive. On the other hand, when we just enter the kernel-space from user-space, e.g. when invoking a system call, these page tables remain the same, this is why invoking a system call is faster.

There is a solid boundary between user-space and kernel-space for obvious reason. When user-space tries to access a kernel-space address, a page fault is triggered and access is denied by kernel with a segment fault. On the other hand, kernel-space has the privilege to do everything, it has the permission to access user-space memory whenever it needs.

Now with Meltdown attack, user-space is able to leak kernel-space memory via a side-channel, since they share a same set of page tables. PTI addresses this by isolating the kernel-space from user-space by providing two different sets of page tables: the one for user-space only contains user-space memory (for sake of simplicity); the one for kernel-space still contains everything like before:

With PTI, Meltdown is completely closed since user-space does not even possibly have any kernel page table entries cached. But, now when we enter kernel-space from user-space via each system call, we have to switch to a new set of page tables, which slows down system calls. This is what people have been complaining about in LKML. To minimize the performance regression, a per-process control of PTI is proposed in LKML, in my opinion this is a good compromise, applications hurt by PTI could revert it for their own processes, while PTI is still enabled by default for those who don’t care.

Defending Spectre, on the other hand, is more complicated, there is probably no one true solution here. Essentially, we need to add a speculation barrier to stop the speculative execution in each code gadget like this:

```
if (x < array1_size) {    tmp = array1[x] * cache_line_size;    /* speculation barrier goes here */    y = array2[tmp];}
```

For Intel CPU, lfence is the right instruction to do so:

```
lfence
```

> After reviewing an initial draft of this paper, Intel engineers indicated that the definition of lfence will be revised to specify that it blocks speculative execution.

Two kinds mitigations are happening at this time:

```
lfence
```

```
lfence
```

Last but not least, even for me, a full-time Linux kernel developer, one thing still surprises me is that after 26 years of development Linux kernel community could still move forward so rapidly. It is also interesting to see how Linux kernel community works with other community (GCC) together to come up with a complete solution.