---
name: extended-usage
description: >
  Usage hints and coding preferences for Intel 8085 extended (undocumented)
  instructions, using Zilog mnemonics. Strong rule: avoid static RAM variables;
  keep locals and intermediates on the stack. Prefer this skill when choosing
  idioms for stack frames, reentrant code, 16/32-bit shifts, loops with the K
  flag, comparisons with sub hl,bc, or optimizing 8085 assembly beyond the bare
  opcode map. Complements /opcode-reference. Use when the user runs
  /extended-usage or asks how to use 8085 extended ops effectively.
---

# 8085 Extended Instruction Usage

Preferences and idioms for the ten 8085 extended instructions. **Encodings, timings, and flags** live in **[opcode-reference](../opcode-reference/SKILL.md)** and its `references/opcodes.md`. This skill is about **when and how to use them**.

Primary source: [8085 Software — Extended Instructions & Discussion](https://feilipu.me/2021/09/27/8085-software/) (feilipu, 2021). Also formalized in the Tundra CA80C85B datasheet; present on every 8085.

**Always emit Zilog mnemonics** (project convention).

## Hard rule: stack variables, not static RAM

**Do not use static RAM for variables** (no fixed `label: ds n` / absolute `ld hl,(var)` scratch cells for locals, temporaries, or intermediate results) **unless there is absolutely no alternative**.

- **Locals, temporaries, and intermediate values** of implemented functions live **only on the stack** (arguments, return slots, pushed registers, and explicit frame space).
- Access them with **`ld de,sp+*`**, **`ld hl,(de)`**, **`ld (de),hl`**, **`ld a,(de)`**, pushes/pops, and **`ex (sp),hl`** — not with BSS/data-page statics.
- **Acceptable static/absolute memory** is limited to true globals that must outlive the call, hardware/MMIO, vectors, and read-only tables/constants in ROM — and even then, prefer passing pointers on the stack rather than baking in new static scratch.
- If you believe static RAM is required, **justify why the stack cannot hold it** (e.g. interrupt context with no private stack, or a value that must persist across unrelated calls with no owner frame). “Slightly fewer cycles” or “easier to write” is **not** enough.

This rule enables reentrancy, avoids hidden shared state, and matches the extended-instruction design goal (stack as working storage).

## Design intent

Extended ops are especially strong for:

- **Stack-relative / reentrant** code (HLLs, callees, math libs)
- Relieving pressure on the **small register set** (without falling back to static RAM)
- **16-bit** compare, subtract, and loop control
- **32-bit** rotate/shift sequences with DE+HL

They can be relied on for any real 8085 target. Prefer them over longer 8080-compatible sequences when writing new 8085-only code.

## Instruction preferences (cheat sheet)

| Zilog | Prefer for | Avoid / watch |
|-------|------------|----------------|
| `ld de,sp+*` | Stack parameter access; building SP-relative addresses | `*` is **unsigned** 8-bit offset |
| `ld de,hl+*` | DE ← HL+offset (pointer into struct/buffer) | Same unsigned offset rule |
| `ld hl,(de)` / `ld (de),hl` | Load/store 16-bit through DE (stack or table pointer) | Not Z80 `ed`/`dd` prefixes — single-byte 8085 ops |
| `sub hl,bc` | 16-bit subtract; **equality / inequality**; signed compares with K | **No borrow-in**; not for multi-word subtract chain |
| `sra hl` | Arithmetic right shift of 16-bit signed value | Clears V; C ← old bit 0 |
| `rl de` | 16-bit rotate left through C; `add de,de`-style ×2; 32-bit with HL | Pair carefully with `add hl,hl` / `ex de,hl` |
| `jp k,**` / `jp nk,**` | Loops after 16-bit `dec`; signed compare outcomes | K after **dec rp** sets on wrap to **−1**, not on 0 |
| `rst v` | Overflow trap if V set | Vector **0040h**; only if that vector is defined |

## Preferred idioms

### 1. Stack parameters (reentrant / working storage on stack)

Use SP-relative access instead of `ld hl,n` / `add hl,sp` when possible. **Do not spill intermediates to static RAM** to “make room” — push a frame slot or reuse stack via `ex (sp),hl` first.

```asm
    ld  de,sp+n        ; n = unsigned offset to word/byte on stack
    ld  hl,(de)        ; load word parameter
    ; or
    ld  a,(de)         ; load byte parameter
    ; store back:
    ld  (de),hl
```

**Why:** Classic `ld hl,n` + `add hl,sp` costs **21 cycles** and **clears C**, which is expensive when Carry holds useful state. `ld de,sp+n` preserves that better and is central to reentrant sccz80/`l_` style code.

**Cost intuition (from the article):**

- `ld de,sp+n` + `ld a,(de)` ≈ **4 cycles slower** than `ld a,(**)` absolute byte
- `ld de,sp+n` + `ld hl,(de)` ≈ **4 cycles slower** than `ld hl,(**)` absolute word  
  Stack values need no prior store to a static cell, so stack access often **wins overall** (roughly ~3 free stack accesses vs static setup). **Still prefer the stack** even when slightly slower — static scratch is last resort only.

### 2. Establish a stack frame quickly

```asm
    ld  hl,sp+n        ; synthetic/macro if assembler supports; else ld de,sp+n / ex de,hl
    dec h              ; example: carve a larger frame (see article pattern)
    ld  sp,hl          ; e.g. establish 256-n style frame when combined as described
```

Article pattern: **`ld hl,sp+n`**, **`dec h`**, **`ld sp,hl`** to establish a **256−n** stack frame. Use when allocating a frame in one go; keep frame layout documented.

### 3. Loops with K (LDIR-style, counters)

16-bit **`dec bc` / `dec de` / `dec hl` / `dec sp`** affect **K only** among useful flags for this pattern.

- K is set when the counter underflows to **−1**, **not** when it hits **0**
- Therefore: **pre-decrement** the loop counter, then **`jp k`** / **`jp nk`** accordingly

```asm
    ; count iterations with pre-decrement so K tracks "done" cleanly
loop:
    ; ... body ...
    dec bc
    jp  nk,loop        ; continue while not underflowed past 0 (tune to your count convention)
```

Tune `jp k` vs `jp nk` and initial counter so the exit matches **N** iterations; do not assume Z80 `dec`/`jr nz` zero semantics.

### 4. Signed compares with K + `sub hl,bc`

Use **`sub hl,bc`** then **`jp k` / `jp nk`** (and Z/S as needed) for signed relational tests. This is a major reason `sub hl,bc` exists in the opcode map—not only “subtract,” but **fast compare**.

Equality / inequality (very common in C, FatFs, regex, etc.):

```asm
    sub hl,bc          ; HL − BC
    jp  z,equal        ; or jp nz,not_equal
```

Prefer this over long 8080 compare expansions when both operands are in HL/BC (or can be).

### 5. 16-bit multiply / divide building blocks

- **`rl de`** and **`sub hl,bc`** are the workhorses for efficient 16-bit mul/div
- Byte savings vs pure 8080 often fund **partial loop unrolling**
- For **long (32-bit+) subtract with borrow**, do **not** expect `sub hl,bc` to take Carry in; use 8-bit **`sub` / `sbc`** through **A** for multi-word arithmetic

### 6. 32-bit rotate / shift (DE:HL or HL:DE conventions)

Rotate 32-bit field:

```asm
    rl  de
    ex  de,hl
    ; continue rotate on the other half as needed
```

Shift 32-bit field left:

```asm
    rl  de             ; or combine with
    add hl,hl
```

**`rl de` as `add de,de`:** double DE for structure/table scaling (index ×2, etc.).

### 7. Extra “register” via `ex (sp),hl`

If you push a scratch word so the return address sits at **SP+2**, then **`ex (sp),hl`** swaps with that scratch—another 16-bit slot when AF/BC/DE/HL are full.

After an extra push to make room:

- **SP+0** — top of stack (scratch / ex partner)
- **SP+2** — return address (if one word pushed under return… layout depends on push order; **document the frame**)
- **SP+4** — first stack variable in the article’s mental model when return is at SP+2

Always draw the frame; do not assume SP+k without counting pushes.

## Pitfalls (preferences to avoid bugs)

1. **Annoying AF:** bit 3 of F is hardwired **0**. `push af` / `pop af` of `$FFFF` yields **`$FF7F`**. You **cannot** use AF as a transparent temporary 16-bit stack slot the way Z80 code sometimes does. Prefer DE/HL/BC or **stack** slots — not static RAM.

2. **`sub hl,bc` has no borrow-in:** good for HL−BC and compares; bad as the middle of a multi-precision subtract chain—use A with `sbc`.

3. **K vs Z on 16-bit dec:** loop on K with **pre-decrement**; do not copy Z80 `dec bc; jp nz`.

4. **Offset unsigned:** `ld de,sp+*` / `ld de,hl+*` — offset is **unsigned**; negative frame offsets need a different sequence.

5. **Not Z80:** `ld hl,(de)`, `ld (de),hl`, `rl de`, `jp k`, etc. are **8085 encodings**. Do not emit Z80 prefix bytes. Assembler must be 8085-aware (e.g. z88dk-z80asm).

6. **`rst v`:** only if 0040h handler exists; otherwise overflow should use explicit `jp` on V testing via code that sets up V, or avoid.

## Optimization preferences (priority order)

When writing or reviewing 8085-only assembly:

1. **Stack only for function variables** — no new static/BSS temporaries; allocate intermediates on the callee/caller stack frame. Static RAM only if **no alternative** exists (see Hard rule).
2. Prefer **`ld de,sp+*` + `ld hl,(de)` / `ld (de),hl` / `ld a,(de)`** for stack traffic over `add hl,sp` sequences when C must be preserved or size/speed matter.
3. Prefer **`sub hl,bc`** for 16-bit **==**, **!=**, and as the core of signed compares with **K**.
4. Prefer **K + pre-dec** for 16-bit counted loops.
5. Prefer **`rl de`** for ×2 on DE and for 32-bit shift/rotate pairs with HL.
6. Use **`sra hl`** for signed 16-bit divide-by-two / arithmetic shift.
7. Fall back to 8080-portable code only when the binary must also run on 8080/Z80-without-8085-ops.

## Related

- Opcode map, flags, cycle counts: **`/opcode-reference`**
- Background and measured library impact: [feilipu.me — 8085 Software](https://feilipu.me/2021/09/27/8085-software/)
- Example optimized primitives: z88dk `libsrc/_DEVELOPMENT/l/sccz80/7-8085/` (mul, div, eq, long ops)
