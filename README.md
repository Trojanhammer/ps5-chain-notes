# Exploit Chain — Learning Notes
> Browser → Kernel → Hypervisor → Secure Enclave
> Personal notes for understanding modern exploit chains from userland to hardware root of trust

---

## Purpose

Personal learning notes to understand how modern exploit chains work — from a bug in a browser engine all the way down to the kernel, hypervisor, and hardware security enclave. This repository has no affiliation with any hardware manufacturer or platform vendor. All content is strictly for educational purposes.

Concepts covered:
- Memory layout and pointer arithmetic
- Type confusion bugs in JIT compilers
- Out-of-bounds memory access
- Use After Free (UAF)
- Race conditions in the kernel
- Hypervisor and virtualisation security
- Hardware security enclaves

---

## Security Architecture (Layered Model)

```
┌─────────────────────────────────────┐
│          Userland                   │  ← apps, browser, processes
├─────────────────────────────────────┤
│          Kernel                     │  ← syscalls, process mgmt
├─────────────────────────────────────┤
│          Hypervisor                 │  ← memory protection, XOM
├─────────────────────────────────────┤
│          Secure Enclave (ARM)       │  ← crypto root of trust
└─────────────────────────────────────┘
```

Each layer controls the layer above it. An exploit chain must break through from top to bottom.

---

## Files

| File | Contents |
|------|----------|
| [layer1.md](layer1.md) | Browser engine bug → arbitrary R/W userland |
| [layer2.md](layer2.md) | Kernel UAF → arbitrary R/W kernel |
| [layer3.md](layer3.md) | Hypervisor race condition → XOM bypass |
| [layer4.md](layer4.md) | A53 co-processor → secure enclave access |

---

## Resources

- [pwn.college](https://pwn.college) — learn exploitation from basics
- [LiveOverflow YouTube](https://www.youtube.com/@LiveOverflow) — binary exploitation
- [ARM TrustZone Intro — Quarkslab](https://blog.quarkslab.com/introduction-to-trusted-execution-environment-arms-trustzone.html)
