---
layout: post
title: "The design of lock_sock() in Linux kernel"
date: 2018-08-25 00:31:21 +0000
categories: blog
---

### The design of lock_sock() in Linux kernel

Among various kinds of locks in Linux kernel code base, `lock_sock()` is probably the weirdest one (if RCU is not even weirder).

As we all know, basically, there are two categories of locks in Linux kernel: blocking ones like a mutex or a semaphore; non-blocking ones like a spinlock, or a read-write lock. The pick of them largely depends on within which context you plan to use them. The weird part of this sock lock is actually it’s both blocking and non-blocking, depending on its context.

There are two contexts for the software part of the networking stack: Bottom-Half context, which is when a networking packet is received and transmitted, that is often called “data path” or the fast path; process context, which is where the “control path” happens, this is a slow path. Of course I simplify a lot here, for example, on the transmission side, we send packets in process context too until hitting the Qdisc layer or the driver layer.

For a socket, its “data path” is how packets destined to it are queued, this part is not directly influenced by user-space; its “control path” is how we configure a socket, like setting it via `setsockopt()`, and how we change the status of a socket, like via `bind()` and `close()`, which is completely and directly driven by user-space.

Generally speaking, the locking rule is clear: if we want to lock a shared data structure used in both contexts, we want to lock it in both contexts. This is why you see there are many `X_lock_bh()` variants of a given `X_lock()`. So for a socket, locking it in both contexts means a packet being queued in BH context won’t race with a user-space `close()` of a same socket.


Why `lock_sock()` is not just a regular spinlock at all? For performance!!!

If `lock_sock()` were a regular spinlock, then, when we lock it in user-space for `setsockopt()`, the packet receiving path in BH context had to busy-wait until `setsockopt()` finishes. This is very bad as packet receiving is the fast path we certainly don’t want to slow down.

This is why the sock lock is turned into two different locks for process context and BH context:

When process context begins to content with BH context, it becomes complicated:

Without additional logic, it is clearly not safe. To make it safe, `lock_sock()` enforces the following logic to callers:


Take a look at TCP receive path in BH context as an example:

```c
    bh_lock_sock_nested(sk);
    tcp_segs_in(tcp_sk(sk), skb);
    ret = 0;
    if (!sock_owned_by_user(sk)) {
        ret = tcp_v4_do_rcv(sk, skb);
    } else if (tcp_add_backlog(sk, skb)) {
        goto discard_and_relse;
    }
    bh_unlock_sock(sk);
```

See the difference between when the lock is owned by user-space and when it is not? Clearly, `tcp_v4_do_rcv()` is much more complicated than `tcp_add_backlog()`, what about the “missing” part when we just call `tcp_add_backlog()`? It is exactly what is moved into `release_sock()` after we release this lock:

```c
void release_sock(struct sock *sk){
    spin_lock_bh(&sk->sk_lock.slock);
    if (sk->sk_backlog.tail)
        __release_sock(sk);
//...
```

where `__release_sock()` will execute the callback `sk->sk_backlog_rcv()` to continue to process the packets queued in its backlog, and for TCP, this callback is exactly `tcp_v4_do_rcv()`. Bingo!

As you can see, the whole packet receiving process is not always finished in BH context. For TCP, `tcp_v4_do_rcv()` could be either called in BH context as usual, or in process context if locking contention happens on the sock lock.

But the rule is still simple: always call `lock_sock()` and `release_sock()` in process context, and always call `bh_lock_sock()` and `bh_unlock_sock()` in BH context, properly check `sock_owned_by_user()` after acquiring `bh_lock_sock()`.

Hope this clarifies your confusions about this weird lock when you look into it.