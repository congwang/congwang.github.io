---
layout: post
title: "Understanding struct __sk_buff"
date: 2021-08-15 02:28:49 +0000
categories: blog
---

### Understanding struct __sk_buff

I have been mentoring our interns for some eBPF projects. The most common question raised during their internship is about the eBPF struct __sk_buff. It seems there is no document on Internet explains why it is introduced and how it works. Let me explain this a bit.

### What is struct __sk_buff anyway?

You can consider struct __sk_buff as a simplified version of struct sk_buff, but only for eBPF programs. Unlike the over-complicated struct sk_buff, struct __sk_buff is really flat and simple. In your eBPF programs, you can just use struct __sk_buff like all other structs, there is no magic on the users’ side. More importantly, you can even write into __sk_buff too, which is effectively writing to kernel’s struct sk_buff. The magic part is in the eBPF verifier, we will see this later.

### Why do we need struct __sk_buff?

For eBPF programs, it is not as easy as kernel modules to read or write kernel data structures. Reading kernel memory requires, either explicitly or implicitly, an eBPF helper bpf_probe_read(). Writing kernel memory is considered as unsafe generally, so there is no one general way to do so, you will need some eBPF helpers too. For instance, to write into a network packet, you need to call bpf_skb_store_bytes().

So why don’t we just use bpf_probe_read() to read struct sk_buff and add more eBPF helpers to modify it? This is doable for sure, however, if we look into struct sk_buff more closely, there are at least 3 disadvantages with this approach:

Thus, struct __sk_buff offers a simplified and ABI-compatible view of struct sk_buff in kernel. It makes eBPF programmers’ life easier.

### How does it work?

If kernel needs to maintain both struct sk_buff and struct __sk_buff, how does it keep them compatible with each other? Why reading/writing __sk_buff is actually reading/writing sk_buff behind the scene?

Like I said, the magic is in the eBPF verifier. Don’t get fooled by its name, eBPF verifier nowadays does more than just verification. For struct __sk_buff, eBPF verifier converts it to struct sk_buff transparently when you load your eBPF program, this is why there is no difference for eBPF programmers.

eBPF programs only see struct __sk_buff in “user-space”, while kernel, particularly the eBPF verifier, sees both sk_buff and __sk_buff, so it has the knowledge of both. With this, it can translate from __sk_buff to sk_buff. The actual code is in bpf_convert_ctx_access(). For instance, __sk_buff::len is converted to sk_buff::len in this way:

What this code does is generating some eBPF bytecode and inserting it into the eBPF program being loaded. This bytecode fixes the offset of __sk_buff::len, converts it into the actual offset of sk_buff::len. Notice that the base address is same for both sk_buff and __sk_buff. And both are static information available after compilation.

It is more complicated to retrieve information from some inner struct, for example, skb_shinfo(), as it clearly needs more logic and more bytecode. You can take a look at bpf_convert_shinfo_access().

As we can see, it is a really smart choice to invent such a struct __sk_buff for eBPF programs. And it is always amazing to see what eBPF verifier could accomplish.