# Layer 3: Hypervisor → Bypass XOM

---

## Prerequisite: Dari Layer 2

Layer 2 bagi kita arbitrary R/W dalam **kernel memory**. Tapi kernel `.text` masih protected oleh hypervisor (XOM). Kena bypass hypervisor dulu sebelum boleh patch kernel code.

---

## Hypervisor Buat Apa?

Hypervisor duduk bawah kernel. Dia control **page permissions** via NPT (Nested Page Tables).

```
Kernel nak execute code → request ke CPU
CPU check NPT (hypervisor punya table)
NPT kata .text = --x (execute only)
→ kernel .text = XOM ✓
```

Kalau kernel cuba tulis ke .text:
```
Kernel: "aku nak tulis ke 0xFFFF_text"
NPT: "tak boleh, --x je"
CPU: page fault → PANIC
```

Hypervisor = security guard yang kernel sendiri tak boleh bypass.

---

## Normal — PS5 Waktu Rest Mode

Hypervisor jaga semua cores:

```
Core 0:  [jalan]  NPT: secure ✓
Core 1:  [tidur]  NPT: secure ✓
Core 2:  [tidur]  NPT: secure ✓
...
Core 15: [tidur]  NPT: secure ✓
```

NPT = rules "siapa boleh buat apa dengan memory".

---

## PS5 Keluar Rest Mode

Core 0 kena "bangunkan" cores lain. Dia buat via `startup_mp`:

```
Core 0: "eh core 1, bangun!"
        → setup NPT untuk core 1
        "eh core 2, bangun!"
        → setup NPT untuk core 2
        ...
```

Masa setup tu, NPT core 1-15 = **TEMPORARY / INSECURE**.

```
Core 0:  [jalan]  NPT: secure ✓
Core 1:  [bangun] NPT: INSECURE  ← dalam proses setup
Core 2:  [bangun] NPT: INSECURE  ← dalam proses setup
...
```

---

## Window Tu Apa?

Window = masa antara core bangun tapi NPT belum secure.

```
Timeline Core 1:
─────────────────────────────────────────────────────
[tidur] → [bangun] → [NPT insecure] → [NPT secure]
                            ↑
                      WINDOW DI SINI
                      (beberapa millisecond)
```

---

## Race Condition

Exploit dah prepare code dekat core 1 **sebelum** PS5 masuk Rest Mode.

```
Core 1 bangun → NPT insecure → terus run exploit code
                                        ↓
                             Baca/tulis hypervisor memory
                                        ↓
                             Tukar NPT rules sendiri
                                        ↓
                             kernel .text → rwx ✓
```

"Race" sebab kena buat semua tu sebelum Core 0 habis setup NPT jadi secure balik.

---

## Visualize Penuh

```
PS5 masuk Rest Mode
        ↓
Exploit letak code kat core 1 (sebelum tidur)
        ↓
PS5 keluar Rest Mode
        ↓
Core 0 bangunkan core 1 → NPT insecure
        ↓
Core 1 terus run exploit code dalam window tu
        ↓
Baca/tulis hypervisor memory → tukar NPT
        ↓
XOM bypass ✓
```

---

## Apa Yang Dapat Lepas Bypass

```
Sebelum Byepervisor:
kstuff → patch kernel .data / heap sahaja
→ debug settings, bypass checks ✓
→ kernel code execution terhad

Lepas Byepervisor:
boleh patch kernel .text
→ modify syscall handlers
→ inject custom kernel code
→ full kernel code execution ✓
```

---

## Flow Penuh Layer 3

```
Layer 2: arbitrary R/W kernel memory
        ↓
PS5 masuk Rest Mode → exploit letak code kat core 1
        ↓
PS5 keluar Rest Mode → Core 0 bangun
        ↓
Core 0 setup NPT → insecure window
        ↓
Core 1 run exploit code dalam window
        ↓
Tukar NPT → kernel .text jadi rwx
        ↓
XOM bypass ✓ → full kernel code execution
        ↓
Boleh proceed ke Layer 4 (A53)
```
