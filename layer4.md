# Layer 4: A53 Co-processor → Secure Enclave

---

## Prerequisite: Dari Layer 3

Layer 3 bagi kita full kernel code execution + XOM bypass. Tapi ada satu lagi chip yang independent — A53 co-processor. Walaupun main CPU dah fully compromised, A53 masih berdiri sendiri.

---

## A53 Tu Apa?

A53 = ARM Cortex-A53, **chip berasingan** dalam hardware. Bukan part of main x86-64 CPU.

```
Hardware:
┌─────────────────────┐     ┌──────────────────────┐
│   Main SoC (x86-64) │     │  A53 Co-processor    │
│                     │     │  (ARM)               │
│  Runs FreeBSD       │◄───►│  Runs own tiny OS    │
│  Runs games         │     │  RSA/ECDSA verify    │
│  Runs exploits      │     │  SSD encryption keys │
│                     │     │  = Root of Trust     │
└─────────────────────┘     └──────────────────────┘
```

Main CPU boleh kena exploit sepenuhnya — tapi A53 masih independent.

---

## Dua Tanggungjawab A53

### 1. PKG Signature Verification

Same concept as RSA — public/private key pair. Sony holds the private key, A53 holds the public key for verification. → [RSA Beginner's Guide](https://medium.com/@Espress0/rsa-encryption-beginners-guide-6caee3e8cd80)

```
Sony punya:
├── Private key (Sony simpan, secret)  ← untuk sign PKG
└── Public key  (embedded dalam A53)   ← untuk verify signature

Flow:
FPKG kita → A53 verify dengan Sony public key
          → signature invalid (kita tak ada Sony private key)
          → install rejected ✗
```

Sebab public key dalam A53 (isolated), kita tak boleh replace dia walaupun ada kernel R/W.

### 2. SSD Encryption Keys

```
Save data, game data → encrypted dengan per-console key
                     → key stored dalam A53
                     → SSD content = gibberish tanpa key
```

Key ni **per-console** — setiap unit ada key lain. Tak boleh guna key satu PS5 untuk decrypt PS5 lain.

---

## Macam Mana Main CPU Communicate Dengan A53?

Via **shared memory region**:

```
Main CPU                    A53
    │                        │
    ├──► shared memory ◄─────┤
    │    [ request ]         │
    │    [ response ]        │
```

Main CPU hantar request, A53 process, balas dalam shared memory.

---

## Attack Surface

Dengan full hypervisor access (Layer 3), kita boleh:

1. **Baca shared memory** — analyze format communication antara main CPU dan A53
2. **Intercept requests** — modify apa yang dihantar ke A53
3. **Dump A53 firmware** — copy A53 OS keluar untuk analyze dalam Ghidra

---

## Goal: Code Exec Dalam A53

Kalau jumpa bug dalam A53 OS dan dapat code exec, boleh access apa yang A53 control — encryption keys, verify functions, semua benda yang main CPU tak boleh reach.

---

## Status (Nov 2026)

A53 exploitation masih **work in progress**. Dump A53 firmware dah berjaya — sekarang tengah analyze untuk cari bug dalam A53 OS.

---

## Flow Penuh Layer 4

```
Layer 3: full kernel code execution + XOM bypass
        ↓
Dump A53 firmware via shared memory / hypervisor access
        ↓
Reverse engineer A53 OS (Ghidra)
        ↓
Cari bug dalam A53 OS
        ↓
Exploit bug → code exec dalam A53
        ↓
Access A53 controlled resources
        ↓
SSD keys accessible ✓
```

---

## Full Chain Summary

```
Layer 1: Browser bug → arbitrary R/W userland
        ↓
Layer 2: Kernel UAF → arbitrary R/W kernel
        ↓
Layer 3: Hypervisor race condition → XOM bypass
        ↓
Layer 4: A53 exploit → PKG sig bypass + SSD keys
```
