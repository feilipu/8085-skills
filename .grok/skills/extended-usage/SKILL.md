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

## Lessons from sccz80 practice (Z80 vs 8085)

Source tree in z88dk: `libsrc/l/sccz80/`

| Directory | Role |
|-----------|------|
| **5-z80** | Often thin wrappers into the large Z80 math library (`l_muls_*`, `l_divu_*`, IX/`exx`) |
| **7-8085** | Self-contained 8085-tuned bodies using extended ops |
| **9-common** | Portable 8080-style byte compares / stores |
| **8-8080** | Full 8080 mul/div without `rl de` / `sub hl,bc` |

“Z80 vs 8085” is not always the same algorithm twice: Z80 frequently **outsources** to a fast Z80 core; 8085 **implements** the primitive with extended instructions. Use **7-8085** as the model for hand-written 8085 code.

### 1. `sub hl,bc` is the compare/subtract primitive

| Path | Approach |
|------|----------|
| Z80 `l_eq` | `or a` / `sbc hl,de` then test Z |
| 8085 `l_eq` | `ld bc,de` / **`sub hl,bc`** then test Z |
| 9-common | Byte-wise `sub`/`sbc` on L then H |

Same for **`l_ne`**. 8085 also exports **`l_eq_hlbc` / `l_ne_hlbc`** so callers that already hold BC skip `ld bc,de`.

Signed/unsigned relations in **7-8085** (e.g. `l_ge`, `l_lt`, `l_uge`) use one subtract + **K/Z/C**, not 9-common’s “add `$80` and `cp`” path:

```asm
; 8085-style: DE >= HL (signed) — see l_ge
    ld  bc,de
    sub hl,bc
    jp  k,true
    jp  z,true
    ; false
```

**Do this:** On 8085-only code, prefer **`sub hl,bc` + Z/C/K** for 16-bit ==, !=, and ordered compares. Reserve 9-common-style sequences for 8080 portability only.

### 2. `rl de` is the mul/div shift engine

- **`l_rlde`:** 8085 is a single `rl de`; 8080 needs four instructions via A.
- **`l_mult`:** 8085 uses partially unrolled shift-add: **`add hl,hl` + `rl de`**, add BC when carry; product in HL, multiplier rotates in DE. Z80 jumps to `l_muls_16_16x16`.
- **`l_div_u`:** 8085 restoring division with **`rl de`**, **`ex de,hl`**, **`sub hl,bc`**, restore via **`add hl,bc`**, **`ccf`** into quotient bits; often **two bits per loop body** (partial unroll). 8080 uses a long RLA loop and even **push/pop AF** as a counter.

**Do this:** Build 16-bit mul/div on **`rl de` + `sub hl,bc` + `add hl,bc`**. Unroll when the extended ops keep the body tiny.

### 3. `sra hl` vs Z80 `sra h` / `rr l`

```asm
; Z80 signed >>
    sra h
    rr  l
; 8085 signed >>
    sra hl
```

No `srl hl` on 8085 — **unsigned >>** in `l_asr_u` is:

```asm
    sra hl
    ld  a,$7f
    and h
    ld  h,a          ; clear bit 15 after arithmetic shift
```

**Do this:** Signed 16-bit right shift → **`sra hl`**. Logical right shift → **`sra hl` + force bit 15 clear**.

### 4. Stack as memory: `ld de,sp+n`, `ld hl,(de)`, `ld (de),hl`

- **`l_pint_pop`:** 9-common stores a word as two byte writes; 8085 is **`ld (de),hl`**.
- **`l_long_add` / `l_long_sub`:** secondary in DEHL; primary on stack; **`ld de,sp+2`** then byte `add`/`adc` or `sub`/`sbc` through A while walking with `inc de`; finally drop the 4-byte primary around the return address. Z80 long sub often uses **IX** and **`sbc hl,bc`**.
- **`l_long_mult`:** push secondary; pull primary/secondary pieces with **`ld de,sp+N` / `ld hl,(de)`**; partial products on stack; rewrite return with **`ld (de),hl`**; **`ld sp,hl`** to discard parameters — **no static cells**.

**Do this:** Multi-word and multi-arg work uses **SP-relative DE + word load/store**. Do not invent BSS temps because Z80 would have used IX/`exx`.

### 5. 32-bit left shift: `add hl,hl` + `rl de`

```asm
; 8085 l_long_asl loop (dehl <<= 1)
    add hl,hl
    rl  de
```

Z80 typically pops 32-bit and jumps to a shared `l_lsl_dehl`. Same structure; 8085 keeps it local and explicit.

### 6. What Z80 has that 8085 substitutes

| Z80 asset | 8085 substitute in sccz80 |
|-----------|---------------------------|
| `sbc hl,de` | `sub hl,bc` (after `ld bc,de`) or byte `sub`/`sbc` |
| `exx`, IX/IY | Stack traffic; dual entry points (`*_hlbc`) |
| Fat `l_mul*` / `l_div*` library | Open-coded shift-add/div with extended ops |
| `srl h` | `sra hl` + mask bit 15 |

8085 is not copying every Z80 micro-op; it **closes the 8080 gap** and often reaches Z80-level library results (e.g. equality-heavy code) via these substitutes.

### 7. Entry-point style worth copying

1. **Public DE/HL form** → **`ld bc,de`** → **HL/BC core** (`l_eq_hlbc`, `l_mult_0`, `l_div_0`).
2. **Compare results** via tiny helpers: HL=1 + carry set (true), HL=0 + carry clear (false).
3. **Partial unroll** when extended ops shrink the loop body (mul/div).
4. **Signed div** = take absolutes → unsigned core (`l_div_0`) → fix quotient/remainder signs (remainder follows dividend).

### Practice rules (summary)

1. **Compares:** `sub hl,bc` + Z/C/K — not 9-common’s `$80` bias — when 8085-only.
2. **Mul/div/shift:** **`rl de`** and **`sub hl,bc`**; unroll if the body stays small.
3. **Signed >>:** **`sra hl`**; logical >> = sra + clear bit 15.
4. **Word through DE:** always **`ld (de),hl` / `ld hl,(de)`**.
5. **Long ops / multi-arg:** **`ld de,sp+n`**, walk with `inc de`, drop args by adjusting SP around the return — **no static RAM**.
6. **32-bit left shift:** **`add hl,hl` / `rl de`**.
7. **Do not ape Z80 IX/`exx` on 8085** — ape **stack + BC as second operand**.

## Lessons from IEEE float / 32-bit practice (Z80 vs 8085)

Derived from parallel Z80 and 8085 implementations in z88dk  
`libsrc/math/float/am9511/asm/{z80,8085}/` (same-named pairs for stack traffic, compare, frexp/ldexp, ×2/÷2/×10, classify, arg swap, format conversion, device I/O).

Here the **algorithm is often the same** on both CPUs; the split shows how to **replace Z80 register-set luxuries** with extended 8085 ops while keeping **stack-only** working storage. Complements the sccz80 section (integer CRT primitives).

### 1. `exx` → stack + `ld de,sp+n` (second 32-bit value on the stack)

| Z80 | 8085 |
|-----|------|
| Keep **left** in DEHL and **right** in DEHL' via repeated **`exx`** (binary float compare) | Point with **`ld de,sp+4` / `ld de,sp+8`**, load MSWs with **`ld hl,(de)`**, process in place |
| **`exx`** to preserve DEHL across a helper that needs the primary regs | Save DE in BC if needed: **`ld bc,de`**, **`ld de,sp+n`**, restore **`ld de,bc`** |
| Pop args into registers early | Leave args on stack; read with SP-relative DE; clean frame at exit |

**Arg swap / SP math:** Z80 often does `ld hl,n` / `add hl,sp`; 8085 prefers **`ld de,sp+n`** — same layout, cheaper SP addressing.

**Binary compare (8085):** may **mutate stack parameters** (e.g. total-order bit flips for signed IEEE sort keys); creates **scratch words on the stack** (`push` / `ld de,sp+0`) for multi-byte subtract results — still **no BSS**.

**Do this:** When Z80 would use **`exx` as a second DEHL**, on 8085 keep the second value **on the call stack** and address it with **`ld de,sp+*`**. Prefer reordering or carefully documented in-place stack updates over static 32-bit cells.

### 2. IEEE sign/exponent field: prefer `rl de` (not `sla e` / `rl d` / `rr e`)

Z80 float helpers commonly peel the biased exponent with:

```asm
    sla e
    rl  d        ; sign → C, exponent in D
    ...
    rr  d
    rr  e        ; repack
```

8085 equivalents use:

```asm
    rl  de       ; sign → C; exponent ends in D (mantissa bits shift in E)
    ; ... work on D (and EHL) ...
    ld  a,d
    rra
    ld  d,a
    ld  a,e
    rra
    ld  e,a      ; repack sign+exponent through A (no 16-bit rr de)
```

**Test “register was zero”** without destroying it (e.g. exponent in D after isolate):

```asm
    inc d
    dec d
    jp  z,zero_case
```

**Do this:** For IEEE DEHL packing, treat **`rl de`** as the canonical open of the sign+exponent field. Repack with **byte `rra` through A**. Use **`inc r` / `dec r` / `jp z`** to test a register for zero.

### 3. No `srl` / Z80 block I/O — byte A paths

| Z80 | 8085 practice |
|-----|----------------|
| `srl e` / `rr h` / `rr l` (logical multi-byte >>) | Chain **`rra` through A** across E, H, L |
| `outi` / `out (c),r` / `in r,(c)` | **`ld a,…`** then **`out (*),a`** / **`in a,(*)`** (or thin macros that leave the byte in A) |
| `djnz` | **`dec b` / `jp nz`** |
| `jr` | **`jp`** is fine (relative jump optional) |

**Scaled mantissa work** (e.g. ×10 as `2*(4*a+a)`): same algebra on both CPUs; Z80 may use `srl`, 8085 uses A-based shifts and **stack temps** (`push de` / `push hl` / `ex (sp),hl` / `add hl,de`) for partial sums.

**Do this:** Logical multi-byte shifts go **through A**. Port I/O goes **through A**. Do not emit Z80 block I/O or `srl`.

### 4. Same high-level optimisations on both CPUs (keep them)

These are **not** 8085-specific but must survive a Z80→8085 port:

- **×2 / ÷2 float** = **inc/dec the exponent**, with overflow/underflow/NaN traps — not a full multiply/divide.
- **IEEE zero** ≡ **exponent field zero** (for “is zero” in compare/classify, mantissa can be ignored).
- **Total-order float compare** (flip sign bit; if negative flip all bits) then **unsigned multi-word subtract** — float order becomes integer order.

**Do this:** Preserve these algorithmic shortcuts; only change the **register/stack mechanics**.

### 5. Callee / stack cleanup patterns

Multi-arg float helpers (e.g. frexp-style: value + pointer out-parameter) on 8085 often:

1. Read args with **`ld de,sp+n`** / **`ld hl,(de)`** without popping first  
2. Write through a pointer in BC  
3. End with a **structured epilogue** that drops stack args and restores the return address:

```asm
    pop bc          ; return
    pop hl          ; keep what must survive
    pop af          ; drop consumed arg words
    pop af
    push bc         ; return
```

Z80 often pops into registers at entry instead. Both end with a clean stack; neither should park intermediates in static RAM.

**Do this:** For multi-arg callees, either pop to regs (Z80 style) **or** SP-relative access + **one structured epilogue** — never a global temp.

### 6. Generalised rules (any 8085 code)

1. **Second 32-bit value** → stack slot + **`ld de,sp+*`**, not **`exx`**.
2. **Preserve DEHL across a helper** → stack or **`ld bc,de`** (if BC free), not **`exx`**.
3. **IEEE/bitfield rotate on DE** → **`rl de`** open; **`rra` via A** close.
4. **Logical >> on multi-byte** → A-chain **`rra`**, not **`srl`**.
5. **I/O** → A + port address; no **`outi` / `in r,(c)`** assumptions.
6. **Scratch** for compare/mul partials → **push frame / `ex (sp),hl`**, not BSS (reinforces hard rule).
7. **Keep domain optimisations** (exp ±1 for ×2/÷2, exponent-zero ⇒ zero, integerised float compare).

## Related

- Opcode map, flags, cycle counts: **`/opcode-reference`**
- Background and measured library impact: [feilipu.me — 8085 Software](https://feilipu.me/2021/09/27/8085-software/)
- Integer CRT reference: z88dk `libsrc/l/sccz80/7-8085/` (contrasts: `5-z80/`, `9-common/`, `8-8080/`)
- Float practice source: z88dk `libsrc/math/float/am9511/asm/8085/` (contrasts: `.../asm/z80/`)
