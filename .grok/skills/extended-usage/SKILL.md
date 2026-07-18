---
name: extended-usage
description: >
  Usage hints and coding preferences for Intel 8085 extended (undocumented)
  instructions, using Zilog mnemonics. Strong rule: stack-only locals and
  intermediates; static/BSS only for state that must survive across function
  calls — never for intermediate storage. Prefer when choosing idioms for stack
  frames, reentrancy, 16/32-bit shifts, K-flag loops, sub hl,bc compares, or
  optimizing 8085 assembly beyond the bare opcode map. Complements
  /opcode-reference. Use when the user runs /extended-usage or asks how to use
  8085 extended ops effectively.
---

# 8085 Extended Instruction Usage

When and how to use the ten 8085 extended instructions. **Encodings, timings, and flags** are in **[opcode-reference](../opcode-reference/SKILL.md)** and `references/opcodes.md`.

Background: [8085 Software — Extended Instructions](https://feilipu.me/2021/09/27/8085-software/) (feilipu, 2021). Present on every 8085 (also Tundra CA80C85B).

**Always emit Zilog mnemonics** (project convention).

## Hard rule: stack variables, not static/BSS

**Static memory (BSS / `label: ds n` / fixed absolute cells) must only hold state that must survive across function calls.** Never use it for intermediate variable storage.

| Allowed in static/BSS | Forbidden in static/BSS |
|-----------------------|-------------------------|
| Values that **must outlive** the current call (true globals, module state, buffers callers re-enter later) | Locals, temps, intermediate results, scratch across a few instructions |
| MMIO, interrupt vectors, ROM constants | “Scratch” cells to avoid a push or stack frame |
| | Anything justified only by fewer cycles or easier coding |

- Function **locals, temporaries, and intermediate results** live **only on the stack** (arguments, return slots, pushes, explicit frames).
- Access with **`ld de,sp+*`**, **`ld hl,(de)`**, **`ld (de),hl`**, **`ld a,(de)`**, push/pop, **`ex (sp),hl`**.
- Prefer **pointers passed on the stack** over new static cells, even for long-lived data when the caller already owns the buffer.
- “Slightly fewer cycles” or “easier to write” is **not** enough justification for static/BSS scratch.

## Instruction preferences

| Zilog | Prefer for | Avoid / watch |
|-------|------------|----------------|
| `ld de,sp+*` | SP-relative address of a byte/word on the stack | `*` is **unsigned** 8-bit |
| `ld de,hl+*` | DE ← HL + unsigned offset (struct/buffer) | Same unsigned rule |
| `ld hl,(de)` / `ld (de),hl` | 16-bit load/store through DE | Not Z80 prefix encodings |
| `sub hl,bc` | 16-bit subtract; **== / !=**; signed compares with K | **No borrow-in**; not multi-word subtract chains |
| `sra hl` | Signed 16-bit arithmetic right shift | Clears V; C ← old bit 0 |
| `rl de` | Rotate DE left through C; ×2 on DE; 32-bit with HL | Pair with `add hl,hl` / `ex de,hl` as needed |
| `jp k,**` / `jp nk,**` | After 16-bit `dec`; signed compare outcomes | K after `dec rp` sets on **−1**, not on **0** |
| `rst v` | Branch to handler if V set | Vector **0040h** must exist |

## Core formulations

### 1. Stack access (primary working storage)

```asm
    ld  de,sp+n        ; n = unsigned offset (0…255)
    ld  hl,(de)        ; word load
    ld  a,(de)         ; byte load
    ld  (de),hl        ; word store
```

Often best: leave the pointer in **DE** and use `(de)` / `ld hl,(de)` without swapping. Second 32-bit value: keep it **on the stack**, not a shadow register bank.

### 2. HL ← SP+n — prefer extended over `ld hl,nn` / `add hl,sp`

Classic (any 16-bit offset):

```asm
    ld  hl,nn          ; 10c, 3B
    add hl,sp          ; 10c, 1B  → total 20c / 4B; sets V,C; DE preserved
```

**Unsigned 8-bit offset** — use `ld de,sp+n` (10c, 2B, no flags) plus `ex de,hl` (4c, 1B, no flags):

| Goal | Sequence | Bytes | Cycles | Flags | DE | Notes |
|------|----------|------:|-------:|-------|-----|-------|
| Pointer in DE, HL untouched | `ld de,sp+n` | 2 | **10** | none | = SP+n | Prefer when HL must stay |
| HL = SP+n, DE free | `ld de,sp+n` / `ex de,hl` | 3 | **14** | none | becomes old HL | **6c faster**, 1B smaller than classic |
| HL = SP+n, **preserve DE** | `ex de,hl` / `ld de,sp+n` / `ex de,hl` | 4 | **18** | none | restored | **2c faster** than classic; same size; no flag damage |

**DE-preserving form** (only final HL is the new value; DE restored; flags untouched):

```asm
    ex  de,hl          ; 4c   DE↔HL
    ld  de,sp+n        ; 10c  DE = SP+n  (old HL in DE is overwritten — OK)
    ex  de,hl          ; 4c   HL = SP+n, DE = original DE
```

Trace: start DE=D₀, HL=H₀ → after 1st `ex`: DE=H₀, HL=D₀ → after `ld de,sp+n`: DE=SP+n, HL=D₀ → after 2nd `ex`: **DE=D₀**, **HL=SP+n**.

Temporary use then restore previous HL (DE ends as SP+n, not original DE):

```asm
    ld  de,sp+n
    ex  de,hl          ; HL = SP+n, DE = old HL
    ; ... use HL ...
    ex  de,hl          ; HL restored; DE = SP+n
```

**Still prefer classic** when offset is not an unsigned 0…255, or when you need the **C/V** from `add hl,sp`.

### 3. Stack frame

```asm
    ; HL = SP+n (pick a sequence from §2)
    ; adjust HL as needed, then:
    ld  sp,hl
```

Document every slot. Drop consumed args in one epilogue:

```asm
    pop bc             ; return address — never pop af for this
    ; pop/discard arg words as required (pop af is OK here to discard only)
    push bc            ; return
```

**`pop af` and the return address:** F bit 3 is hardwired 0 on the 8085, so a word popped into AF can never be a faithful 16-bit value (`$FFFF` → `$FF7F`). **Never `pop af` the return address** (and never `push af` / `ret` a return path that depends on an intact address). **Do** use `pop af` to **discard** intermediate stack words on return when A/F need not be preserved — the corrupted F is irrelevant because the value is thrown away.

#### Multi-word frame rebuild (no `exx`)

Without alternate registers, a second long value lives **on the stack**, not in a shadow bank. When assembling a clean frame on top of junk:

1. **Push order vs layout.** Stack grows down. For layout top→bottom `W0, W1, W2` (W0 at lowest address / first pop), push **W2, then W1, then W0**. After `pop bc; pop de; pop hl` of pushed temps, restore with `push bc; push de; push hl` only if that matches the desired top word — verify with a depth diagram.
2. **Overlapping copy (memmove).** Copying a block upward when `dest = src + k` and `k < size` **overlaps**. Copy **high → low** (last byte first). Forward copy corrupts the tail.
3. **Raise SP over junk.** After a correct prefix of *N* good bytes sits above *J* junk bytes: copy the *N*-byte frame up by *J* (non-overlapping if *J ≥ N*, else high→low), then `ld hl,J` / `add hl,sp` / `ld sp,hl`.
4. **Product / result in BC·DE·HL while scrubbing.** Hold the full result in registers; do not park the return address in AF. Typical pattern: write result over a callee-owned slot, drop temps with SP math, then:

```asm
    pop hl             ; ret
    pop bc             ; result.bc
    pop de             ; result.ml
    ex  (sp),hl        ; HL = result.mh; (sp) = ret
    ex  de,hl          ; DE = mh, HL = ml
    ret                ; BC DEHL = result; only ret on stack
```

### 4. 16-bit compare and subtract — `sub hl,bc`

```asm
    ld  bc,de          ; if second operand is in DE
    sub hl,bc          ; HL − BC
    jp  z,equal        ; or jp nz,not_equal
```

Signed order (illustrative — tune K/Z/C to the relation):

```asm
    ld  bc,de
    sub hl,bc
    jp  k,...          ; use K together with Z/S/C as required
```

For multi-word subtract **with borrow**, use **`sub` / `sbc` through A**, not `sub hl,bc`.

### 5. Counted loops — K and pre-decrement

16-bit **`dec bc` / `dec de` / `dec hl` / `dec sp`** update **K** (not Z as the loop signal).

- K sets when the pair underflows to **−1**, **not** when it hits **0**
- **Pre-decrement** the counter; branch with **`jp k` / `jp nk`**

```asm
loop:
    ; body
    dec bc
    jp  nk,loop        ; adjust sense to match your initial count
```

Do not copy Z80 `dec bc; jp nz` semantics.

#### Alternative 16-bit counted loop structure.

Alternatively 16-bit loops can be created using the **`dec bc / inc b / inc c`** set up to create inner and outer loops, using any 16 bit register pair. Typically **`bc`** would be used, as **`hl`** and **`de`** have other priority uses.

```asm
   dec bc
   inc b
   inc c

loop:
   ; code body to be repeated bc times.

   dec c
   jr NZ,loop
   dec b
   jr NZ,loop
```

### 6. Multiply / divide building blocks — `rl de` + `sub hl,bc`

Shift-add mul and restoring div center on:

- **`rl de`** (and often **`add hl,hl`**) to shift
- **`sub hl,bc`** / **`add hl,bc`** to trial-subtract and restore
- **`ccf`** into quotient bits when dividing

Partial **unroll** when the body is small. Entry style: public DE/HL form → **`ld bc,de`** (or `ld bc,hl`) → HL/BC core so callers that already have BC can join mid-routine.

### 7. Shifts

**Signed 16-bit >>**

```asm
    sra hl
```

**Logical 16-bit >>** (no `srl hl`):

```asm
    sra hl
    ld  a,$7f
    and h
    ld  h,a            ; force bit 15 clear
```

**Logical multi-byte >>** (24/32-bit etc.): chain **`rra` through A** across bytes — not Z80 `srl`.

**32-bit <<** (value in DEHL):

```asm
    add hl,hl
    rl  de
```

**32-bit rotate** (sketch):

```asm
    rl  de
    ex  de,hl
    ; continue on the other half
```

**`rl de` as ×2 on DE** for table/struct scaling.

**Bitfield open/close on DE** (packed fields in D/E): open with **`rl de`**; repack with **`rra` via A** into D then E. Test a register for zero without destroying it: **`inc r` / `dec r` / `jp z`**.

### 8. Extra 16-bit slot — `ex (sp),hl`

Push a scratch word; **`ex (sp),hl`** swaps with it when AF/BC/DE/HL are full. **Do not** use `push af`/`pop af` as a free 16-bit temp or to hold a return address: F bit 3 is hardwired 0 (`$FFFF` → `$FF7F` on the round trip). **`pop af` is fine only when the popped word is discarded** (e.g. clearing intermediates off the stack in an epilogue).

### 9. I/O and Z80-only ops to avoid

| Prefer on 8085 | Avoid (Z80-only or wrong) |
|----------------|---------------------------|
| `out (*),a` / `in a,(*)` (byte in A) | `outi`, `in r,(c)`, block I/O |
| `dec b` / `jp nz` | `djnz` |
| Stack + DE for second long | `exx`, IX/IY as default temps |
| `sub hl,bc` | Assuming `sbc hl,de` exists |
| Open-coded extended-op sequences | Assuming Z80 library mul/div cores |

Assembler must be **8085-aware** (these encodings are not Z80 prefixes).

### 10. Synthetic opcodes

The z88dk-z80asm assembler has a MACRO capability, and it has the capability to support "synthetic" opcodes. Synthetic opcodes are usually created by a series of useful operations that can be chained together without side effects. For example `ld bc,de` which is a 16-bit emulation by two 8-bit operations, or for example `ld a,(hl+)` which is an indirect load followed by an increment of the index register. These operations can make code more concise and easier to debug, or allow an operation supported by one CPU to be used across multiple CPUs without deploying conditional assembly.

## Pitfalls

1. **`pop af` never for function return** — F bit 3 is hardwired 0, so AF cannot hold a correct return address. Use BC/DE/HL for the return word. **`pop af` is OK only to discard** intermediate stack values when the popped data is unused. AF is also not a clean 16-bit temp (`$FFFF` → `$FF7F`).
2. **`sub hl,bc` has no borrow-in** — multi-precision use A + `sbc`.
3. **K ≠ Z on 16-bit dec** — pre-dec + `jp k`/`jp nk`.
4. **Offsets on `ld de,sp+*` / `ld de,hl+*` are unsigned.**
5. **`rst v`** only if **0040h** is defined.
6. **`rla` / `rra` / `rlca` / `rrca` do not set Z** — never `rla; jp z,...`. Test with `or a` / `and a` / explicit mask first, or use `inc`/`dec` on a copy.
7. **No `exx`, IX, IY, `djnz`** — second long operand on stack; counted loops via `dec b`/`jp nz` or K pre-dec (§5).
8. **Forward overlapping stack copy corrupts** — see multi-word frame rebuild above.

## Preference order (when writing 8085-only code)

1. Stack-only locals/temps/intermediates; **static/BSS only for state that must survive across calls** — never intermediate storage.
2. **`ld de,sp+*`** for stack pointers; **`ex de,hl`** forms for HL←SP+n (see §2) over `ld hl,nn`/`add hl,sp` when offset is u8.
3. **`ld hl,(de)` / `ld (de),hl` / `ld a,(de)`** for stack traffic through DE.
4. **`sub hl,bc`** for 16-bit ==, !=, and signed compares with **K**.
5. **K + pre-dec** for 16-bit counted loops.
6. **`rl de`** for ×2, mul/div shifts, 32-bit with HL.
7. **`sra hl`** for signed 16-bit >>; logical multi-byte >> via A.
8. Fall back to 8080-portable sequences only when the binary must run without 8085 extended ops.

## Related

- Opcode map, flags, cycles: **`/opcode-reference`**
- Design notes: [feilipu.me — 8085 Software](https://feilipu.me/2021/09/27/8085-software/)
