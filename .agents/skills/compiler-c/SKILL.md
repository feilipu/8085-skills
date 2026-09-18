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

Evidence for the shapes: z88dk `support/benchmarks/` C sources (sieve,
fannkuch, fasta, sorting, binary-trees, mandelbrot, n-body, spectral-norm,
pi, whetstone, dhrystone) and z88dk-libraries **FatFs** (`ff/source/ff.c`)
and **FreeRTOS** (`freertos/source/{list,queue,tasks}.c`, `MemMang/heap_4.c`,
`include/sccz80/portmacro.h`).

## C data model on 8085

Default **`char` is signed**. Little-endian. Integer promotions go to
16-bit `int`. Struct members: `offset += size` (no 2-byte ABI holes).
Bitfields: pack by bytes; if the layout is unclear, stop rather than guess.

| Type | Bytes | In registers |
|------|------:|----------------|
| `char` | 1 | **L** on return (H dead); park live bytes in **C**, not A |
| `unsigned char` | 1 | same; zero-test is `or a`, not a signed compare |
| `short` / `int` / `enum` / near pointer | 2 | **HL** as ALU result; park in **BC** or **DE** |
| `long` / `unsigned long` | 4 | **DEHL** (DE high, HL low) |
| `long long` | 8 | Memory + `__i64_acc` / helpers; not a register home |
| `float` / `double` / `double_t` | maths mode | IEEE32 = 4, DEHL; MBF32 = 4; genmath is not IEEE64. If the source has `float` and no library is named, **ask** (TIMER work) or use IEEE32 for a toy |
| `_Float16` | 2 | `library-math16` |

Ingest C90 plus common source sugar: `//` comments, implicit-int `main()`,
`TIMER_*` macros as **labels** at those sites, `register` as a BC/DE hint
only.

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

Refuse: VLA, mixed declarations, designated initializers, nested functions.
`stdint.h` only if the source already uses it (`uint16_t` / `uint32_t` in
pi). Do not mix newlib `FILE*` cores into a classic 8085 image
(`library-classic`). Do not rewrite 1-based arrays (whetstone `E1[1]…[4]`)
to 0-based.

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
`/` and `%` of the same operands: one helper (pi `ldiv` / `res.quot` +
`res.rem`), not two divides.

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
4. 8-bit live values in **C** (or B if C is taken). A is the 8-bit ALU.
   Mandelbrot `byte_acc` lives in C while the float kernel uses DEHL/A.
5. `volatile`: memory at every sequence point — no BC/DE home that skips
   a reload or store.
6. Do not park a 16-bit value in **AF**. `pop af` forces F bit 3 to 0
   (`$FFFF` → `$FFF7`); never use AF for a return address.

Hot function: `; residency: i in BC; p in DE; x at sp+4`.

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

```asm
; void f(int a, int b)   /* SMALLC: push a, then b; caller cleans */
; entry: [sp+0]=ret, [sp+2]=b, [sp+4]=a
; locals: push, or ld hl,-n / add hl,sp / ld sp,hl
; exit: restore SP; ret
```

## C → 8085 primitives

| C | Emit |
|---|------|
| Automatic (no `static`) | Stack slot; cursor in DE. **Not** BSS |
| `static` keyword / file-scope | BSS/data as declared. Do not invent extra BSS |
| `if (x)` / `while (x)` (16) | Test that **writes Z** (`ld a,h` / `or l`) — leftover K is not a truth test |
| `if (p)` pointer / `== NULL` | `ld a,h` / `or l` |
| `if (c)` byte | `ld a,c` (or `(hl)`) / `or a` |
| `a && b` / `a \|\| b` | Short-circuit; each arm writes Z. Cheap int test before a float/call |
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
| `s << n` n const 2…8 | repeated `add hl,hl`; larger: loop `add hl,hl` with count in B (`dec b` / `jp nz`, not `djnz`) |
| signed `s >> 1` | `sra hl` (**Z unchanged**) |
| unsigned `u >> 1` | `sra hl`; force H bit 7 clear (**Z unchanged**) |
| `long << 1` | `add hl,hl` / `rl de` |
| `unsigned long >> 1` | `rra` through A across DEHL — not `sra hl` on both halves |
| `int * int` | shift-add for small constants; else `call l_mult` (HL = DE×HL) |
| `x * 2` / `* 3` / `* 5` | `add hl,hl`; `ld de,hl` / `add hl,hl` / `add hl,de`; `*4`+orig |
| signed `/` `%` | `call l_div` (HL = DE/HL, DE = remainder) unless power-of-two |
| unsigned `/` `%` | `call l_div_u` |
| both `/` and `%` of same ops | **one** helper; take quot and rem |
| `x % 2` | `ld a,l` / `and 1` — not `l_div` |
| `x % (1<<n)` unsigned | `and (1<<n)-1` (sector 512 → `and 0x01FF`) |
| `x / (1<<n)` unsigned | n logical `>>` (sra + clear high bits, or drop bytes) |
| `x / 2` (non-neg) | `sra hl` |
| LE `*(WORD *)p` / `ld_16` | pointer in DE: `ld hl,(de)` — 8085 is little-endian, unaligned is legal |
| LE `*(DWORD *)p` / `ld_32` | `ld hl,(de)` (low) / park HL / `inc de`×2 / `ld hl,(de)` (high) / `ex de,hl` / restore low into HL → DEHL. Not a byte-shift chain |
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

Rewrite the C to the ISA shape, then allocate. “Seen in” names the
source that makes the shape hot. Storage still follows the C (`static`
/ file-scope vs automatic) — a hot loop does not justify BSS.

### Arrays, strides, induction

| Shape | C | 8085 |
|-------|---|------|
| Walking **byte** array | `flags[k]`; `k += i` (sieve) | Cursor **HL** = `&flags[k]`, stride **BC** = `i`. `ld a,(hl)` / `ld (hl),1` / `add hl,bc`. Bound: `&flags[SIZE]` vs HL with **unsigned** `sub hl,bc` / `jp c` |
| `!byte` as 0/1 | `count -= !flags[k]` (sieve) | `ld a,(hl)` / `or a` / skip `dec` of count if non-zero. Do not sign-extend `unsigned char` |
| Square induction | `i_sq += i+i+1` (sieve) | Keep `i_sq` live; `add hl,bc` twice + `inc hl`. Do not recompute `i*i` |
| Walking **word** array | `perm[i] = perm1[i]` (fannkuch) | Cursor **DE**, `ld hl,(de)` / `ld (de),hl` / `inc de` twice. Index `i` is not a home if the walk is linear |
| `a[i]` random word | `perm[k-i]` (fannkuch) | `k-i` in HL; `add hl,hl`; `ld de,perm` / `add hl,de` / `ex de,hl` / `ld hl,(de)` |
| Word swap | `t=a[i]; a[i]=a[j]; a[j]=t` (fannkuch) | Two addresses: one in DE, the other parked (BC as **byte** addr is wrong; park the second addr on the stack or rebuild). `ld hl,(de)` / `ex (sp),hl` / `ld (de),hl` |
| 2D `int a[R][C]` | `A[i][j]` (dhrystone) | `i*C+j` then `add hl,hl`. `*50` = `add hl,hl`×1 + `add hl,hl`×4 (`*2 + *16 + *32`) |
| `p + k` / `s[k]` | `ss+k`, `s+k` (fasta) | byte: `ld de,hl+off` if off is u8, else `add hl,de` |
| `&arr[i]` then fields | `b = &bodies[i]; b->x` (n-body) | Form `b` **once** in DE; `ld de,hl+off` per field. Do not redo `i*sizeof` for every member |
| `*out++ = byte` | mandelbrot `PUTC` | `ld (de),a` / `inc de` |
| 1-based array | whetstone `E1[1]…E1[4]` | Keep the unused `[0]` hole; index is already FORTRAN-shaped |

### Loops and conditions

| Shape | C | 8085 |
|-------|---|------|
| `for (i=lo; i<hi; ++i)` unsigned | sieve `i_sq < SIZE` | Condition is **unsigned** `<` → DSUB **C**, not K. Do not use a signed `jp k` |
| `for (k=n; k>0; k-=s)` | pi `k -= 14` | Keep `k` in BC/DE; subtract const; test **Z** or unsigned `>` via C |
| `for (d=min; d<=max; d+=2)` | trees `depth += 2` | `inc de` / `inc de` (or `inc l` twice if H is known 0 and no wrap) |
| `for (; n>0; n-=W)` | fasta line chunks | `n` in HL; `ld bc,W` / `sub hl,bc`; remainder in last iter (`if (todo<W) m=todo`) |
| `for (i=0; i<n && f; ++i)` | mandelbrot inner | Test cheap `i<n` first (Z or C); only then the float compare. Hoist `limit*limit` |
| `while (1)` + `break` | fannkuch, pi | `jp` to header; `jp` out. No `djnz` |
| `if (--i == 0)` | pi | 16-bit `dec` does **not** write Z. `dec hl` then `ld a,h` / `or l` / `jp z`. **Not** `jp k` (K is −1) |
| Assign in predicate | `while (!((k = perm[0])==0))` | Load once into the home; test Z; keep `k` |
| `a ? b : -b` / `cond ? x : y` | fannkuch checksum | `% 2` is `and 1`; negate is `sub hl,hl` / `sub hl,bc` (or `xor 255` / `inc` on 8-bit) |
| `max(a,b)` | fannkuch `a>b?a:b` | signed: `sub hl,bc` / `jp k` take BC |
| `switch (enum)` | dhrystone `Proc_6` | values 0…4: `ld a,l` / `cp` chain. Larger: table of addresses, `jp (hl)` |
| `goto L` | whetstone `IILOOP` / `L10` | `jp L`. Counted `J+=1; if (J<6) goto L10` is an 8-bit `inc c` / `ld a,c` / `cp 6` / `jp c` (unsigned) or `cp` / `jp nz` |

### Widths, bits, 32-bit

| Shape | C | 8085 |
|-------|---|------|
| `(k+1)>>1` | fannkuch `k2` | `inc hl` / `sra hl` (k is non-negative) |
| `byte <<= 1` then `\|= 1` | mandelbrot pack | `add a,a` / `or 1`. Park the acc in **C** across calls |
| **Variable** `byte <<= n` | mandelbrot leftover `(8-w%8)` | `n` in B: `add a,a` / `dec b` / `jp nz`. `w%8` is `and 7` when w≥0. Count 0 must be a no-op (do not shift by an uninitialised imm) |
| `t ^= t >> c` / `t << c` | sorting xorshift 32 | One long in DEHL; `>>` is **logical** (`unsigned long`) via `rra` through A or `l_long_asr_u`. `t & 0x7fff` → HL low 15, D=0 |
| LCG 32 | fasta `(last*IA+IC)%IM` | `static long last` in BSS; `l_long_mult` / `l_long_div_u` (or combined). Calls clobber parking |
| `uint32_t d += uint16 * 10000` | pi | Zero-extend word to DEHL (`ld de,0`); const× 10000 by shift-add or one `l_long_mult`; then **one** divmod by `b` |
| `i*2-1` | pi `b` | `add hl,hl` / `dec hl` |
| `(uint16_t)long` / `(uint32_t)word` | pi | Truncate = HL; zero-extend = `ld de,0` |
| 32-bit add/sub | fasta / pi | Low: `add hl,bc` (sets C). High: `adc` **through A** on D/E. No `adc hl,de` chip op. Multi-word borrow: `sbc a` through A, not `sub hl,bc` on the high half |
| `12L * LOOP` | whetstone | Width is **long**. 16-bit `l_mult` is a miscompile if the product can exceed 65535 |

### Structs, pointers, calls

| Shape | C | 8085 |
|-------|---|------|
| Packed struct | fasta `{char c; double p}` | `p` at offset `sizeof(c)` (no ABI hole). `ld de,hl+1` then float load |
| Recursive tree | `ItemCheck` / `BottomUpTree` | SMALLC push / `call` / caller pop. `tree->left==NULL`: word at DE, `ld hl,(de)` / `or`. `2*item` is `add hl,hl`. Locals on **stack** even if other TUs used `-DSTATIC` |
| `malloc` / `free` / `memcpy` / `strlen` / `strcpy` / `strcmp` / `qsort` | trees, fasta, sorting, dhrystone | Bind the preprocessed `PUBLIC`. Do not open-code unless the count is tiny and in the TIMER region |
| Function pointer | `qsort(..., cmp)` | Push args then `ld hl,_ascending_order` / call-through-HL (`jp (hl)` / library trampoline). No IX vtable |
| `*(const int *)p` | sorting `cmp` | Arg pointer in DE: `ld hl,(de)` |
| Out-parameter | whetstone `P3(..., double *Z)` | Address in DE; store DEHL with `ld (de),hl` / high half via `inc de`×2 or a second cursor |
| Array parameter `E[]` | whetstone `PA` | Pointer, not a copy. 1-based stores through that pointer |
| `enum` / `Boolean` | dhrystone | `int` (2 bytes) on the stack; values often fit in A for `cp` |
| `register` | dhrystone `-DREG` | Hint: BC/DE only. Ignore if it parks a live value in HL/A |

### Float kernels

One live float in **DEHL**; every other **automatic** float on the stack.
File-scope / `static` floats stay in BSS/data. Never park an automatic
float in BSS to “free” DEHL.
`sin` / `cos` / `atan` / `log` / `exp` / `sqrt` / `pow` / `fsdiv` clobber
**A F BC DE HL**. Reload cursors after the call. Bind `*_fastcall` when
the prototype says so.

| Shape | C | 8085 |
|-------|---|------|
| `1.0/((i+j)*(i+j+1)/2+i+1)` | spectral `eval_A` | Integer triangular first (`n*(n+1)` is even → `sra hl`); convert; restoring `/`. Do not emit `fsinv` for `/` |
| `1.0/sqrt(r)` | n-body | `sqrt` then restoring `/` unless the source writes `invsqrt` |
| Horner / iterate | mandelbrot `Zr,Zi,Tr,Ti` | Hoist `limit*limit`. Spill the second complex pair; do not pretend `exx` exists |
| `pow(2, integer)` assigned to `long` | trees `iterations` | `ld hl,1` / shift `add hl,hl` `n` times (or 32-bit if width is long). Do not call `pow` for that |
| Compare `<=` float | mandelbrot | Library compare; flags after the call are the helper’s, not K from DSUB |

### FatFs (`ff/source`)

Typedefs on this 8085 C model: `BYTE` 1, `WORD`/`WCHAR`/`UINT` 2, `DWORD`/`LBA_t`/`FSIZE_t` 4. `QWORD` is not a register home.

Under `__Z88DK`, `ld_16`/`ld_32`/`st_16`/`st_32` are `*(WORD*)` / `*(DWORD*)`. Emit native LE word traffic, not the portable byte-shift bodies.

| Shape | C | 8085 |
|-------|---|------|
| Sector window | `fs->win[bc % SS]` SS=512 | Keep `win` in DE. `% 512` is `and 0x01FF`. `/ 512` is `>> 9`. Index can make `fs+off` **> 255** — `ld hl,nn` / `add hl,de`, not `ld de,hl+*` |
| Large-struct field | `fs->database`, `fp->obj` | Offset += size (no holes). If off>255, 16-bit add. Cursor in DE; `ld hl,(de)` for WORD, two loads for DWORD |
| `clst2sect` | `(clst-2) * csize + database` | `csize` is a power of two (1…128). `clst-2` in DEHL; **shift** by log2(csize), then 32-bit add `database`. Not `l_long_mult` |
| FAT12 entry | `bc += bc/2`; odd nibble (ff `get_fat`/`put_fat`) | `bc/2` is `sra hl`. `clst & 1` is `ld a,l` / `and 1`. Pack: `<< 4` is `add a,a`×4; `wc & 0xFFF` is `ld a,h` / `and 0x0F`. Two window bytes may sit on a **sector boundary** — do not assume one `ld hl,(de)` spans them |
| FAT16 array | `ld_16(win + clst*2 % SS)` | `clst*2` is `add hl,hl`; then `% 512` / `ld hl,(de)` |
| FAT32 array | `ld_32(...) & 0x0FFFFFFF` | LE DWORD load; `ld a,d` / `and 0x0F` / `ld d,a` |
| Nibble insert | `(*p & 0x0F) \| (val << 4)` | Load byte, mask in A, `or`; store `ld (hl),a`. Adjacent byte is a second access |
| Time bitfield | `year<<25 \| mon<<21 \| day<<16` | Build in DEHL with `add hl,hl`/`rl de`. Do not call `l_long_mult` for `1<<k` |
| UTF-8 / UTF-16 | `uc << 6 \| (tb & 0x3F)` (ff `tchar2uni`) | 32-bit shift-or; continuation test `tb & 0xC0` is `and 0xC0` / `cp 0x80` |
| Path byte class | `IsUpper` / `IsDigit` / `IsSeparator` | 8-bit `cp` ranges in A. `TCHAR` is `char` unless LFN Unicode is on |
| `do { … } while (n)` fills | directory/LFN loops | Count in B/C; `dec b` / `jp nz`. Body often a byte store through HL |
| `FRESULT` / `fs_type` switch | `get_fat` | Small dense enum: `ld a,l` / `cp` chain |
| `volatile BYTE` lock | `SysLock` | Memory at every sequence point; 8-bit in C only between points |

### FreeRTOS (`freertos/source`)

This pack’s `portmacro.h` is a **Z80** port (`exx`, `ix`, `iy`, `ld a,i`). When **emitting 8085**, keep the C shapes; replace that asm (see do-not-emit).

On the sccz80 port types: `BaseType_t` = **`int8_t`**, `UBaseType_t` = **`uint8_t`**, `StackType_t` = `uint16_t`, `TickType_t` = 16 or 32, `portBYTE_ALIGNMENT` = 1.

| Shape | C | 8085 |
|-------|---|------|
| Intrusive circular list | `list.c` insert/remove | `pxNext`/`pxPrevious` are **word** cursors in DE. Four stores: `n->prev=p; n->next=p->next; p->next->prev=n; p->next=n`. Self-end: `end->next = end`. Do not linear-search an array |
| Nested chase | `pxIterator->pxNext->xItemValue` | Load `pxNext` into DE first, then field. One live struct pointer at a time |
| Sorted insert | `xItemValue <= xValueOfInsertion` | `TickType_t` **unsigned**. 16-bit: DSUB **C**/Z. 32-bit: high then low through A. `portMAX_DELAY` is a sentinel — test equality first (as the C does) |
| List count | `uxNumberOfItems` (`UBaseType_t`) | **8-bit**: `inc c` / `dec c`, not 16-bit `inc hl`. Return width-1 in L |
| `pdTRUE` / `BaseType_t` | API returns | **L only; H dead.** Signed 8-bit compare in A (`cp` / `jp m` / `or a`) |
| Owner back-pointer | `pvOwner` / `pxContainer` | Stored `void *` (2 bytes). Load DE; do not synthesise `offsetof` unless the source does |
| `* const` pointer param | `List_t * const pxList` | Pointer value is const, not the object. Still a stack/DE home |
| `configLIST_VOLATILE` | list members | If defined `volatile`: reload after every sequence point and after `call`. Parking in BC/DE is a miscompile across a yield |
| Ready-list array | `pxReadyTasksLists[uxPriority]` | `UBaseType_t` index: `add hl,hl` (pointer scale) + base. Empty walk is 8-bit `dec c` / `jp nz` |
| Circular item queue | `pcWriteTo += uxItemSize`; wrap `pcHead` | Byte cursor HL or DE; add item size (often 1, 2, or small const). Unsigned `>= pcTail` via DSUB **C**. Copy payload with `_memcpy` or a counted `ld a,(de)` / `ld (hl),a` / `inc` loop |
| Queue locks | `volatile int8_t cRxLock` | 8-bit; `++` saturates. Touch only inside a critical section |
| Heap free list | `heap_4` `BlockLink_t` | Address-ordered **word** pointers. Split/coalesce: compare addresses unsigned (C). `size_t` is 16-bit |
| Allocated-bit in size | `xBlockSize & (1<<(sizeof(size_t)*8-1))` | MSB of a 16-bit size is **H bit 7**. Test `ld a,h` / `or a` / `jp m` (or `rla` / `jp c`). **No** Z80 `bit 7,h` |
| Overflow guards | `a > SIZE_MAX - b` | Unsigned 16-bit: `ld hl,MAX` / `sub hl,bc` / `jp c` then compare `a` |
| Task function | `void (*)(void *)` | SMALLC: push `pvParameters`, `call` through HL (`jp (hl)`). Yield **clobbers** A F BC DE HL; reload from TCB/stack |
| `do { … } while (0)` | critical/lock macros | Not a loop — emit the body once |
| Critical section / yield asm | `portENTER_CRITICAL`, `portSAVE_CONTEXT` | 8085: `di`/`ei`. Save IFF with **`rim`/`sim`**, not `ld a,i`. Save **AF BC DE HL** and SP via `ld (de),hl` into the TCB. **No** `exx`, IX, IY, `reti` unless the CPU is not 8085 |

## Optimise from the ISA

Correct C90 + this ABI first. Then **cycles**, then **bytes**. Timings and
flag side effects: **`cpu-8085`**.

1. **Name the shape**, then emit it (pointer walk, induction, combined
   divmod, variable shift). Index arithmetic is the fallback.
2. **Use the extra ops.** `ld de,sp+n` beats `ld hl,nn`/`add hl,sp` for
   unsigned 8-bit offsets. `sub hl,bc` is the 16-bit subtract, signed
   compare (K), and unsigned compare (C). `rl de` with `add hl,hl` is the
   32-bit shift. `sra hl` is signed `>>`.
3. **HL is the ALU, not a local.** Every deref and most arithmetic destroy
   it. Park first. Word cursor in DE; stride in BC.
4. **Calls kill parking.** `l_mult`, `l_div`, `l_div_u`, `l_long_*`, float
   helpers, and unknown C functions clobber **A F BC DE HL**. Reload from
   slots. Do not call `l_gint*sp` — open-code `ld de,sp+*`.
5. **Inline vs helper.** A short shift-add for `* 10` in a hot loop beats
   `l_mult` (call + full clobber). A one-shot multiply can call `l_mult`.
   Measure with **`tool-ticks`** (`-m8085` before the binary) when unsure.
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
| Z80 `bit n,r` / `ld a,i` / `exx` / IX / IY in FreeRTOS port asm | Not on 8085; `rim`/`sim` for IFF; `di`/`ei` for critical |
| `jp k` after `dec rp` for `== 0` | K means the pair is **−1** |
| `jp k` for unsigned `<` | Unsigned order is **C** after `sub hl,bc` |
| `pop af` as return address or 16-bit temp | F bit 3 hardwired 0 |
| BSS for an automatic, temp, or spill | Static storage only if C wrote `static` or the object is file-scope |
| Inserting `static` / `-DSTATIC` on locals the source left automatic | Would alias frames (recursion) and change the program |
| `-fframe-pointer` | No IX |
| Intel mnemonics | Zilog in this tree |
| `__sdcccall(1)` objects | Not this ABI |

## Workflow

1. Read the C as C90. Note widths, signedness, `static`, attributes, float.
2. Name the shapes per function (tables above).
3. Lay out objects: `static` keyword and file-scope → BSS/data; every
   automatic → stack. No extra BSS.
4. Plan residency (word cursor DE, stride BC, byte acc C, one long DEHL).
5. Lower with extended ops; spill across `call`.
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
- Classic vs newlib: `library-classic`
- Optional quality loop vs other compilers: `methodology-measure`
