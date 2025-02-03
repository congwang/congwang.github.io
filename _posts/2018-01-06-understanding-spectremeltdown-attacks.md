---
layout: post
title: "Understanding Spectre/Meltdown attacks"
date: 2018-01-06 02:03:11 +0000
categories: blog
---

### Understanding Spectre/Meltdown attacks

(Disclaimer: I am not at all a security expert, nor a hardware engineer. Don’t trust everything I write here.)

> Behind each exploit there is a history of creativity and incredible knowledge.

Like many other security exploits, the Meltdown and Spectre attacks are creative too. They exploit some seemingly harmless CPU optimizations, creatively put them together, finally form an attack blows up literally all popular CPU’s. What an irony!

Basically, they exploit three common modern CPU optimizations:

So far so good, right? Well, the art of security exploit is hacking seemingly harmless things in unexpected ways and tweaking them into a weapon. This is the case too.

Think about the timing I mentioned above about CPU caches, that is, cached memory access can be distinguished from uncached memory access in a reliable way. This somehow leaks a bit of information (cached or uncached) to us, although this bit still looks very useless. Can we exploit this to leak a bit of memory? Yes! We need to combine it with out-of-order or speculative execution.

The following pseudo C code is pretty normal, we do a boundary check before reading an element with a given index of an integer array, and use one bit in this integer to selectively access two magic addresses:

```
int *foo_array;unsigned int array_size;int* addresses[2] = {....};
```

```
unsigned int controlled_index;
```

```
foo_array = malloc(sizeof(int) * array_size);
```

```
if (controlled_index < array_size) {    int bit = foo_array[controlled_index] & 0x1;    int bar = *(addresses[bit]);    //...}
```

The logic is solid too. But how does CPU execute this? With branch predication, CPU could pick the TRUE branch after some “training” and starts to speculatively execute the code ahead of instruction pointer which may be still before the if check. And this is actually a good guess, since this branch is a non-error case which we probably care more than the error case.

```
if
```

Again this is just a guess, if the index is actually out-of-bound, CPU actually reads out-of-bound data when speculatively execute these instructions. This is usually fine too, as long as CPU doesn’t trigger a page fault here. Until, in this case, it still caches either addresses[0] or addresses[1] when returns to the non-speculative branch!!!

```
addresses[0]
```

```
addresses[1]
```

We already know cached or uncached memory access makes a noticeable difference, it is simple to figure out which of them is cached, then we can infer a single bit from an out-of-bound memory!!! If we iterate all 8 bits then we can infer a single byte from an out-of-bound memory. If we could even carefully control the index, we can literally read a specially-targeted memory location of a victim process… Bingo!!!

However, even with this we still can not cross the user-kernel boundary here, speculative execution can not bypass the page table permission check. According to Meltdown paper, out-of-order execution can bypass this:

> … Meltdown exploits the out-of-order execution of modern CPUs, which still executes instructions in the small time window between the illegal memory access and the raising of the exception

And the following pseudo C code shows how it basically works:

```
char probe_array[256 * 4096];char *kernel_addr = 0xffff8......;char foo;
```

```
ffff8......
```

```
pid_t pid = fork();if (pid == 0) {    char byte = *kernel_addr; // trigger a page fault    int index = byte << 12; // out-of-order, executed before fault    foo = probe_array[index]; // out-of-order, executed before fault    // Finally die} else if (pid > 0) {    waitpid();    for (int i = 0; i < 256; i++) {        if (is_cached(&probe_array[i * 4096]))            break;    }    // Now, i is the byte leaked from kernel memory}
```

Of course, it is impossible to write C code like this to exploit it, we can’t rely on compilers to pick instructions for us, we have to very carefully pick right x86 instructions manually in order to “help” the CPU to really reorder these instructions in a way we expect. Needless to say, this requires expert-level understanding of Intel CPU details.

To make Spectre attack happen, we still either need to find such a piece of code to exploit or inject our own code for kernel to execute. It turns out it is very hard to find existing code to re-use in kernel, so we need to write our own eBPF code and inject into kernel JIT engine to read all physical memory without needing any privilege! This is basically how Spectre and Meltdown attacks could read a target memory contains credentials or the whole physical memory without privilege.

Obviously, this is very much simplified, there are a lot of details about Intel CPU architecture, instructions and PoC I omit here. You have to read the two papers for the details. But we can see the creativity behind these attacks, and we can also see how security researchers exploit this area one step forward each time and eventually come up with one blows up every CPU.