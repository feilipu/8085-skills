---
name: compiler-c
description: >
  C90 compiler guide for Intel 8085: emit Zilog assembly for z88dk-z80asm
  -m8085 from standard C. Data model, SMALLC ABI, stack frames, HL as ALU
  bus, BC/DE parking, 8085 extended ops, C90 shape → ISA lowering. Use for
  /compiler-c, "C compiler guide for 8085", "compile this C to 8085
  assembly", "8085 C lowering". Do not use when invoking zcc, 80cc, or
  sccz80 as the compiler, or when editing src/80cc (those are
  compiler-80cc, compiler-sccz80, tool-zcc).
---

# C compiler guide — 8085

Compile **ISO C90** to **Zilog** 8085 assembly that `z88dk-z80asm -m8085`
accepts. Load **`cpu-8085`** for the opcode map, flag effects, timings, and
stack-access sequences. This card is the **C** layer: types, ABI, register
assignment, and how C constructs map onto that ISA.

Optimise from the **8085 instruction set**, not from Z80 habit. Extended
ops (`ld de,sp+*`, `ld hl,(de)`, `ld (de),hl`, `sub hl,bc`, `rl de`,
`sra hl`, `jp k` / `jp nk`) are first-class on every 8085. There is no IX,
no `exx`, no native `djnz`, and no 2-byte `jr` (`18` is `rl de`; assembler
`jr` becomes a 3-byte `jp`).

**Shape before statements.** Name the C90 shape in a function (walking
byte array, word swap, pre-dec-to-zero, struct cursor, …) and emit the
ISA sequence for that shape. Do not lower a naïve `a[i]` / `i++` form
when the shape is a pointer walk or an induction.

## C data model on 8085

Default **`char` is signed**. Little-endian. Integer promotions go to
16-bit `int`. `short` is 16-bit (same home as `int`). Struct members:
`offset += size` (no 2-byte ABI holes).

**Unsigned bitfields** (classic 8085 emit): pack **LSB-first** in the
declared unit (`unsigned` → 16-bit, `unsigned long` → 32-bit). Adjacent
fields share a unit; a field that does not fit starts a new unit. Emit
read-modify-write on that unit — not a C bitfield opcode (none exists).
Offset-0 width-*n* is a mask only (`and (1<<n)-1`). A field that spans a
byte boundary is a multi-byte RMW. `f->rate++` is extract / add / mask /
insert, not `inc` of the whole unit. Consecutive writes to the same
object: keep the unit live in HL or DEHL; store once. **Signed**
bitfields, or a binary layout that must match another compiler: stop.

If the source typedefs 1/2/4-byte aliases (`BYTE`, `WORD`, `DWORD`,
`UINT`, `WCHAR`, …), use those widths. They are not extra register homes.
`QWORD` / `long long` is memory + helpers.

| Type | Bytes | In registers |
|------|------:|----------------|
| `char` | 1 | **L** on return (H dead); park live bytes in **C**, not A |
| `unsigned char` | 1 | same; zero-test is `or a`, not a signed compare |
| `short` / `int` / `enum` / near pointer | 2 | **HL** as ALU result; park in **BC** or **DE** |
| `long` / `unsigned long` | 4 | **DEHL** (DE high, HL low) |
| `long long` | 8 | Memory + `__i64_acc` / helpers; not a register home |
| `float` / `double` / `double_t` | maths mode | IEEE32 = 4, DEHL; MBF32 = 4; genmath is not IEEE64. If the source has `float` and no library is named, **ask** (TIMER work) or use IEEE32 for a toy |
| `_Float16` | 2 | `library-math16` |

Ingest C90 plus common source sugar: `//` comments (emit as `;` — Comments),
implicit-int `main()`,
`for (T i = 0; …)` and mixed declarations (hoist to the enclosing block;
do not treat as C99), `TIMER_*` macros as **labels** at those sites,
`register` as a BC/DE hint only. `Assert` / `assertEqual` bind as ordinary
calls.

**Storage duration follows the C, not the ISA.** BSS/data only for objects
that already have static storage duration in the source:

| Source | Storage |
|--------|---------|
| Block-scope **without** `static` (automatics, including `-DSTATIC` identifiers the source did **not** wrap) | **Stack only.** Never BSS, never a compiler scratch cell |
| The **`static` keyword** (block- or file-scope) | BSS/data as declared |
| File-scope without `static` | Still C static storage duration (external BSS/data). Do not move it to the stack |
| Temps, spills, induction variables, register homes | Stack, BC, DE, or C — **not** BSS |

Do not insert `static` the source did not write. `-DSTATIC` only affects
identifiers the source spelled `STATIC` (that macro becomes `static`). Do
not promote every local. Do not apply it to automatics of a recursive
function the source left automatic.

Refuse: VLA, designated initializers, nested functions. Mixed declarations
and `for`-init declarations: hoist, then emit C90.
`stdint.h` only if the source already uses it. Do not mix newlib `FILE*`
cores into a classic 8085 image (`library-classic`). Do not rewrite
1-based arrays to 0-based.

`sizeof` is a compile-time constant. String concatenation of literals is
one object.

## ABI (classic 8085 C)

One convention: **SMALLC** — arguments pushed **left to right**, **caller
cleans**. Fastcall / callee only when the **prototype** says so. Do not
invent a private register-only ABI.

C `foo` → asm `_foo`. `char` arguments occupy **2** stack bytes.
Return address is 2 bytes (`params-offset` 2). No saved IX.

```
high addresses
  first-pushed argument
  …
  last-pushed argument
  return address          ; [sp+0] on entry
  callee locals / pushes  ; grow down
low  (SP)
```

`__stdc`: reverse stacked args. `__z88dk_fastcall`: last scalar already in
HL (or DEHL if 32-bit). `__z88dk_callee`: callee pops (use `pop af` **only**
to discard a word).

| Return | Location |
|--------|----------|
| `char` / width-1 | **L only; H dead.** Caller extends. Do not `ld h,0` in the callee |
| 16-bit | HL |
| 32-bit int / IEEE32 float | DEHL |
| `long long` | hidden pointer / `__i64_acc` |

**Variadic SMALLC** (`printf` and any `__smallc` varargs): `ld a,N` (or
`xor a`) immediately before `call`. N = argument **words** on the stack.
A stuffed `&__i64_acc` is not part of N. Non-variadic calls must not assume
A is live.

**Libc:** do not lower headers. Bind the **preprocessed** prototype to a
library `PUBLIC`: `__z88dk_callee` → `call _foo_callee` (callee pops);
plain SMALLC → `call _foo` (caller pops). C may see `__builtin_memset`;
asm uses `_memset` / `memset`. Never `EXTERN __builtin_memset`. Combined
`/` and `%` of the same operands: one helper (`quot` and `rem`), not two
divides.

If the source defines `ld_16` / `ld_32` / `st_16` / `st_32` as
`*(WORD *)` / `*(DWORD *)`, emit native little-endian word traffic
(`ld hl,(de)`), not a portable byte-shift body. Unaligned LE loads are
legal on this CPU.

Two image shapes, same ABI:

| Shape | Use |
|-------|-----|
| Relocatable module | `SECTION` + `PUBLIC _name`; CRT supplies `_main` / start |
| Complete image | Tiny `start` plus your functions; still SMALLC |

```asm
    SECTION code_compiler
    PUBLIC  _foo
    EXTERN  _memset

_foo:
    ret
```

Zilog, lowercase, four-space indent. No Intel names (`LXI`, `DSUB`, `LDSI`).
Blank line after a PC break: **Listing layout**.

## Registers — what the ISA actually gives C

The 8085 is not a bank of interchangeable GPRs. Treat it as a **bus** plus
two parking pairs.

```text
Bus (scratch, not homes)
  A     8-bit ALU,  ld a,(de) / ld (de),a / ld a,(bc)
  HL    16-bit ALU, (hl), addresses, add hl,rp, sub hl,bc

Parking (C locals that must survive the bus)
  C     8-bit local   — never A (ALU) and never L inside a live HL
  BC    count, stride, 8-bit (bc) cursor
  DE    word cursor, 2nd operand, stack cursor, 32-bit high half
  stack everything else
```

**Cursor law (this ISA):**

| What you walk | Home | Why |
|---------------|------|-----|
| Word array / `int *` / pointer-to-struct | **DE** | only `ld hl,(de)` / `ld (de),hl` |
| Byte array, any-register load/store | **HL** | `ld r,(hl)` / `ld (hl),r` |
| Byte array, A-only | DE or BC | `ld a,(de)` / `ld (de),a` / `(bc)` |
| Two byte strings at once | **HL** and **DE** | `ld a,(de)` vs `cp (hl)` / `ld (hl),a`. Park one cursor on the stack if HL is needed as ALU |
| Two word arrays at once | **DE** + **stack** | No second `(de)` bus, no IX. Load one array through DE; the other cursor lives on the stack (`ex (sp),hl` or a slot) |
| Stride or trip count | **BC** | `add hl,bc` keeps the cursor in HL |

There is no `ld hl,(bc)` and no `ld (bc),hl`. Do not put a word cursor in
BC. `ex de,hl` is a native 4-cycle swap. Use it when it wins. There is no
alternate bank: a second live 32-bit value lives on the **stack**.

**After any op that uses HL as the bus, the previous HL is gone** unless
it was parked in BC, DE, or a stack slot.

### Assignment heuristic

1. Hottest 16-bit values (loop index, walking pointer) in **BC** or **DE**
   across a window with no `call`. Word cursor prefers DE; stride prefers BC.
2. Keep a stack cursor in **DE** and use `ld hl,(de)` / `ld (de),hl` /
   `ld a,(de)` rather than re-materialising SP every access.
3. One live `long` or IEEE32: **DEHL**. A second: stack slots.
   Two walking **word** pointers: DE plus a stack slot — not IY/`exx`.
4. 8-bit live values in **C** (or B if C is taken). A is the 8-bit ALU.
   Park an 8-bit accumulator in C while a float kernel uses DEHL/A.
5. `volatile`: memory at every sequence point — no BC/DE home that skips
   a reload or store.
6. Do not park a 16-bit value in **AF**. `pop af` forces F bit 3 to 0
   (`$FFFF` → `$FFF7`); never use AF for a return address.

Hot-function homes go in the function header **Uses** line (Comments).

## Frames — stack only

No frame pointer. Automatics live on the stack. BSS/data only where C
gives static storage duration (`static` keyword, or file-scope). Do not
invent BSS for temps, and do not rewrite a `static`/file-scope object to
the stack “for 8085 style.”

Prefer **`cpu-8085`** sequences:

| Need | Sequence | Why it wins |
|------|----------|-------------|
| Pointer to a slot, HL free | `ld de,sp+n` | 2 bytes, 10 cycles, no flags |
| Word load/store | `ld hl,(de)` / `ld (de),hl` | Native LHLX/SHLX |
| Byte load/store | `ld a,(de)` / `ld (de),a` | Only legal `(de)` byte forms |
| HL = SP+n, DE free | `ld de,sp+n` / `ex de,hl` | Cheaper than `ld hl,nn` / `add hl,sp` when n is 0…255 |
| Offset > 255 | `ld hl,nn` / `add hl,sp` | `*` on LDSI/LDHI is **unsigned 8-bit** |

If DE is a live home, `push de` around the SP op and add 2 to the offset.
Function headers (purpose, inputs with this map, outputs, registers): **Comments**.

```asm
; void f(int a, int b)   /* SMALLC: push a, then b; caller cleans */
; entry: [sp+0]=ret, [sp+2]=b, [sp+4]=a
; locals: push, or ld hl,-n / add hl,sp / ld sp,hl
; exit: restore SP; ret
```

## Comments

Carry the C into the listing. The `.asm` must still show intent.

**C comments.** Every `/* … */` and `//` appears as `;` at the matching site. Do not drop them when hoisting mixed declarations or rewriting a shape. Keep the C wording. If that wording is too thin to identify the following instructions, **expand** the comment with the C the block implements (the statement, the named shape, or a one-line restatement of the function).

**Function header (required).** Immediately before each `_name:` (and before `start` in a complete image), even when the C function has no comment:

```asm
; _foo — walk p for n bytes and return the count
; C: int foo(int n, char *p)
; Inputs:  n at [sp+4], p at [sp+2]; [sp+0]=ret; SMALLC, caller cleans
; Outputs: HL = count
; Uses:    DE = p; BC = n; A HL scratch
```

- **Purpose** — from the C comment on the function if present; otherwise what the function computes.
- **C:** the C90 signature (hoisted types, not C99 `for`-init).
- **Inputs:** each parameter: type, name, stack slot or incoming register (`__z88dk_fastcall` last scalar in HL / DEHL). Include the SMALLC map.
- **Outputs:** ABI return home (`L` / `HL` / `DEHL` / void). Out-parameters: address + what is stored.
- **Uses:** live homes (which C object in BC, DE, C, DEHL, which stack slots). A and HL as bus are scratch unless a home. If the body `call`s, **A F BC DE HL** die across that call.

**Attached C on non-obvious blocks.** After a shape rewrite, one `;` line with the C that block implements (`while (p < end)`, `acc = (acc << 1) ^ K`, `y[i] = a*x[i] + y[i]`). Do not narrate every opcode.

Zilog `;` only. Same-line or above the block. Do not invent commentary that is not the required header, a carried C comment, or an expansion of the C being implemented.

## Listing layout

One **blank line** after any instruction that does not fall through: `jp`, `jp cc` (`z`/`nz`/`c`/`nc`/`k`/`nk`/`m`/`p`/`pe`/`po`), `jp (hl)`, `ret`, `ret cc`. Assembler `jr` is a 3-byte `jp` here — same blank.

**Not** after `call` / `call cc`: the program counter returns. Do not insert a blank between `call` and the next instruction.

```asm
    sub hl,bc
    jp  c,less

    ; unsigned >=
    ...
    ret

less:
    ...
    call l_mult_ulong
    ld   a,e           ; next insn; no blank after call
    ...
    ret
```

## Helpers (library vs inline)

When a C op is not a few native/extended insns, **consider** the 8085 catalogs below (integer helpers, integer math, IEEE32, half float). **Hot path: inline** a short body instead of calling (`*10` shift-add, DSUB compare, `ld de,sp+*`, `rl de`, `sra hl`, `<<8` byte move). One-shot or bulky work (general mul/div, 32-bit mul, float) **`call`** the catalog name and `EXTERN` it. Every `call` clobbers **A F BC DE HL**.

| Catalog | Use |
|---------|-----|
| `libsrc/l/sccz80/8085.lst` | 16/32-bit sccz80 runtime (`l_mult`, `l_div`, `l_div_u`, `l_mult_ulong`, `l_long_*`). Includes `8080.lst` for the rest |
| `libsrc/l/util/8085.lst` | 32-bit shifts `l_lsl_dehl` / `l_asr_dehl`; small ASCII `l_small_utoa` / `l_small_atoul` / `l_small_htoul` / `l_small_otoul` |
| `libsrc/math/integer/small/` | `l_small_mul_*` / `l_small_muls_*` / `l_small_divu_*` / `l_small_divs_*` (16/32/64). **16×16→32** is `l_small_mul_32_16x16` or `l_mult_ulong` (DEHL = DE×HL) |
| `libsrc/math/float/math32/` (`asm/8085/`) | IEEE32 cores: `f32_fsadd`, `f32_fsmul`, `f32_fsdiv` (**restoring**), `f32_fsinv` (NR — not for `1.0/x`), `f32_fssqrt`, `f32_fscompare`, `f32_fsconv`, `f32_f2long`, … Higher: `m32_sinf` and peers. Policy: **`library-math32`** |
| `libsrc/math/float/math16/` (`asm/8085/`) | Half: `asm_f16_add` / `mul` / `div` (restoring) / `inv` (NR) / `sqrt` / `compare` / … Higher: `sinf16` and peers. Policy: **`library-math16`** |

**Open-code; do not call** on 8085: `l_eq`/`l_ne`/`l_lt`/`l_le`/`l_gt`/`l_ge`/`l_ult`/`l_ule`/`l_ugt`/`l_uge` (`sub hl,bc` + K/C/Z); `l_rlde` (native `rl de`); `l_gint*sp` (`ld de,sp+*` / `ld hl,(de)`); `l_pint_*` (`ld (de),hl`); `l_asr` / `l_asr_u` when the count is 1 or a small constant (`sra hl` / logical `>>`). Do not bind `l_setix` / `l_setiy` / f48.

16×16→16 is `l_mult`. 16×16→32 is **`l_mult_ulong`** or **`l_small_mul_32_16x16`**, not `l_mult`. Combined `/` and `%`: one `l_div` / `l_div_u` / `l_long_div*`.

## C → 8085 primitives

| C | Emit |
|---|------|
| Automatic (no `static`) | Stack slot; cursor in DE. **Not** BSS |
| `static` keyword / file-scope | BSS/data as declared. Do not invent extra BSS |
| `if (x)` / `while (x)` (16) | Test that **writes Z** (`ld a,h` / `or l`) — leftover K is not a truth test |
| `if (p)` pointer / `== NULL` | `ld a,h` / `or l` |
| `if (c)` byte | `ld a,c` (or `(hl)`) / `or a` |
| `a && b` / `a \|\| b` | Short-circuit; skip the second arm. Side-effecting operands **must not** run when skipped. Each arm writes Z. Cheap int test before a float/call |
| `x + y` (16) | `add hl,de` or `add hl,bc` |
| `x - y` / `==` / `!=` (16) | y in BC; `sub hl,bc`; **Z** for `==` / `!=` |
| signed `<` / `>=` (16) | `sub hl,bc` then **immediately** `jp k` / `jp nk` |
| signed `<=` / `>` (16) | same `sub hl,bc`: `<=` is K **or** Z; `>` is NK and NZ |
| unsigned `<` / `>=` (16) | `sub hl,bc` then **immediately** `jp c` / `jp nc` — **C**, not K |
| unsigned `<=` / `>` (16) | same: `<=` is C **or** Z; `>` is NC and NZ |
| `*p` / `*p =` (word) | pointer in DE: `ld hl,(de)` / `ld (de),hl` |
| `*p` / `*p =` (byte) | pointer in HL: `ld a,(hl)` / `ld (hl),a` (or `(de)` if A-only) |
| `p->field` | Pointer in HL: `ld de,hl+off` (off unsigned 8-bit) then `(de)`. Pointer already in DE: `ld hl,off` / `add hl,de` (or `ex de,hl` first). Off > 255: `ld hl,nn` / `add hl,de` |
| `p++` (byte / word ptr) | `inc de` / `inc de` twice (or `ld hl,2` / `add hl,de`) |
| `*p++ = byte` | `ld (de),a` / `inc de` |
| `s << 1` (16) | `add hl,hl` |
| `s << n` n const 2…7 | repeated `add hl,hl` |
| `u << 8` / `u >> 8` unsigned 16 | **byte move**: `ld h,l` / `ld l,0`; `ld l,h` / `ld h,0`. Not eight `add hl,hl` |
| `u >> 12` unsigned 16 | `>>8` then two `sra hl` + clear H7, or `ld a,h` / four logical `>>` |
| signed `s >> 1` | `sra hl` (**Z unchanged**) |
| unsigned `u >> 1` | `sra hl`; force H bit 7 clear (**Z unchanged**) |
| variable `u << n` / `u >> n` (n live, 1…15) | n in B: `add hl,hl` or logical `>>`; `dec b` / `jp nz`. n==0 is a no-op |
| variable `byte << n` / `>> n` | n in B: `add a,a` or `or a` / `rra` (logical). Signed byte `>>`: sign-extend to HL first, then `sra hl`, then take L |
| `(v << n) \| (v >> (16-n))` | 16-bit rotate: park v; `<< n`; OR with logical `>> (16-n)` |
| `long << 1` | `add hl,hl` / `rl de` |
| `unsigned long >> 1` | `or a` / `rra` through A across D,E,H,L — not `sra hl` on both halves. C after the last `rra` is the old bit 0 |
| `int * int` | shift-add for small constants; else `call l_mult` (HL = DE×HL) |
| `x * 2` / `* 3` / `* 5` / `* 8` / `* 10` / `* 25` | `*8`=`add hl,hl`×3; `*10`=`*8+*2`; `*25`=`*16+*8+*1`. Do not `l_mult` these in a hot loop |
| `(unsigned long)a * (unsigned long)b` of two 16-bit values | **16×16→32** (`l_mult_ulong` / `l_small_mul_32_16x16`), then keep DEHL. `l_mult` (16×16→16) is a miscompile |
| signed `/` `%` | `call l_div` unless power-of-two |
| unsigned `/` `%` | `call l_div_u` unless power-of-two |
| both `/` and `%` of same ops | **one** helper; take quot and rem |
| `x % 2` | `ld a,l` / `and 1` — not `l_div` |
| `x % (1<<n)` unsigned | `and (1<<n)-1` (512 → `and 0x01FF`) |
| `x / (1<<n)` unsigned | n logical `>>` (sra + clear high bits, or drop bytes) |
| `x / 2` (non-neg) | `sra hl` |
| signed `v / (1<<n)` | C rounds **toward zero**. `sra hl` rounds toward −∞. If v<0, add `(1<<n)-1` then `sra` n times. `%` = v − quot×(1<<n) |
| LE `*(WORD *)p` | pointer in DE: `ld hl,(de)` — 8085 is little-endian, unaligned is legal |
| LE `*(DWORD *)p` | `ld hl,(de)` (low) / park HL / `inc de`×2 / `ld hl,(de)` (high) / `ex de,hl` / restore low into HL → DEHL. Not a byte-shift chain |
| `*p++ = (BYTE)val; val >>= 8` | `ld (de),a` with A=L; `inc de`; then **logical** byte slide L←H←E←D, D=0 (unsigned). Not `sra hl` |
| Range `c >= 'A' && c <= 'Z'` | `ld a,c` / `cp 'A'` / `jp c` / `cp 'Z'+1` / `jp nc` — 8-bit, unsigned |
| `float /` | restoring divide (`library-math32`), not inv×mul |
| `1.0 / x` | still restoring `/`, not `fsinv` |
| Counted 16-bit loop | `cpu-8085` trip-count identity **or** `dec bc` / `inc b` / `inc c`. **Not** C `n--` → `jp nk` |
| 8-bit counted loop | `dec b` / `jp nz` (opcode `10` is `sra hl`) |
| C `n == 0` / post-dec to zero | A test that **writes Z** |
| `switch` | compare chain (tiny dense enum) or address table via HL; no `jp (ix)` |
| `goto` | `jp` (cost assembler `jr` as 3-byte `jp`) |
| `?:` | compare then two tails; signed test uses K, unsigned uses C |
| `return` 16-bit | HL; never `pop af` for the return address |
| `return` `char` | L; H dead |
| `return` `long` | DEHL |
| `printf` / varargs | `ld a,N` then `call` |

**K is two different flags in two different ops.** After `dec bc` (and
other 16-bit `dec`), K means the pair became **−1**, not 0. After
`sub hl,bc`, K means signed **HL < BC**, and only **immediately** after
that instruction. Unsigned order is **C** (borrow) on that same `sub hl,bc`.
Do not mix them. Do not copy Z80 `dec bc; jp nz`.

**`(de)` stores:** only `ld (de),a` and `ld (de),hl`. There is no
`ld (de),l` as a chip op (the assembler may expand it; do not rely on that
in hot code).

## Recognise these C90 shapes

Rewrite the C to the ISA shape, then allocate. Storage still follows the
C (`static` / file-scope vs automatic) — a hot loop does not justify BSS.

### Arrays, strides, induction

| Shape | C | 8085 |
|-------|---|------|
| Walking **byte** array | `a[k]`; `k += stride` | Cursor **HL** = `&a[k]`, stride **BC**. `ld a,(hl)` / `ld (hl),r` / `add hl,bc`. Bound vs `&a[N]`: unsigned `sub hl,bc` / `jp c` |
| `!byte` as 0/1 | `n -= !a[k]` | `ld a,(hl)` / `or a` / skip `dec` if non-zero. Do not sign-extend `unsigned char` |
| Square induction | `i_sq += i+i+1` | Keep `i_sq` live; `add hl,bc` twice + `inc hl`. Do not recompute `i*i` |
| Walking **word** array | `dst[i] = src[i]` linear | Cursor **DE**, `ld hl,(de)` / `ld (de),hl` / `inc de` twice. Index `i` is not a home if the walk is linear |
| `a[i]` random word | `a[k-i]` | `k-i` in HL; `add hl,hl`; `ld de,a` / `add hl,de` / `ex de,hl` / `ld hl,(de)` |
| Word swap | `t=a[i]; a[i]=a[j]; a[j]=t` | Two addresses: one in DE, the other parked (BC as a **word** addr is wrong; park the second on the stack or rebuild). `ld hl,(de)` / `ex (sp),hl` / `ld (de),hl` |
| 2D `int a[R][C]` | `A[i][j]` | `i*C+j` then `add hl,hl`. `*50` = `*2 + *16 + *32` (`add hl,hl` then `add hl,hl`×4 + orig) |
| `p + k` / `s[k]` | byte pointer + const | `ld de,hl+off` if off is u8, else `add hl,de` |
| `&arr[i]` then fields | `b = &arr[i]; b->x` | Form `b` **once** in DE; `ld de,hl+off` per field. Do not redo `i*sizeof` for every member |
| `*out++ = byte` | packed output | `ld (de),a` / `inc de` |
| 1-based array | `E[1]…E[n]` | Keep the unused `[0]` hole; do not rewrite indexes to 0-based |
| Power-of-two window | `buf[off % 512]` / `off / 512` | `% 512` is `and 0x01FF`. `/ 512` is `>> 9`. Keep the window base in DE |
| Multiply by power-of-two field | `(x-2) * (1<<k) + base` | Shift `x-2` by `k`, then add. Not `l_long_mult` when the scale is 1…128 and a power of two |
| Histogram RMW | `bins[(x >> k) & mask]++` | Index **once**: logical `>>`, `and mask`, `add hl,hl`, add base, **DE** at the cell, `ld hl,(de)` / `inc hl` / `ld (de),hl`. Do not recompute the address for the store |
| Dual-array saxpy / dot | `y[i]=a*x[i]+y[i]`; `s+=x[i]*y[i]` | Two word cursors: **DE** + stack. `short` stride = 2. Mul clobbers parking — reload both cursors. Cast to `unsigned` before mul if the C does |
| 5-point stencil | `out[i][j]=in[i][j]+N+S+E+W` | W compile-time: `i*W` is shift-add (40=`*32+*8`). Keep a **row** word cursor; N/S = ±`W` words (`ld bc,±2*W` / `add hl,bc`); E/W = ±2 bytes. No `l_mult` on the hot path. Double-buffer: swap two pointers (DE + stack) |
| Flattened 2D | `((int *)m)[i]` | Linear word walk after the cast. Param `T m[R][C]` decays to `T (*)[C]`; row stride `C*sizeof(T)` |
| Open-address probe | `idx = (idx + 1) & (N-1)` | N = `1<<k`: `inc hl` / `ld a,l` / `and` low mask / if N>256 also mask H. Do not `l_div` |
| Coprime wrap | `idx += STEP; if (idx >= N) idx -= N` | Unsigned `>=` (DSUB **C**); then `sub hl,bc` with N in BC. Not `% N` |
| djb2 / `h*33+c` | `h = (h<<5)+h+k[i]` | Park h in DE; `add hl,hl`×5; `add hl,de`; add the byte (`ld d,0` / `ld e,c` / `add hl,de`). Mask `& 0xffff` is free on 16-bit |
| 16-bit LCG | `seed = seed * A + C` | `l_mult` or shift-add for A; `add hl,bc`; wrap is free. File-scope seed is BSS |

### Loops and conditions

| Shape | C | 8085 |
|-------|---|------|
| `for (i=lo; i<hi; ++i)` unsigned | `i < SIZE` | Condition is **unsigned** `<` → DSUB **C**, not K. Do not use a signed `jp k` |
| `for (k=n; k>0; k-=s)` | subtract const | Keep `k` in BC/DE; subtract; test **Z** or unsigned `>` via C |
| `for (d=min; d<=max; d+=2)` | step 2 | `inc de` / `inc de` (or `inc l` twice if H is known 0 and no wrap) |
| `for (; n>0; n-=W)` | chunked remainder | `n` in HL; `ld bc,W` / `sub hl,bc`; last iter `if (todo<W) m=todo` |
| `for (i=0; i<n && f; ++i)` | cheap then expensive | Test `i<n` first (Z or C); only then the float/call. Hoist invariants (`limit*limit`) |
| `while (1)` + `break` | infinite + exit | `jp` to header; `jp` out. No `djnz` |
| `if (--i == 0)` | pre-dec to zero | 16-bit `dec` does **not** write Z. `dec hl` then `ld a,h` / `or l` / `jp z`. **Not** `jp k` (K is −1) |
| Assign in predicate | `while (!((k = a[0])==0))` | Load once into the home; test Z; keep `k` |
| `a ? b : -b` / `cond ? x : y` | signed select | `% 2` is `and 1`; negate is `sub hl,hl` / `sub hl,bc` (or `cpl` / `inc` on 8-bit) |
| `max(a,b)` | `a>b?a:b` | signed: `sub hl,bc` / `jp k` take BC |
| `switch (enum)` | small dense 0…n | `ld a,l` / `cp` chain. Larger: table of addresses, `jp (hl)` |
| `goto L` | labelled loop | `jp L`. Counted `j+=1; if (j<n) goto L` is an 8-bit `inc c` / `ld a,c` / `cp n` / `jp c` (unsigned) |
| `do { … } while (n)` fill | counted body | Count in B/C; `dec b` / `jp nz`. Body often a byte store through HL |
| `do { … } while (0)` | statement macro | Not a loop — emit the body once |
| `while (p < end)` pointers | byte/word walk vs sentinel | **Unsigned** `<` on the addresses: DSUB **C**. End in BC, p in HL/DE |
| Nested run-length | inner `while` equal bytes, cap 255 | Outer in-cursor **HL**, out-cursor **DE**, run in **C**. Inner: `ld a,(hl)` / `cp v` / `inc hl` / `inc c` / stop on Z of `inc c` (wrap 255→0) or mismatch |
| Binary search | `mid=(lo+hi)>>1`; `lo=mid+1` / `hi=mid-1` | Non-neg: `add hl,de` / `sra hl`. `lo<=hi` is K **or** Z. Indexed load: `add hl,hl` + table base → DE / `ld hl,(de)`. Masked `tab[mid]&m`: AND in A per byte; do not steal the `hi` home (park `hi` in BC or stack) |
| Insertion shift-up | `while (j>=0 && v[j]>key) v[j+1]=v[j]` | Short-circuit: signed `j<0` (S/K) **before** the load. Word copy: DE at `&v[j]`, `ld hl,(de)` / `inc de`×2 / `ld (de),hl`, then step DE back 4 |
| `while (*p)` / `p-s` | strlen | HL at s; `xor a` / `cp (hl)` / `jp z` done / `inc hl` / loop. Do not `inc` on the NUL. Length = HL − start (`ex de,hl` / start in BC / `sub hl,bc`) |
| `while (*a && *a==*b)` | strcmp | DE and HL; `ld a,(de)` / `cp (hl)` / `jp nz`; `or a` / `jp z` equal; `inc de` / `inc hl`. Return `(unsigned char)*a - (unsigned char)*b` in HL |
| `while ((*d++=*s++))` | strcpy | DE=src, HL=dst; `ld a,(de)` / `ld (hl),a` / `inc de` / `inc hl` / `or a` / `jp nz` |
| Range ladder | `if (c<32)… else if (c<48)…` | Keep the byte in **A**. Successive `cp` / `jp nc` — do not reload. Lexer class: ws / alpha / digit / other as 8-bit unsigned ranges (`'_'` is a `cp`) |
| Clamp / saturate | `if (v>hi) v=hi; if (v<lo) v=lo` | Signed: DSUB then K; assign the bound. Same variable on both arms — one home |
| Side-effect `&&` / `\|\|` | `a < b && probe(c) > d` | Jump over `probe` when `a<b` is false. Flattening to arithmetic is a miscompile |

### Widths, bits, 32-bit

| Shape | C | 8085 |
|-------|---|------|
| `(k+1)>>1` | non-neg k | `inc hl` / `sra hl` |
| `byte <<= 1` then `\|= 1` | pack bits | `add a,a` / `or 1`. Park the acc in **C** across calls |
| **Variable** `byte <<= n` | live count | `n` in B: `add a,a` / `dec b` / `jp nz`. `w%8` is `and 7` when w≥0. Count 0 is a no-op (do not shift by an uninitialised imm) |
| `t ^= t >> c` / `t << c` | 32-bit xorshift | One long in DEHL; `>>` is **logical** (`unsigned long`) via `rra` through A or `l_long_asr_u`. `t & 0x7fff` → HL low 15, D=0 |
| LCG 32 | `(last*A+C)%M` | `static` state stays BSS; `l_long_mult` / `l_long_div_u` (or combined). Calls clobber parking |
| `uint32_t d += uint16 * K` | widen then madd | Zero-extend word to DEHL (`ld de,0`); const× by shift-add or one `l_long_mult`; then **one** divmod if both quot and rem are used |
| `i*2-1` | odd from index | `add hl,hl` / `dec hl` |
| `(uint16_t)long` / `(uint32_t)word` | narrow / widen | Truncate = HL; zero-extend = `ld de,0` |
| 32-bit add/sub | two words | Low: `add hl,bc` (sets C). High: `adc` **through A** on D/E. No `adc hl,de` chip op. Multi-word borrow: `sbc a` through A, not `sub hl,bc` on the high half |
| `12L * N` | long product | Width is **long**. 16-bit `l_mult` is a miscompile if the product can exceed 65535 |
| Packed 12-bit LE | `i += i/2`; odd nibble | `i/2` is `sra hl`. `i & 1` is `ld a,l` / `and 1`. Pack: `<< 4` is `add a,a`×4; 12-bit mask is `ld a,h` / `and 0x0F`. Two window bytes may sit on a **sector/page boundary** — do not assume one `ld hl,(de)` spans them |
| WORD array in a window | `*(WORD *)(win + i*2 % W)` | `i*2` is `add hl,hl`; then `% W` if W is `1<<n`; `ld hl,(de)` |
| DWORD array + mask | `*(DWORD *)p & 0x0FFFFFFF` | LE DWORD load; `ld a,d` / `and 0x0F` / `ld d,a` |
| Nibble insert | `(*p & 0x0F) \| (val << 4)` | Load byte, mask in A, `or`; store `ld (hl),a`. Adjacent byte is a second access |
| Time / bitfield or | `y<<25 \| m<<21 \| d<<16` | Build in DEHL with `add hl,hl`/`rl de`. Do not call `l_long_mult` for `1<<k` |
| UTF-8 / UTF-16 unit | `uc << 6 \| (tb & 0x3F)` | 32-bit shift-or; continuation test `tb & 0xC0` is `and 0xC0` / `cp 0x80` |
| Allocated-bit in `size_t` | `x & (1<<(sizeof(size_t)*8-1))` | MSB of a 16-bit size is **H bit 7**. Test `ld a,h` / `or a` / `jp m` (or `rla` / `jp c`). **No** Z80 `bit 7,h` |
| Overflow guard | `a > SIZE_MAX - b` | Unsigned 16-bit: `ld hl,MAX` / `sub hl,bc` / `jp c` then compare `a` |
| Bit-serial feedback, MSB out | `acc ^= *p++;` then 8× `(acc & msb) ? (acc<<1)^K : (acc<<1)` | 8-bit: acc in **A** (park in C across the pointer inc), `add a,a` / `jp nc` / `xor K`. 16-bit: acc in **HL**, XOR the byte into H, eight× `add hl,hl` / `jp nc` / XOR K through A. Pointer vs `end`: unsigned |
| Bit-serial feedback, LSB out | `acc ^= *p++;` then 8× `(acc & 1) ? (acc>>1)^K : (acc>>1)` | Acc in **DEHL**. XOR the byte into L. Logical `>>1`; if C (old bit 0) XOR K into DEHL. Final `^ ~0UL` is `cpl` on D, E, H, L |
| 16-bit rotate | `(a<<5)\|(a>>11)` | Park in BC; `add hl,hl`×5; OR with logical `B>>3` into L. General: `(v<<n)\|(v>>(16-n))` |
| Boolean mix | `(b&c)\|((~b)&d)` | 16-bit: `cpl` both bytes of b; AND/OR per byte in A. Four live words: extras on the stack |
| Q8.8 mul | `(u16*u16)>>8` → u16 | 16×16→**32** then **byte slide** `>>8` (L←H←E←D, D=0). Not `l_mult`, not `sra hl` |
| Sign-extend `char`→`int` | `int x = sc;` | `ld a,l` / `add a,a` / `sbc a,a` / `ld h,a` |
| Zero-extend `unsigned char`→`int` | `int x = uc;` | `ld h,0`. Promotes to **signed** `int` (the sum can go negative) |
| Sign-extend `int`→`long` | `(long)i` / `(unsigned long)(long)i` | HL as-is; `ld a,h` / `add a,a` / `sbc a,a` / `ld d,a` / `ld e,a` |
| Zero-extend via `(unsigned)` | `(unsigned long)(unsigned)i` | HL truncated; `ld de,0` — not sign-extend |
| Narrowing store | `(signed char)(v>>k)` / `(unsigned char)` | **One** byte `ld (de),a` or `ld (hl),a`. Must not write the neighbour |
| Unsigned bitfield RMW | `r->rate++`; `r->flag=1` | LSB-first in the unit. Extract = shift + mask; insert = `and ~mask` / `or`. Keep the unit live across a burst of field writes |
| `~b` 16-bit | `~b` in a mix | `ld a,l` / `cpl` / `ld l,a`; same for H. 8-bit: `cpl` A |

### Structs, pointers, calls

| Shape | C | 8085 |
|-------|---|------|
| Packed struct | `{char c; double p}` | `p` at offset `sizeof(c)` (no ABI hole). `ld de,hl+1` then float/word load |
| Large-struct field | offset can exceed 255 | Offset += size (no holes). `ld hl,nn` / `add hl,de`, not `ld de,hl+*`. Cursor in DE; `ld hl,(de)` for WORD, two loads for DWORD |
| Recursive tree | `node->left==NULL` | SMALLC push / `call` / caller pop. Word at DE, `ld hl,(de)` / `or`. `2*item` is `add hl,hl`. Locals on **stack** even if other TUs used `-DSTATIC` |
| Intrusive circular list | `n->prev=p; n->next=p->next; p->next->prev=n; p->next=n` | `next`/`prev` are **word** cursors in DE. Four stores. Self-end: `end->next = end`. Do not linear-search an array |
| Nested chase | `p->next->value` | Load `next` into DE first, then field. One live struct pointer at a time |
| Sorted insert | `item <= key` on unsigned ticks | **Unsigned** compare. 16-bit: DSUB **C**/Z. 32-bit: high then low through A. Equality sentinel first if the C tests it |
| Width-1 count / status | `uint8_t n++;` / `int8_t ok` | **8-bit**: `inc c` / `dec c`, not 16-bit `inc hl`. Return **L only; H dead.** Signed 8-bit compare in A (`cp` / `jp m` / `or a`) |
| Owner back-pointer | `void *owner` | Stored pointer (2 bytes). Load DE; do not synthesise `offsetof` unless the source does |
| `* const` pointer param | `T * const p` | Pointer value is const, not the object. Still a stack/DE home |
| `volatile` members | list/queue/lock fields | Reload after every sequence point and after `call`. Parking in BC/DE is a miscompile across a yield or ISR |
| Array of lists | `lists[prio]` | Small unsigned index: `add hl,hl` (pointer scale) + base. Empty walk is 8-bit `dec c` / `jp nz` if the count is 8-bit |
| Circular byte/item queue | `w += size`; wrap at `tail` | Byte cursor HL or DE; add item size. Unsigned `>= tail` via DSUB **C**. Copy payload with `_memcpy` or a counted `ld a,(de)` / `ld (hl),a` / `inc` loop |
| Address-ordered free list | split/coalesce blocks | Word pointers. Compare addresses **unsigned** (C). `size_t` is 16-bit |
| `malloc` / `free` / `memcpy` / `strlen` / `strcpy` / `strcmp` / `qsort` | hosted libc **call** | Bind the preprocessed `PUBLIC`. If the source **writes** the loop (`while (*p)`), emit the byte-walk — do not replace it with a library call |
| Function pointer | `(*fp)(args)` / `qsort(..., cmp)` | Push args then `ld hl,_fn` / call-through-HL (`jp (hl)` / library trampoline). No IX vtable |
| Pointer table dispatch | `ops[k & 7](a, b)` | `k&7`; `add hl,hl` (pointer scale); base + DE; `ld hl,(de)`; push args; `jp (hl)` / trampoline. Same stride as `int ops[]` |
| Opaque callback loop | `acc = f(acc, i)` | **Every** home dies across `call`. Acc, i, n, f on the stack; reload each iter |
| Dense `switch` VM | `op = prog[pc++]; switch(op)` | Table of addresses, `jp (hl)`. Operand stack `stk[++sp]`: word cursor DE, `sp` in BC (`inc bc` / `dec bc` with scale). Parallel arrays `code[pc]`,`a1[pc]`: three bases or one insn stride |
| Stationary `p->a` / `p->b` | many fields, p does not move | **No IX.** Park p in DE (or stack). Each field: `ld hl,off` / `add hl,de` / `ex de,hl` / `ld hl,(de)` — or `ld de,hl+off` from a parked copy of p in HL. Do not reload p from BSS per field |
| Array of structs walk | `s += a[i].x+a[i].y; a[i].z = s` | Element cursor **DE**, stride `sizeof` in **BC**. Fields at +0,+2,+4 via `ld hl,(de)` / `inc de`×2 or `ld de,hl+off`. Step DE by sizeof once per element |
| Singly-linked chase | `while (p) { s+=p->val; p=p->next; }` | p in DE. Load `val` first, load `next` last, `or` for NULL. Write pass: `ld hl,(de)` / `inc hl` / `ld (de),hl` then chase |
| Deep recursion | N-queens / qsort_rec | Automatics **stack-only** (col, row, lo, hi, i). File-scope `board[]` is BSS. `if (d<0) d=-d`: DSUB then K, negate (`sub hl,hl` / `sub hl,bc`). Fnptr compare: spill the partition state |
| Frame-resident array | `int loc[16]; … loc[k]` | Allocate on SP (`ld hl,-n` / `add hl,sp` / `ld sp,hl`). Base `ld de,sp+*`. Dynamic k: `add hl,hl` / `add hl,de` / `ex de,hl` / `ld hl,(de)`. **No** `add hl,ix` |
| Address-taken local | `f(&v)` / `f(&t[i])` | Real stack slot. Reload v and the array after the call. Cannot keep v in BC across `call` |
| `T m[R][C]` parameter | decays to `T (*)[C]` | Row is pointer + `i * C * sizeof(T)`. Inner j walks a row cursor |
| Task / `void (*)(void *)` | push param, call through HL | SMALLC: push `void *`, `jp (hl)`. A yield **clobbers** A F BC DE HL; reload from TCB/stack |
| `*(const T *)p` | comparator load | Arg pointer in DE: `ld hl,(de)` |
| Out-parameter | `void f(..., T *out)` | Address in DE; store DEHL with `ld (de),hl` / high half via `inc de`×2 or a second cursor |
| Array parameter `E[]` | decays to pointer | Pointer, not a copy. 1-based stores through that pointer stay 1-based |
| `enum` / small `Boolean` | `int` on the stack | 2 bytes stacked; values often fit in A for `cp` |
| `register` | storage-class hint | BC/DE only. Ignore if it parks a live value in HL/A |
| Path / class tests | `IsUpper` / `IsDigit` / `c=='/'` | 8-bit `cp` ranges in A |
| Critical section / context save | `di` around a sequence; save regs to a TCB | 8085: `di`/`ei`. Save IFF with **`rim`/`sim`**, not `ld a,i`. Save **AF BC DE HL** and SP via `ld (de),hl` into the TCB. **No** `exx`, IX, IY, `reti`. If existing port asm uses those, keep the C shapes and replace the asm |

### Float kernels

One live float in **DEHL**; every other **automatic** float on the stack.
File-scope / `static` floats stay in BSS/data. Never park an automatic
float in BSS to “free” DEHL.
`sin` / `cos` / `atan` / `log` / `exp` / `sqrt` / `pow` / `fsdiv` clobber
**A F BC DE HL**. Reload cursors after the call. Bind `*_fastcall` when
the prototype says so.

| Shape | C | 8085 |
|-------|---|------|
| Integer then convert | `1.0/(n*(n+1)/2+…)` | Integer triangular first (`n*(n+1)` is even → `sra hl`); convert; restoring `/`. Do not emit `fsinv` for `/` |
| `1.0/sqrt(r)` | unless the source writes `invsqrt` | `sqrt` then restoring `/` |
| Horner / iterate | several live floats | Hoist invariants. Spill the second complex pair; do not pretend `exx` exists |
| `pow(2, integer)` assigned to integer | `1 << n` | `ld hl,1` / `add hl,hl` `n` times (or 32-bit if width is long). Do not call `pow` for that |
| Compare `<=` float | library compare | Flags after the call are the helper’s, not K from DSUB |
| `fabs(a-b) < eps` | tolerance | Library sub; if negative, negate; compare to eps. Do not `==` on IEEE32 vs MBF32 |
| `static T v[N]` inside a function | local with `static` | BSS/data. Automatic `T v[N]` is stack |

### Sequences (use these, not Z80 `(ix+d)` / `exx`)

**Unsigned long `>> 1`**, C ← old bit 0:

```asm
    xor a              ; C = 0 (logical). `or a` on D also works if D is tested
    ld  a,d
    rra
    ld  d,a
    ld  a,e
    rra
    ld  e,a
    ld  a,h
    rra
    ld  h,a
    ld  a,l
    rra
    ld  l,a            ; C = old L bit 0
```

**Shift then conditional XOR** (one bit, after mixing a data byte): `add a,a` or `add hl,hl`, then `jp nc` skip, else XOR a constant through A.

**Q8.8** `(uint16)((uint32)a * b >> 8)`: `l_mult_ulong` or `l_small_mul_32_16x16` into DEHL, then byte slide L←H, H←E, E←D, D←0; result in HL. Hot path may inline that mul.

**Sign-extend L to HL:**

```asm
    ld  a,l
    add a,a
    sbc a,a
    ld  h,a
```

**Stationary struct** (p in DE, field at +4):

```asm
    push de
    ld  hl,4
    add hl,de
    ex  de,hl
    ld  hl,(de)
    pop de             ; p restored
```

If several fields, copy p to the stack once and form each `ld de,hl+off` from a parked HL = p.

**Dense switch** (opcode in A, 0…n): `add a,a` / `ld h,0` / `ld l,a` / add table base / `ld e,(hl)` / `inc hl` / `ld d,(hl)` / `ex de,hl` / `jp (hl)`. Tiny n: `cp` chain.

## Optimise from the ISA

Correct C90 + this ABI first. Then **cycles**, then **bytes**. Timings and
flag side effects: **`cpu-8085`**.

1. **Name the shape**, then emit it (pointer walk, dual cursors, bit-serial
   shift-xor, Q8.8, bitfield RMW, combined divmod, variable shift). Index
   arithmetic is the fallback.
2. **Use the extra ops.** `ld de,sp+n` beats `ld hl,nn`/`add hl,sp` for
   unsigned 8-bit offsets. `sub hl,bc` is the 16-bit subtract, signed
   compare (K), and unsigned compare (C). `rl de` with `add hl,hl` is the
   32-bit shift. `sra hl` is signed `>>`.
3. **HL is the ALU, not a local.** Every deref and most arithmetic destroy
   it. Park first. Word cursor in DE; stride in BC.
4. **Calls kill parking.** Helpers and unknown C functions clobber
   **A F BC DE HL**. Reload from slots. Do not call `l_gint*sp` —
   open-code `ld de,sp+*`.
5. **Inline vs helper.** Hot path: inline a short body. Otherwise `call`
   from **Helpers**. Measure with **`tool-ticks`** (`-m8085` before the
   binary) when unsure.
6. **Do not emit Z80-only forms.** No IX, `(ix+d)`, `exx`, native `djnz`,
   native 2-byte `jr`, `sbc hl,de`. Cost assembler `jr` as `jp` (3 bytes,
   cond 10/7). Opcode `10` is `sra hl`; opcode `18` is `rl de`.
7. **No copt** on this output (hand-written asm). Drop copy-backs yourself
   (`ld a,e` then `ld e,a`; prefer `ex de,hl` / `ld bc,hl` over push/pop
   transfers).
8. **Listing check.** Assemble `-m8085 -l`. If the hot path shows
   `call __z80asm__*`, rewrite to a native or extended op.

### Do not emit

| Form | Why |
|------|-----|
| IX, IY, `(ix+d)` | No index registers |
| `exx`, AF′/BC′/DE′/HL′ | No alternate bank |
| Native `djnz` | Opcode `10` = `sra hl` |
| Native 2-byte `jr` | Opcode `18` = `rl de` |
| `sbc hl,de` / `sbc hl,bc` as a chip op | Not on 8085; may become a helper `call` |
| `sub hl,de` as a chip op | DSUB is **HL−BC only** |
| `adc hl,de` as a chip op | 32-bit carry is `adc a` through the high bytes |
| `ld (de),r` for r ≠ A, or `ld (de),n` | Illegal |
| Word cursor in BC | No `ld hl,(bc)` |
| Z80 `bit n,r` / `ld a,i` / `exx` / IX / IY | Not on 8085 — even if accompanying port asm uses them. IFF is `rim`/`sim`; critical is `di`/`ei` |
| `jp k` after `dec rp` for `== 0` | K means the pair is **−1** |
| `jp k` for unsigned `<` | Unsigned order is **C** after `sub hl,bc` |
| `pop af` as return address or 16-bit temp | F bit 3 hardwired 0 |
| BSS for an automatic, temp, or spill | Static storage only if C wrote `static` or the object is file-scope |
| Inserting `static` / `-DSTATIC` on locals the source left automatic | Would alias frames (recursion) and change the program |
| `-fframe-pointer` | No IX |
| Intel mnemonics | Zilog in this tree |
| `__sdcccall(1)` objects | Not this ABI |
| `l_mult` for 16×16→32 (Q8.8, widening mul) | Product exceeds 16 bits — use a long mul, then shift |
| Eight `add hl,hl` for `<< 8` | Byte move `ld h,l` / `ld l,0` |
| `sra hl` for signed `/ (1<<n)` of a negative | C toward zero needs the bias `(1<<n)-1` first |
| `(iy+d)` / `add iy,de` for `p->field` | No IY. Park p in DE / stack |
| Evaluating both arms of `&&` / `\|\|` | Side effects and checksum probes diverge |

## Workflow

1. Read the C as C90. Note widths, signedness, `static`, attributes, float,
   and every comment.
2. Name the shapes per function (tables above).
3. Lay out objects: `static` keyword and file-scope → BSS/data; every
   automatic → stack. No extra BSS.
4. Plan residency (word cursor DE, stride BC, byte acc C, one long DEHL).
   Write the function header (Comments) from that plan.
5. Lower with extended ops; spill across `call`. Carry and expand comments.
   Blank line after `jp` / `ret` (not after `call`). Inline hot helpers.
6. Assemble `z88dk-z80asm -m8085 -l`. Rewrite helper calls on the hot path.
7. If the source uses TIMER macros, emit `TIMER_START` / `TIMER_STOP` as
   **labels at those source points**, not around CRT.

Comparing this output to another compiler, and feeding lessons back into
**this** skill, is **`methodology-measure`** (with `compiler-80cc` /
`compiler-sccz80` / `tool-ticks` as needed). Do not load those to emit.

## Related

- ISA, flags, timings, stack sequences: `cpu-8085`
- Assemble: `tool-z80asm`
- Float algorithms: `library-math32`, `library-math16` (div = restoring, inv = NR)
- Integer helpers: `libsrc/l/sccz80/8085.lst`, `libsrc/l/util/8085.lst`, `libsrc/math/integer/small/`
- Classic vs newlib: `library-classic`
- Optional quality loop vs other compilers: `methodology-measure`
