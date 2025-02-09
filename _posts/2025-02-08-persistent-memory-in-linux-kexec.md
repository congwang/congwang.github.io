---
layout: post
title: Persistent Memory in Linux Kexec
date: 2025-02-08 19:47 -0800
---

If you've been following the Linux kernel development community over the past few years, you might have noticed an interesting emerging trend in the field of memory persistence when using kexec to boot into a new kernel.

Between 2020 and 2025, five different teams of developers have tackled the same thorny problem: how to preserve memory contents when using kexec to boot into a new kernel. It's a fascinating case study in how the open source community approaches complex technical challenges, with each proposal bringing fresh insights to the table.

## Why Is This Problem So Important?

To understand why this problem has attracted so much attention, we need to first talk about kexec. There are two kexec implementations: kexec fast reboot and kdump. In this context, we are talking about kexec fast reboot, which is a feature that allows you to switch to a new kernel without shutting down the system. Think of kexec as a shortcut in the boot process - instead of going through the whole BIOS/firmware dance when you need to switch to a new kernel, kexec lets you jump directly there. This capability is incredibly useful for reducing downtime during kernel updates or implementing fast reboot mechanisms.

But there's a catch. By default, kexec treats the boot into the new kernel as a fresh start, wiping the slate clean of all previous memory state. While this clean-slate approach might seem sensible at first glance, it causes real problems in several important scenarios.

1. Virtualized Machine. When running virtual machines, their memory contains active processes, cached data, and vital state information. If this memory isn't preserved during a kernel transition, every VM would effectively crash during the host kernel update. This isn't just inconvenient - it could mean breaking service level agreements and disrupting critical services.

2. Virtualization using direct device assignment (also known as PCI passthrough). These setups allow virtual machines to directly control hardware devices for better performance. The IOMMU maintains critical memory mappings that enable devices to safely access memory. If these mappings are lost during kexec, any ongoing device operations could fail, potentially corrupting data or crashing the system.

3. Database systems and other applications that maintain large in-memory caches. Having to reload these caches after every kernel update introduces significant performance penalties and service disruptions.

What makes this problem particularly challenging is the fundamental tension between two competing needs: the need to preserve specific memory regions exactly as they are, and the need to give the new kernel enough flexibility to boot successfully. Add to this the complexity of modern hardware, the requirements of virtualization, and the need for driver support, and you have a problem that requires careful balance between competing constraints.

Let's dive into each of the proposals that have tried to solve this challenge, and see how their approaches have evolved over time.

## The Evolution of Solutions

### [PKRAM (2020): The Filesystem Pioneer](https://lwn.net/Articles/819778/ "PKRAM (2020): The Filesystem Pioneer")

Anthony Yznaga's PKRAM proposal was the first major attempt to tackle this problem, and it took an intriguingly practical approach. Rather than creating entirely new mechanisms, PKRAM leveraged something that Linux developers and users were already familiar with: tmpfs.

The core idea was elegant in its simplicity. PKRAM implemented a tmpfs-style filesystem that could mark certain memory regions for preservation across kexec. When you mounted a PKRAM filesystem, you could create and manipulate files just like you would with any other filesystem, but behind the scenes, PKRAM was carefully managing the memory to ensure it would survive the kexec process.

The genius of this approach lay in its user-friendly interface. System administrators didn't need to learn new concepts or APIs - they could just mount the filesystem and use standard file operations. Under the hood, PKRAM handled all the complexity of memory preservation, passing a root page pointer through the kernel command line to help the new kernel locate and restore the preserved memory.

Where PKRAM really shined was in its dynamic allocation capabilities. Rather than requiring administrators to decide upfront how much memory they might need to preserve, PKRAM could grow and shrink as needed. This flexibility made it particularly well-suited for general-purpose use cases where memory requirements might not be known in advance.

However, PKRAM wasn't without its limitations. Its filesystem-centric approach, while user-friendly, didn't always align well with kernel-level needs. It also lacked NUMA awareness, which could impact performance on large systems, and its general-purpose nature meant it wasn't necessarily optimized for specific use cases like VM memory preservation.

### [Memory Pools (2023): The Kernel-Centric Approach](https://lwn.net/Articles/945581/ "Memory Pools (2023): The Kernel-Centric Approach")

When Stanislav Kinsburskii introduced the Memory Pools proposal in 2023, it represented a significant shift in thinking. Instead of approaching the problem from a user space perspective, Memory Pools focused squarely on kernel-level requirements.

Built on top of the Continuous Memory Allocator (CMA), Memory Pools took a more structured approach to memory preservation. The proposal introduced the concept of persistent memory pools - dedicated regions of memory that could be preserved across kexec operations. These pools were particularly well-suited for maintaining kernel-specific states like DMA mappings and IOMMU configurations.

What set Memory Pools apart was its deep integration with kernel subsystems. By working directly with the CMA, it could ensure efficient memory management and reduce fragmentation. The proposal also introduced a clever way of passing metadata between kernels using the Flattened Device Tree, which provided a robust mechanism for maintaining continuity across the kexec boundary.

One of the most interesting aspects of Memory Pools was its focus on predictability. Unlike PKRAM's dynamic approach, Memory Pools required memory regions to be defined at boot time. While this might seem like a limitation, it actually provided important guarantees about memory availability and location that were crucial for certain kernel subsystems.

### [PRMEM (2023): The Flexible Synthesizer](https://lwn.net/Articles/948014/ "PRMEM (2023): The Flexible Synthesizer")

Later in 2023, Madhavan T. Venkataraman introduced PRMEM, which attempted to bridge the gap between kernel and user space needs. PRMEM took a more comprehensive approach, implementing what you might call a "kitchen sink" solution to the persistence problem.

The heart of PRMEM was its sophisticated memory management system. It could handle both fixed allocations, like Memory Pools, and dynamic growth, like PKRAM. One of its most innovative features was the concept of named persistent instances, which provided a clean way to organize and manage different types of persistent state.

PRMEM also introduced some clever technical innovations. It integrated with the generic memory allocator for efficient memory management, and its support for persistent XArrays made it particularly well-suited for handling complex data structures. The proposal even included provisions for expanding the persistent memory region on demand, up to a configurable maximum size.

What made PRMEM particularly interesting was its attention to real-world operational needs. It included features for metadata validation, support for NUMA topologies, and even considerations for memory error handling. The proposal demonstrated a deep understanding of the challenges faced in production environments.

### [Pkernfs (2024): The Clean Slate](https://lore.kernel.org/lkml/20240205120203.60312-1-jgowans@amazon.com/ "Pkernfs (2024): The Clean Slate")

James Gowans' Pkernfs proposal, introduced in 2024, took a bold step by suggesting a completely new filesystem specifically designed for kernel persistent state. Rather than trying to adapt existing mechanisms, Pkernfs proposed a clean-slate solution with a laser focus on the requirements of modern virtualization environments.

The core innovation of Pkernfs was its complete separation of persistent memory from the regular kernel memory management system. This separation provided important security benefits and made it easier to reason about the state of persistent memory. The proposal included specific support for removing guest memory from the direct map, which was particularly valuable for secure virtualization scenarios.

What set Pkernfs apart was its holistic approach to persistent state. Rather than just handling memory, it provided a unified interface for managing all types of persistent state, including IOMMU mappings and device configurations. This comprehensive approach made it particularly well-suited for complex virtualization environments where multiple types of state needed to be preserved.

### [KHO (2025): The Foundation Builder](https://lore.kernel.org/linux-mm/20250206132754.2596694-1-rppt@kernel.org/ "KHO (2025): The Foundation Builder")

The most recent proposal, KHO (Kexec HandOver) by Mike Rapoport, takes a fundamentally different approach. Instead of trying to solve the entire persistence problem, KHO focuses on providing a solid foundation that other solutions can build upon.

At its core, KHO is all about metadata management. It uses the Flattened Device Tree to exchange information between kernels, but does so in a way that's both flexible and extensible. The proposal includes sophisticated handling of scratch regions for the new kernel's bootstrap process, ensuring that preserved memory won't be accidentally overwritten during the boot process.

What makes KHO particularly interesting is its architecture-aware design. Rather than trying to provide a one-size-fits-all solution, it acknowledges that different architectures might need to handle persistence differently. This flexibility, combined with its clean integration points for kernel subsystems, makes it a promising foundation for future persistence solutions.

## Looking to the Future

As we look at these five proposals, we can see a clear evolution in thinking about the problem of memory persistence across kexec. Each proposal has brought valuable insights to the table, and the ultimate solution might well incorporate ideas from multiple approaches.

What's particularly fascinating is how each proposal reflects different priorities and constraints. PKRAM prioritized ease of use, Memory Pools focused on kernel integration, PRMEM sought comprehensiveness, Pkernfs emphasized clean separation, and KHO focused on providing a solid foundation.

In the end, the key takeaway is that while each proposal has its own unique approach, they all share a common goal: to provide a robust solution for preserving memory across kexec operations.The future solution will likely need to provide clean separation of persistent and non-persistent memory, ideally support both fixed and dynamic allocation strategies, handle metadata robustly, and integrate well with existing kernel drivers. Most importantly, it will need to support both kernel and user space use cases while maintaining the security and reliability that modern systems demand.