---
layout: post
title: Two-Phase eBPF Program Signing
date: 2025-02-01 18:31 -0800
---

The Extended Berkeley Packet Filter (eBPF) has revolutionized how we extend and observe the Linux kernel. However, with great power comes great responsibility, and securing eBPF programs has been a persistent challenge in the Linux kernel community. Today, I want to share an innovative approach to eBPF program signing that addresses some fundamental challenges in this space.

## The Evolution of eBPF Security

Before diving into the two-phase signing solution, let's look at how eBPF security has evolved. The Linux kernel has always been cautious about eBPF security, implementing various safeguards:

1. The eBPF verifier, which performs static analysis to ensure programs can't harm the kernel
2. Privilege restrictions requiring `CAP_BPF` capability for loading programs
   * BPF tokens were introduced to provide more fine-grained control of capabilities
3. Various LSM (Linux Security Module) hooks for additional security controls

Previous attempts at eBPF program signing in the kernel community focused on traditional code signing approaches. However, these attempts faced a fundamental challenge: eBPF programs undergo necessary modifications during the loading process, invalidating traditional signatures.

## The Challenge: Why Traditional Signing Doesn't Work

The core issue lies in how eBPF programs are prepared for execution. When we compile an eBPF program, the resulting binary isn't in its final form. The loader, that is, the libbpf library needs to modify this binary before it can run in the kernel. These modifications include:

- Updating map file descriptors: When an eBPF program is compiled, maps are referenced using placeholder file descriptors. During loading, libbpf creates the actual maps and updates these references with real file descriptors to ensure correct map access at runtime.
- Patching relocations: The program contains references to functions, maps, and other resources that need to be resolved to their actual memory locations. libbpf updates these references with the correct addresses where the resources will be located in memory, similar to dynamic linking in regular programs.
- Making other runtime adjustments: This includes program size and offset calculations, updating program type-specific parameters, adjusting instructions for kernel version compatibility, and setting up program attachments (e.g., to network interfaces or kernel hooks).

This creates a catch-22 situation:
- If you sign the original binary, the signature becomes invalid after these necessary libbpf's modifications
- If you sign after modifications, you lose the ability to verify the program's original authenticity

## In-kernel eBPF Program Loader

There were also attempts to move the eBPF program loader into the kernel. These proposals aimed to perform all program preparations (relocations, map creation, etc.) entirely in kernel space to maintain a single trust boundary. However, they faced several critical issues:

1. **Complex Privileged Code**: Moving the loader into the kernel would add a significant amount of complex code to the privileged kernel space. This increases the attack surface and the potential for exploitable vulnerabilities.

2. **Compatibility Issues**: While BTF provides kernel version independence for CO-RE-enabled programs, an in-kernel loader would still face compatibility challenges:
   - Legacy programs without BTF support require kernel-specific adjustments
   - Different kernel versions may require different approaches to map creation and program attachment
   - Programs using newer features need fallback paths for older kernels
   - Kernel ABI stability would become more critical as loader logic moves into the kernel

3. **Flexibility Issues**: User-space loading provides significantly more flexibility than a kernel-space approach would allow:
   - New program types and features can be supported without kernel updates
   - Programs can be preprocessed or transformed based on runtime conditions
   - Debug information and symbols can be handled more freely

4. **Verification Complexity**: The eBPF verifier would need to be substantially modified to verify the loader's operations, making an already complex component even more complicated and potentially introducing new verification bypasses.

The two-phase signing approach leverages existing eBPF infrastructure without requiring kernel modifications, offering several key advantages:
- Uses proven LSM hooks for program verification
- Maintains compatibility with existing eBPF tooling and workflows
- Reduces security risks by avoiding kernel-space complexity
- Allows for rapid deployment and adoption in production environments

## Introducing Two-Phase eBPF Program Signing

To solve this dilemma, I've developed a two-phase signing approach that mirrors the eBPF program preparation and loading process. Think of it like a legal document that requires both initial notarization and subsequent verification of modifications.

### Phase 1: The Baseline Signature

The first phase occurs when the eBPF program is initially compiled:
- A PKCS#7 signature is generated for the original, unmodified program
- This signature serves as proof that the original program came from a trusted source
- It's analogous to getting a document notarized before filling in the details

### Phase 2: The Modified Program Signature

The second phase happens after libbpf has made its necessary modifications:
- A new signature is created covering both the modified program and its original signature
- This establishes a chain of trust
- It proves that the modifications were authorized and applied to legitimate code

### The Verification Process

The kernel verifies these signatures in sequence during program loading:

1. First, it verifies the original program against its baseline signature
2. Then, it verifies the secondary signature covering both the modified program and the original signature

This two-step verification ensures:
- The program originated from a trusted source
- Any modifications were authorized
- The chain of trust remains unbroken

## Benefits of This Approach

The two-phase signing system offers several advantages:

1. **No Kernel Modifications Required**
   - Built entirely on top of existing eBPF infrastructure
     * Uses standard BPF LSM bpf() syscall hook for verification
     * Uses bpf_lookup_user_key() kfunc to retrieve keys from keyrings
     * Uses bpf_verify_pkcs7_signature() kfunc to verify signatures
   - Maintains compatibility with existing eBPF tooling and workflows
   - Modification of libbpf is transparent and relatively non-invasive

2. **Strong Auditability**
   - Failures can be precisely traced to either the original program or post-compilation modifications
   - Creates clear audit trails for security investigations

3. **Practical Security**
   - Accommodates necessary program modifications while maintaining security
   - Prevents signature stripping attacks
   - Creates a verifiable link between original and modified code

## Implementation Details

I've implemented this system using:
- PKCS#7 signatures for both phases
- BPF LSM hooks to intercept program loading
- Standard cryptographic primitives from OpenSSL
- Leverage the existing Linux kernel keyring infrastructure

The implementation is available as open source, though it's currently a proof of concept and not yet suitable for production use.

## Looking Forward

This two-phase signing approach opens new possibilities for securing eBPF programs while maintaining their flexibility. Future work could include:

- Integration with hardware security modules (HSM) for private key storage
- Enhanced private key management systems
- libbpf updates for the second phase signing
- Production hardening of the implementation

The eBPF ecosystem continues to evolve, and security measures must evolve with it. This two-phase signing approach represents a step forward in balancing security with the practical needs of eBPF program loading and execution.

For those interested in trying this out or contributing to the project, you can find the implementation at [github.com/congwang/ebpf-2-phase-signing](https://github.com/congwang/ebpf-2-phase-signing).
