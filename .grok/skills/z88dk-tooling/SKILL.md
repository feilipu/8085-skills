---
name: z88dk-tooling
description: >
  How to use z88dk host tools to measure, profile, and diagnose 8085 (and
  related) code in the z88dk tree: z88dk-ticks timing and debugger, hotspots,
  map/nm/disassembly, suite tests, A/B library swaps, and common measurement
  pitfalls. Use when optimising library asm, explaining benchmark deltas,
  verifying regressions, running +test -clib=8085, or the user runs
  /z88dk-tooling.
---

# z88dk tooling for 8085 measurement and diagnosis

This skill covers **host tools** used with z88dk when writing or optimising
8085 (and portable 8080/gbz80) library code. Opcode and coding rules stay in
**opcode-reference** and **extended-usage**; here the focus is *how to prove*
what is slow, what changed, and what is correct.

Assume a built z88dk tree with `bin/` on `PATH` and `ZCCCFG` pointing at
`lib/config` (see env below).

---

## Environment (always set)

```bash
export PATH=/path/to/z88dk/bin:$PATH
export ZCCCFG=/path/to/z88dk/lib/config
```

Rebuild only what you touch (example: classic 8085 runtime object + crt0 lib):

```bash
# after editing libsrc/l/sccz80/7-8085/...
rm -f libsrc/classic/z80_crt0s/obj/8085/l/sccz80/7-8085/.../file.o
rm -f libsrc/classic/z80_crt0s/obj/8085-crt0
make -C libsrc/classic/z80_crt0s obj/8085-crt0
# relink and install crt0
(cd libsrc && TYPE=8085 z88dk-z80asm -d -I"$ZCCCFG/.." -m8085 \
  -DSTANDARDESCAPECHARS -x8085_crt0 @classic/8085.lst)
cp -f libsrc/8085_crt0.lib lib/clibs/
```

Math32 cores live under `libsrc/math/float/math32/`; rebuild via the 8085 lst
and `math32_8085.lib` into `lib/clibs/` the same way.

---

## Tool map (what to reach for)

| Goal | Tools |
|------|--------|
| Wall-clock-free **cycle count** of a timed region | `z88dk-ticks` + map symbols / `TIMER_*` |
| **Where** time goes (PC histogram) | ticks **debugger** + `hotspot on` → file `hotspots` |
| Call-level profile (function enter/leave) | ticks debugger **`profiler`** (needs debug symbols) |
| Confirm a symbol is **actually linked** | `.map` + `z88dk-z80nm` on `.o` / `.lib` |
| See codegen / library expansion | `z88dk-dis`, assembler **`-l` listing**, map file refs |
| Correctness of float/int libraries | `test/suites/math` (`make test_*_8085.bin` etc.) |
| Publishable microbenchmarks | `support/benchmarks/*` + classic `+test` TIMER recipes |
| A/B “did this patch matter?” | Swap one `.asm`, rebuild lib, **same** `zcc` line, compare ticks **and** `cmp` binaries |
| Assembler synthetic expansion | `z88dk-z80asm -m8085 -l` and read the `.lis` opcodes |

Other z88dk host tools that often help in this workflow: **`zcc`** (driver),
**`z88dk-z80asm`**, **`z88dk-sccz80`**, **`z88dk-dis`**, **`z88dk-z80nm`**,
**`z88dk-appmake`**, **`z88dk-lib`**, optional **`z88dk-gdb`** for source-level
debug when the target supports it.

---

## 1. `z88dk-ticks` — cycle-accurate runs

`z88dk-ticks` emulates the CPU and counts T-states. Prefer it over wall clock
for library and benchmark work.

### CPU model

| Flag | Use |
|------|-----|
| (default) | Z80 |
| `-m8085` | 8085 (required for 8085 binaries) |
| `-m8080`, `-mgbz80`, … | Matching classic clibs |

Wrong model → wrong illegal-opcode behaviour and wrong timings.

### Timed region (preferred)

Instrument C with TIMER labels (classic benchmarks already do):

```c
#ifdef TIMER
  #define TIMER_START() intrinsic_label(TIMER_START)
  #define TIMER_STOP()  intrinsic_label(TIMER_STOP)
#endif
```

Compile with map file, then:

```bash
zcc +test -clib=8085 -vn -DSTATIC -DTIMER -D__Z88DK -O2 prog.c -o prog.bin -m -lndos
z88dk-ticks -m8085 prog.bin -x prog.map \
  -start TIMER_START -end TIMER_STOP -counter 999999999999
```

- **`-x map`** resolves symbolic start/end (and helps disassembly).
- **`-counter`** must exceed the expected cycle count or the run aborts early.
- Output is a **single integer**: cycles between start and end.

### Whole-program run

```bash
z88dk-ticks -m8085 prog.bin
# prints suite / printf output, then "Ticks: N"
```

Use for correctness (`printf` / test harness). Prefer TIMER bounds for
performance so CRT and I/O do not dominate.

### Common pitfalls

1. **Missing `-m8085`** on an 8085 binary.
2. **Start/end swapped** or wrong label (`TIMER_END` vs `TIMER_STOP` typos in docs).
3. **Counter too small** → truncated measurement near the counter value.
4. **Comparing different `zcc` lines** (e.g. with/without `--opt-code-speed`) and
   blaming a library edit.
5. **Historical published ticks** vs today’s compiler/lib — always remeasure both
   sides of a patch on the **same** toolchain revision when attributing a delta.

---

## 2. Debugger + **hotspots** (PC histogram)

When a number is bad, do not guess. Record **where** PCs burn cycles.

### Workflow (scriptable)

```bash
# From the directory where you want the hotspots file written:
printf 'hotspot on\nbreak TIMER_STOP\ncont\nquit\n' | \
  z88dk-ticks -m8085 -x prog.map -debug -start TIMER_START prog.bin \
    -counter 999999999999
```

On exit (or quit), ticks writes **`hotspots`** in the **current working
directory**.

Reference pattern (also used for MS Basic profiling discussions in the z88dk
project): enable hotspot, run to a breakpoint, quit; then sort the file.

### `hotspots` file format

Whitespace-separated columns (conceptually):

```text
<hit_count>  <cycle_sum>  <disassembly including [addr]>
```

- Sort by **cycles**: `sort -nrk2 hotspots | head`
- Sort by **hits**: `sort -nrk1 hotspots | head`

### Aggregating to symbols / source

Map file lines with `addr` give symbol bases. Attribute each hotspot PC to the
nearest preceding symbol (binary search on sorted addresses). Then roll up:

- by **symbol** (`l_long_div_0`, `l_lt_hlbc`, `div_loop`, …)
- by **source file** from the map’s file path field
- by **category** (app C vs `l/sccz80` vs `math32`, …)

This is how you discover e.g. “fasta is 40%+ in 32-bit div” vs “fannkuch is 74%
in generated C and never calls `l_long_div_0`”.

### Useful debugger commands (non-exhaustive)

Entered after `-debug` (or interactive debugger):

| Command | Role |
|---------|------|
| `hotspot on` / `off` | PC/cycle histogram |
| `break <addr\|label>` | Breakpoint (`b` alias) |
| `cont` | Continue |
| `step` / `next` / source variants | Single-step |
| `disassemble` | Disasm around PC |
| `registers` | CPU state |
| `print` / `examine` | Values / memory |
| `backtrace` / `frame` | Call stack when symbols allow |
| `profiler` | Function-level profiling (symbol-driven) |
| `trace` | Instruction trace (verbose; use sparingly) |
| `quit` | Exit (flush hotspots if enabled) |

Pipe commands via stdin for unattended runs; keep the working directory intentional
so `hotspots` lands where you expect.

---

## 3. Map files, nm, disassembly

### Map (`-m` on `zcc`)

- Symbol → address (and often module + **source path:line**).
- Section sizes (`__code_*_size`) for size regressions.
- **First question after a “library is slower” claim:** is the suspect symbol
  **present**?  
  `rg 'l_long_div_0' prog.map` — if missing, that routine cannot explain a TIMER
  delta (fannkuch lesson: 16-bit `l_div` only; binary identical with/without
  long-div patches).

### `z88dk-z80nm`

```bash
z88dk-z80nm path/to/file.o
z88dk-z80nm lib/clibs/8085_crt0.lib | rg long_div
```

Confirm public symbols, CPU (`8085`), section sizes, and that the object you
think you rebuilt is the one in the library.

### `z88dk-dis` / assembler listings

```bash
z88dk-z80asm -m8085 -l -m -o/tmp/x.o file.asm   # produces .lis with opcodes
# Inspect synthetics, e.g. ld a,(de+) → 1A 13
```

Use listings to verify post-inc synthetics, `ld de,sp+*`, and that no illegal
Z80 CB ops slipped into an 8085 file.

---

## 4. Correctness gates

### Math suite (`test/suites/math`)

```bash
cd test/suites/math
make test_math32_8085.bin    # IEEE math32 on 8085
make test_mbf32_8085.bin
make test_mbf32_8080.bin
make test_mbf32_gbz80.bin
# each rule builds and runs ticks with the matching -mCPU
```

Expect `N run, N passed, 0 failed` plus a suite tick count. Use after **any**
change to integer long helpers or float cores.

### Small probes

- Minimal C: one `100/7` style long div/mod, print `q`/`r`.
- Wider random long div/mod loops (signed + unsigned) before trusting a div
  rewrite.
- Float: dedicated `<` / `==` cases including ±0 and signs (compare cores).

If a ticks run **never hits `TIMER_STOP`**, suspect infinite loops (classic:
  clobbering the loop counter register that is also used as `B` in `BC`).

---

## 5. Benchmarks (`support/benchmarks`)

Classic TIMER recipes live under each bench’s `z88dk-classic/readme.txt`.
Typical 8085 pattern:

```bash
zcc +test -clib=8085 -vn -DSTATIC -DTIMER -D__Z88DK -O2 … \
  -o bench.bin -m -lndos
z88dk-ticks -m8085 bench.bin -x bench.map \
  -start TIMER_START -end TIMER_STOP -counter 999999999999
```

Parent `readme.txt` holds **CLASSIC Z80 / 8085 SUMMARY** tables; full RESULT
blocks are often duplicated in parent + `z88dk-classic/`.

When publishing a library opt:

1. Remeasure **only configs that previously published** (or document why new).
2. Update **both** summary table and RESULT block (ticks, size, date, note).
3. Report **percentage**: `(old − new) / old × 100` (positive = faster).
4. Do **not** attribute a delta to a symbol the map does not reference.

Long-running benches (e.g. spectral-norm, pi) need large counters and patience;
100% CPU with rising runtime is normal, not a hang, if PC is advancing.

---

## 6. A/B methodology (required for “why is X slower?”)

1. **Hypothesis** — name the function/file you blame.
2. **Link check** — map / nm must show that symbol in the timed binary.
3. **Controlled rebuild** — only that source differs; same `zcc` flags.
4. **Binary identity** — `md5sum a.bin b.bin` or `cmp`; if identical, the patch
   cannot affect TIMER results (fannkuch + `l_long_div_0`).
5. **Ticks** — same `-start`/`-end`/`-m`/`-counter`.
6. **Hotspots** — optional but decisive: roll up cycles by symbol/file.
7. **Historical numbers** — if comparing to an old published RESULT, note year
   and that sccz80/lib may have moved; remeasure both sides on one tree when
   possible.

### Worked pattern (real issues seen in practice)

| Observation | How tools settled it |
|-------------|----------------------|
| Infinite hang after a “faster” div rewrite | Never reached `TIMER_STOP`; counter alone insufficient; debugger break showed loop counter `B` overwritten when loading divisor into `BC` |
| “Long div opt made fannkuch 5% worse” | Map: **no** `l_long_div_*`; A/B binaries **identical**; hotspots: app + `l_div` (16-bit) + `l_lt` |
| “Fasta long-div win” | Map has `l_long_div_0`; A/B ticks 216M → 205M; hotspots concentrated in div_loop / batch |
| 8080 batch div wrong after port | `ld hl,sp+*` is `add hl,sp` → **clobbers C**; save/restore Carry around SP math (8085 `ld de,sp+*` does not) |

---

## 7. Library rebuild cheat-sheet

| Area | Typical path | Install target |
|------|----------------|----------------|
| sccz80 8085 runtime | `libsrc/l/sccz80/7-8085/` | `8085_crt0.lib` → `lib/clibs/` |
| sccz80 8080 / gbz80 | `8-8080/`, `8-gbz80/` | `8080_crt0.lib` / `gbz80_crt0.lib` |
| math32 8085 | `libsrc/math/float/math32/` + `newlibfiles_8085.lst` | `math32_8085.lib` |
| classic tests | `+test -clib=8085` | pulls `test8085_clib` + crt0 + math libs |

After install, **force** recompile of the test/benchmark binary (delete `.bin` /
`.map`) so `zcc` does not reuse stale objects.

---

## 8. What “good” looks like in a report

When finishing an optimisation or regression investigation, state:

1. **Env** — tree path, CPU (`-m8085`), exact `zcc` line.
2. **Link proof** — map lines for the routine under test (or explicit absence).
3. **Numbers** — old/new ticks, % change, size if relevant.
4. **Hotspot top** — top few % by file/symbol if non-obvious.
5. **Correctness** — suite or probe results.
6. **Non-goals** — benches that cannot show the change (no call sites).

---

## Related

- Opcode map / flags: **opcode-reference**
- 8085 coding rules / stack / synthetics: **extended-usage**
- Upstream z88dk: https://github.com/z88dk/z88dk  
- Hotspot discussion (ticks debugger): z88dk issue tooling notes around
  `z88dk-ticks` interactive `hotspot on` usage
- Classic benchmarks: `support/benchmarks/` in the z88dk tree
- Math suite: `test/suites/math/`
