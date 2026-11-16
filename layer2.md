# Layer 2: Kernel UAF → Arbitrary R/W Kernel

---

## Prerequisite: Dari Layer 1

Layer 1 bagi kita arbitrary R/W dalam **userland memory**. Dari sini kita boleh call **syscalls** — pintu masuk ke kernel.

---

## Kernel Memory Layout

```
┌─────────────────────────────────────┐
│  .text   (--x)                      │  ← compiled kernel code
│  e.g. sys_read(), sys_write()       │     XOM: execute only, tak boleh baca/tulis
├─────────────────────────────────────┤
│  .data   (rw-)                      │  ← global variables (fixed size)
├─────────────────────────────────────┤
│  heap    (rw-)                      │  ← dynamic structs (kmalloc)
│  e.g. struct aio_op, file handles   │     tumbuh masa runtime
└─────────────────────────────────────┘
```

**XOM (Execute Only Memory):** Hypervisor enforce kernel `.text` pages sebagai execute-only. Tak boleh baca atau tulis. Ini halang exploit biasa yang overwrite kernel code.

---

## Masalah: Kenapa Tak Boleh Terus Overwrite Kernel Code?

```
Exploit biasa:
arbitrary_write → overwrite function ptr dalam .text → code exec ✓

Dengan XOM:
arbitrary_write → cuba tulis ke .text → hypervisor page fault → PANIC ✗
```

Kena cari cara lain yang tak sentuh `.text`.

---

## Penyelesaian: UAF Attack Pada Kernel Heap

Kernel heap = `rw-` = boleh tulis. Kalau ada struct dalam heap yang ada **function pointer**, kita overwrite pointer tu (bukan code), pastu trick kernel panggil pointer tu.

---

## UAF (Use After Free) — Konsep

```
Rule: malloc → guna → free → JANGAN guna lagi

UAF = guna pointer LEPAS dah free
```

```c
struct *ptr = kmalloc(64);  // allocate
ptr->func = kernel_fn;      // guna
kfree(ptr);                 // free

ptr->func();                // ← UAF! masih guna ptr
```

**Kenapa bahaya:** OS boleh allocate semula memory yang dah free tu untuk struct lain.

```
ptr freed → allocator recycle 0x5000 untuk struct baru
ptr masih point ke 0x5000
Kita tulis via ptr → overwrite struct baru!
```

---

## Bug: aio_multi_wait Race Condition

`aio_multi_wait` = syscall untuk tunggu async I/O operations selesai.

**Bug berlaku bila 2 thread jalan serentak:**

```
Thread A: aio_multi_wait(ops, 3)   ← tunggu ops selesai
Thread B: cancel/free op2           ← free salah satu op
```

**Timeline:**

```
Thread A: check op2 → masih valid ✓
Thread B: free(op2 @ 0x5000)        ← op2 dah freed!
Thread A: guna op2 lagi             ← UAF! ptr masih 0x5000
```

**Timing gap** wujud sebab kernel ada lock tapi ada window kecil antara baca pointer dan guna dia:

```
Thread A baca pointer op2 = 0x5000
Thread A lepas lock
                            ← Thread B masuk, kfree(0x5000)
Thread A guna pointer 0x5000 → struct dah free = UAF
```

---

## Exploit Flow

```
1. Trigger aio_multi_wait UAF
        ↓
2. Dalam gap tu, kmalloc struct kita (saiz sama) → dapat 0x5000
   struct kita ada function pointer yang kita control
        ↓
3. aio_multi_wait guna freed ptr → tapi sekarang point ke struct kita
        ↓
4. Kernel call function ptr → pergi ke address kita letak
        ↓
5. Execute code dalam kernel context ✓
```

---

## kstuff — Apa Dia?

`kstuff` = tool yang **guna** kernel arbitrary R/W untuk patch kernel memory.

```
aio_multi_wait UAF bug
        ↓
dapat kernel arbitrary R/W
        ↓
kstuff GUNA kuasa tu
untuk patch kernel .data / heap
        ↓
enable debug settings, bypass checks, etc.
```

**aio = bug yang bagi kuasa**
**kstuff = tool yang guna kuasa tu untuk patch**

kstuff patch kernel `.data` dan heap — bukan `.text` — sebab tu dia tak perlukan hypervisor bypass.

---

## Flow Penuh Layer 2

```
Layer 1: arbitrary R/W userland
        ↓
trigger aio_multi_wait syscall dari userland
        ↓
race condition dalam kernel
        ↓
UAF → overwrite function ptr dalam heap struct
        ↓
kernel execute our code
        ↓
arbitrary R/W kernel memory ✓
        ↓
kstuff patch kernel settings
        ↓
boleh proceed ke Layer 3 (Hypervisor)
```
