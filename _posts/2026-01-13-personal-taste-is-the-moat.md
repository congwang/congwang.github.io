---
layout: post
title: Personal Taste Is the Moat
date: 2026-01-13 12:00 -0800
---

AI can now tell you whether code works. It reviews patches, spots bugs, suggests fixes, and explains trade-offs. Correctness is becoming cheap. Competence is being commoditized.

But there's something AI cannot do: tell you whether something *should exist*.

That requires **taste**: judgment formed by long exposure to the best work humans have done, and by living with the consequences of decisions over time. In the AI era, personal taste is the moat.

## What Taste Actually Means

When people hear "taste," they think of preference, something arbitrary and subjective. That's wrong.

Taste is not about what you like. It's about what you've *seen*. It comes from studying great systems, watching bad ideas fail, understanding where complexity accumulates, knowing which shortcuts age badly, and internalizing what users actually experience. Taste is judgment compressed by time.

That's precisely why it's hard to automate, and impossible to shortcut.

## Proof: Rejecting an AI-Approved Kernel Patch

Recently, I [rejected a Linux kernel patchset](https://lore.kernel.org/netdev/CAM_iQpXXiOj=+jbZbmcth06-46LoU_XQd5-NuusaRdJn-80_HQ@mail.gmail.com/){:target="_blank"} that had already passed AI-based reviews.

The patch wasn't broken. It solved a real problem. It was technically coherent. I NACKed it anyway.

Why? The solution was wrong in ways that only matter long-term. The patch introduced a new `skb->ttl` mechanism to prevent packet loops. My objections weren't about correctness; they were about design:

1. **Bloat**: It increased `sk_buff` size even under minimal configuration.
2. **Wrong layer**: It fixed the symptom (infinite loops) instead of the root cause (enqueuing to the root qdisc).
3. **Hidden constraints**: It made `netem` behavior less predictable by introducing a kernel-internal limit invisible to userspace.

No AI reviewer would reject this patch. These aren't bugs; they're judgment calls. And judgment calls are exactly where taste lives.

## The Limits of AI

AI excels at pattern matching, local correctness, consistency with existing code, and applying known best practices. These are valuable. They're also becoming table stakes.

But systems like the Linux kernel aren't shaped by correctness alone. They're shaped by the collective taste of hundreds of maintainers, accumulated over decades, enforced through code review, and passed down through mentorship. The kernel's design reflects countless judgment calls about what belongs and what doesn't. These are properties you only *feel* after years of exposure.

Here's the key distinction: AI evaluates whether a change fits the rules. Taste decides whether the rules themselves are being bent in the wrong direction.

## Why Taste Is Now the Differentiator

Before AI, engineers differentiated themselves through speed and raw execution. AI has leveled that playing field. Everyone now has AI reviews, automated tests, copilots, and fast iteration loops.

When correctness becomes commoditized, the differentiator moves up the stack:

- Who decides what belongs in the system?
- Who recognizes a bad direction before it's too late?
- Who can say "this works, but it shouldn't exist"?

That judgment, that taste, is the moat.

## The Paradox: Better AI Makes Humans More Important

As AI removes mechanical friction, it surfaces harder decisions: Should this abstraction exist at all? Is this trade-off worth locking in? Are we leaking complexity to users? Will this design age gracefully?

These questions can't be answered with more data. They require judgment shaped by years of exposure.

As [Steve Jobs put it](https://www.youtube.com/watch?v=5y03eFMmOKY){:target="_blank"}:

> Ultimately, it comes down to taste — exposing yourself to the best things humans have done, and bringing those forward into what you're doing.

That's not romanticism. It's engineering wisdom.

## AI Assists. Taste Decides.

AI should be part of every process. Let it catch mistakes, suggest alternatives, and reduce toil. But passing AI review should never be the acceptance bar.

In domains that endure (kernels, runtimes, protocols, platforms), the final filter must be human judgment, informed by taste. Not because humans are always right, but because the hardest decisions aren't reducible to rules.

---

AI can tell you whether something works.

Only taste can tell you whether it belongs.

In the AI era, when execution is cheap and correctness is abundant, **personal taste is the moat.**
