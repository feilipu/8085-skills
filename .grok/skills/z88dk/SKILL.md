---
name: z88dk
description: >
  z88dk newlib vs classic architecture for targets, CRT m4 driver instantiation,
  serial/character FILE* wiring, hybrid 8085 consoles, cooked line input
  (getline vs fgets_cons), static stdio heap sizing, disk fcntl (asm_target_open,
  open_max, FCB vs FatFs), mixed multi-CPU trees, target_io / ticks verification,
  and library source layout (one major function per file under libsrc). Use when
  migrating targets, adding serial or disk drivers, writing or reorganising
  library asm/C under libsrc (math, sccz80, target), debugging fopen/open or
  console-after-fopen, dual-stack file I/O, CP/M stdio devices, dual-CPU
  firmware shells, or reviewing CRT/config lists. Complements z88dk-tooling
  (measure) and extended-usage (8085 codegen).
---

# z88dk target I/O architecture (newlib + classic)

General lessons from migrating hardware targets and enabling newlib CP/M file
I/O. **Measurement** stays in **[z88dk-tooling](../z88dk-tooling/SKILL.md)**.
**8085 codegen** stays in **extended-usage** / **opcode-reference**.

Assume a built tree with `bin/` on `PATH` and `ZCCCFG` → `lib/config`.

---

## 0. Library source layout — one major function per file

House style across z88dk **`libsrc/`** (classic and newlib): **one major function
(or one logical operation) per source file**. Do not invent a different layout
when adding or splitting library code.

### What counts as “one major function”

| In one file | Separate files |
|-------------|----------------|
| The operation’s public entry (or entries) | Unrelated operations (e.g. mul vs add) |
| **Callee** and non-callee / sccz80–sdcc glue for that op when they live with the core | Different ops “to save files” |
| f16 + f24 (or similar) variants of the **same** op | Pack/expand family may share a conversion file if that is the existing pattern for that library |
| Helpers that belong only to that op (local or `PUBLIC` if other modules call them) | Helpers that are really a second feature → own file |

Examples (math16-style, but the rule is general):

- `asm_f16_mul.asm` — half mul callee, f24 mul, integer mulu helper  
- `asm_f16_add.asm` — add/sub callee + f24 add  
- `asm_f16_compare.asm` — compare + compare_callee  

### Agent rules

1. **Match neighbours** in the same directory: filename ≈ symbol / operation name.
2. **Do not** merge unrelated ops into one `.asm`/`.c` to share a few lines.
3. **Do not** split a single op across files just to isolate a 10-line helper unless the tree already does that.
4. CPU-specific copies (`asm/z80/`, `asm/8085/`, `l/sccz80/7-8085/`, …) keep the **same one-op-per-file map**; implementations may differ by ISA, not by inventing a parallel file taxonomy.
5. List files (`.lst`) and products reference modules by path — one op per file keeps nm/map/hotspot attribution sane.

This is an **implicit z88dk library requirement**, not a math16-only note. Apply it to sccz80 runtime, float cores, target drivers, and new work under `libsrc/`.

---

## 1. Two library worlds (do not blur)

| World | Home | CPUs | Typical product |
|-------|------|------|-----------------|
| **Classic** | `libsrc/target/<name>/` (historic fcntl/stdio/gfx) + `libsrc/classic/` | Z80, IXIY, Z180, **8080**, **8085**, gbz80, … | `*_clib.lib`, `cpm8085_clib.lib`, … |
| **Newlib** | Was `libsrc/newlib/target/<name>/`; migrated → `libsrc/target/<name>/` beside classic | **Z80-class** (`-clib=new` / `sdcc_ix` / `sdcc_iy`) | `lib/clibs/{sccz80,sdcc_ix}/<target>.lib` |

**Hard rules**

1. Do **not** merge classic and newlib **stdio cores** or **fcntl `open` owners** in one link without a designed bridge.
2. Prefer sharing **device / thin driver** layers, not cores.
3. **CLIB** selects newlib; machine **SUBTYPE**s usually stay classic-owned (esp. CP/M’s 150+ machines).
4. After a path move, put the target name in `MIGRATED_TARGETS` in `libsrc/newlib/Makefile` so builds use `../target/<name>`.

**Mixed tree hazard:** when newlib Z80+ sources land under `libsrc/target/cpm/` next to classic multi-CPU code, isolation is **list ownership** (`*.lst`), never broad globs. Classic 8080/8085 images must never pull newlib objects (Z80-only opcodes / calling conventions).

---

## 2. Serial / character FILE* instantiation (newlib)

### Layers

```text
stdio (printf / FILE*)
    → console_01  (line-edited terminals; default on many CRTs)
    → character_00 (thin byte streams; good for multi-port + RDR/PUN/LST)
         → device (UART/SIO/ACIA/ASCI/HBIOS/BDOS CON…)
```

| Layer | Typical path |
|-------|----------------|
| stdio core | `libsrc/newlib/stdio` |
| character_00 | `libsrc/newlib/drivers/character/` |
| console_01 | `libsrc/newlib/drivers/terminal/console_01/` |
| Target terminals | `libsrc/target/<t>/driver/terminal/*.m4` + `.asm` |
| Devices | `libsrc/target/<t>/device/…` |

**Defaults:** most hardware CRTs still instantiate full **console_01** / `rc_01_*` terminals. Thin **character_00** is the additive multi-port pattern (keep `m4_file_dup` / optional `crt_driver_instantiation.asm.m4`).

### CRT m4 wiring (where FILEs are born)

Startup `*_crt_N.asm.m4` includes driver m4 macros inside:

```text
clib_instantiate_begin.m4
  m4_<driver>(_stdin, …)
  m4_<driver>(_stdout, …)
  m4_file_dup(_stderr, 0x80, __i_fcntl_fdstruct_1)   ; often dup of stdout
  … extra ports …
clib_instantiate_end.m4   ; builds fdtbl + FILE freelist + stdio heap
```

Each static driver m4 typically:

1. Allocates a **FILE** + **FDSTRUCT** on the **stdio heap** sections.
2. Pushes an entry into the **fd table** body.
3. Chains a heap block header (`__i_fcntl_heap_N`).

### Multi-port and dups

| Pattern | Mechanism |
|---------|-----------|
| Second console / teletype | Second input+output terminal pair → **`ttyin` / `ttyout`**; **`ttyerr`** = `m4_file_dup` of `ttyout` (same idea as stderr) |
| stderr | Almost always a **dup** of stdout’s FDSTRUCT (flag `0x80`) |
| Extra static slots | `m4_file_absent` or more drivers |

Declared in newlib `stdio.h` even when a CRT does not instantiate them: `stdrdr`, `stdpun`, `stdlst`, `ttyin`, `ttyout`, `ttyerr`. **Missing instantiation ≠ missing declaration** — apps that reference an uninstantiated `FILE*` will fail at link or runtime.

### CP/M character model (portable apps vs implementations)

**Physical BDOS units** (what drivers usually call):

| Unit | BDOS | Typical newlib FILE* |
|------|------|----------------------|
| CON | 1/2/6/… | `stdin` / `stdout` / `stderr` |
| RDR | 3 | `stdrdr` |
| PUN | 4 | `stdpun` |
| LST | 5 | `stdlst` |

**Logical names** (CRT, TTY, LPT, PTR, PTP, BAT, U\*) are selected by **IOBYTE** (page-0 `$0003`, BDOS 7/8). BIOS maps logical → physical UART/device.

| Audience | What to wire |
|----------|----------------|
| **CP/M implementation** (e.g. CP/M-IDE) | CRT maps `stdin`/`tty*` to real ports; shell seeds IOBYTE; ASM BIOS interprets IOBYTE |
| **CP/M application** (`+cpm -clib=new`) | BDOS only; FILE* + optional logical-name helpers; no UART registers |

Do not assume `fopen("TTY:")` is how implementations work — **FILE* selection + IOBYTE seed** is the real dual-port pattern.

### Hybrid classic+newlib consoles (rc2014-8085 lesson)

When a CRT mixes **classic** `fgetc_cons`/`fputc_cons` with newlib-style startup:

- FILE init flags must match classic expectations (**`18` / `20`** = `_IOSYSTEM|_IOREAD` / `_IOWRITE`).
- Wrong flags (`19`/`21` with spurious `_IOUNGETC`) made first `getchar` return NUL.
- Hybrid clib lists must **not** pull full newlib fcntl/stdio/threads.
- Build: classic `<stdio.h>` must win include order (`-I…/include` **before** `_DEVELOPMENT/common`) when the hybrid needs classic `stdin`/`stdout` objects.

### Cooked line input: newlib vs classic (general)

| World | Line API | Who echoes / edits |
|-------|----------|--------------------|
| **Newlib** | POSIX **`getline` / `getdelim`** | **console_01** (line mode, echo, BS, CR/LF cook) via tied oterm |
| **Classic** | **No `getline`** | **`fgets` on stdin → `fgets_cons`** (echo, DEL, optional soft cursor) |
| **Classic raw** | `fgetc` / `fgetc_cons` | **No** line editor — app must implement if needed |

**Rules of thumb**

1. **`getline` is newlib-only.** Never expect it on 8080/8085 classic products.
2. **One cook layer only.** If the driver/`fgets_cons` already echoes, do **not** also echo in app code (double echo).
3. Hybrid CRTs that only bind `fgetc_cons`/`fputc_cons` are **raw**. App-level line readers (e.g. shell `ya_getline`) are compensating for classic, not for the CPU.
4. Prefer **`fgets` / `fgets_cons`** on classic instead of reimplementing line edit. On serial targets, disable soft cursor if needed (`CLIB_DISABLE_FGETS_CURSOR=1` — already set for `rc2014-8085`).
5. Align dual-CPU apps (Z80 newlib + 8085 classic) at a **single call site** with `#ifdef`, not by linking newlib stdio into 8085 images.

### Dual-port FILE* vs classic `ttyin` macros

| | Newlib CRT | Classic hybrid (e.g. `uart85`) |
|--|------------|--------------------------------|
| Second port | Real drivers: `m4_rc_01_input_uartb(_ttyin, …)` etc. | Often **only** stdin/out/err → primary UART/ACIA |
| `ttyin` / `ttyout` in headers | `extern FILE *` | Classic macros → **`_sgoioblk[3]`…** slots |
| Meaning | Instantiated streams | **Declaration/slots ≠ working UARTB console** |

`fgetc` on classic special-cases **stdin** → `fgetc_cons`. Assigning `input = ttyin` does **not** create a second cooked port unless the CRT initialises that slot and a driver path exists. For dual-port on hybrid: either an **active-console** global in `fgetc_cons`, or real second-stream CRT work — do not copy newlib’s `input = ttyin` pattern blindly.

### CP/M IOBYTE seeds (firmware shells)

Shell may seed **`bios_iobyte`** before handing off to CCP; BIOS copies it to page-0 IOBYTE.

- CON is **low 2 bits** (CRT vs TTY, etc.).
- Hardware-specific high bits (e.g. 8085 module **LST → SOD**) may require seeds like **`0x81` / `0x80`**, not bare `1` / `0`. Match the BIOS `list`/`const` decode, not “Z80 values”.

---

## 2b. Newlib static stdio heap sizing (FDSTRUCT committed)

Each static driver m4 places a heap block:

```text
[next:2][committed:2][prev:2]  +  FDSTRUCT body (+ edit buffer)
         \_____ 6-byte header _____/
```

- **`committed`** (and `__I_FCNTL_HEAP_SIZE` add) must equal **header + body** bytes.
- **Oversized** committed → free = next − (block+committed) **underflows** → later `open`/`fopen` can corrupt the next FDSTRUCT (e.g. stdout). Classic bug: `cpm_00_input_cons` used `$3+29` instead of **`$3+27`**.
- **Undersized** committed → free block accounting wrong (FZX once claimed 63 for a 64-byte block).

| Family | Body (typical) | committed |
|--------|----------------|-----------|
| character_00 / simple out | 17 | **23** |
| console_01 input + edit buf `$4` | 28+`$4` | **`$4+34`** |
| `cpm_00_input_cons` (BDOS buf `$3+1`) | 21+`$3` | **`$3+27`** |
| zx inkey / lastk | … | `$4+41` / `$4+36` |

**Verify:** map spans between `__i_fcntl_heap_N` and `_N+1` must equal the formula (with default edit buf, often 64 → first span 91 or 98, etc.). Multi-arg `defb \`$a, $b\`` counts as multiple bytes when hand-checking m4.

m4 comments: **do not put unquoted commas** in macro body text (breaks m4 argument parsing).

---

## 3. Disk / fcntl instantiation (newlib)

### Open path

```text
open / creat / fopen
  → asm_vopen
       → asm_target_open_p1   ; validate path; return EXTRA bytes for FDSTRUCT
       → heap_alloc(sizeof header + EXTRA) from __stdio_heap
       → asm_target_open_p2   ; fill FDSTRUCT; install JP to driver
```

Target **must** provide **`asm_target_open_p1`** and **`asm_target_open_p2`** (e.g. CP/M FCB driver `cpm_01_file`). If missing → link error:

```text
undefined symbol: asm_target_open_p1
```

That is the classic newlib “stdio disk I/O missing” symptom (#3022-class), not a compiler bug.

### CRT knobs that make or break `open` / `fopen`

Set in target `crt_config.inc` (`TAR__clib_*`):

| Knob | Role | Failure mode if wrong |
|------|------|------------------------|
| **`open_max`** | Size of fd table (static FDs + dynamic `open`s) | `open_max=0` → only static fds; **`open()` ENFILE** / no room |
| **`stdio_heap_size`** | Heap for FDSTRUCTs (FCB driver ~192 B each incl. 128 B sector buf) | Too small → heap_alloc fails on open |
| **`fopen_max`** | Max FILE structures | Must be **>** static FILE count or freelist stays empty → **`fopen` EMFILE** even when `open` works |

Rule of thumb for CP/M-class newlib CRTs with 6 static streams (stdin…stdlst) + user files:

- `open_max = 16`
- `stdio_heap_size = 1024`
- `fopen_max = 10` (or any value **greater than** static FILE count)

### Dual-stack policy (when both exist)

| API | Backend |
|-----|---------|
| Unprefixed `open` / `read` / `write` / `lseek` / `close` | Host / OS file driver (e.g. **CP/M BDOS FCB**) |
| ChaN **`f_*`** | **FatFs** + target `diskio` (raw media) |
| `printf` / console `FILE*` | Character/terminal drivers |

- **`f_*` is never an alias for FCB fcntl.** Volumes stay independent.
- Plain **`+cpm -clib=new`**: BDOS FCB only is enough — no FatFs, no physical `diskio`.
- Hardware **`-subtype=cpm`**: FCB by default; optional `-lff` dual-stack.

**One `open` owner** per binary: do not mix classic `libsrc/target/cpm/fcntl` objects with newlib `cpm_01_file` in the same link.

### Library list / rebuild traps

1. Driver must appear in the target **`library/*_sccz80.lst`** chain (often via `driver/driver.lst`).
2. Newlib `Makefile` often depends only on `config_private.inc` — **lst/source adds do not always rebuild**. Force:

   ```bash
   rm -f lib/clibs/sccz80/<target>.lib lib/clibs/sdcc_ix/<target>.lib
   make -C libsrc/newlib <target>
   ```

3. Prove the symbol is in the lib:

   ```bash
   z88dk-z80nm lib/clibs/sccz80/<target>.lib | rg 'asm_target_open|cpm_01_file'
   ```

4. Prove the app linked it: `rg 'cpm_01_file|asm_target_open|__fcntl_fdtbl_size' app.map`

---

## 4. Testing I/O (`test/suites/target_io`)

Shared serial + disk suite for **z88dk-ticks**.

| File | Role |
|------|------|
| `io_tests.c` | printf/scanf + creat/write/read/lseek/close/multi-fd |
| `fcntl_native.c` | Native `open`/`creat`/… (CP/M BDOS / newlib FCB) |
| `fcntl_host.c` + `ticks_host_fcntl.asm` | Host SYSCALL files (targets without OS fcntl) |
| `Makefile` | Per-product recipes |

### Design rules

1. **Shared tests call only `tio_*`** (`io_port.h`) — backends swap.
2. **Classic CP/M breadth:** default subtype only (`+cpm` z80 / `+cpm -clib=8085`). Do **not** fan out to 150+ machine subtypes in this suite.
3. **Newlib gates:** plain `+cpm -clib=new` (Z80) and hardware `+… -subtype=cpm -clib=new` where dual-stack FCB applies. Newlib CP/M is **not** an 8085 product — 8085 stays classic default CLIB.
4. Extend **serial** when CRTs expose more streams: RDR/PUN/LST (`stdrdr`/`stdpun`/`stdlst`), later `tty*` if instantiated.
5. Extend **disk** when drivers claim flags: keep **lseek** (SET/END + overwrite); add **`fopen`/`fread`/`fwrite`** for newlib stdio path; optional O_TRUNC/O_APPEND if implemented.
6. FatFs `f_*` is a **separate** optional gate on hardware packages — not required for plain `+cpm`.

### Run pattern

```bash
make -C test/suites/target_io                    # all recipes
make -C test/suites/target_io test_rc2014_cpm.com
# scanf tests need piped input (Makefile uses SCANF_INPUT)
```

CP/M outputs need **`.com`** for ticks CP/M mode; some newlib links produce `*_CODE.bin` that must be copied to `.com`.

### What “green” means

- Suite: `N run, N passed, 0 failed`
- Map proof for newlib disk: driver + `__fcntl_fdtbl_size` / `open_max` / heap as expected
- Classic default Z80 + 8085 still pass after mixed-tree moves (isolation)

---

## 5. Config / CRT pipelines (quick map)

| Pipeline | When | Outputs |
|----------|------|---------|
| **Config m4** | library build | `config_*_{private,public}.inc`, `config_*.h` |
| **CRT m4** | each `zcc +target` link | expand `*_crt.asm.m4` + startup + drivers |

- Modes: `CFG_ASM_DEF` / `CFG_ASM_PUB` / `CFG_C_DEF`.
- zcc maps startup → `__STARTUP`, pragmas → `M4__*`, `-I` target home + `src/m4`.
- Edit **only** the CLIB lines that point at newlib CRT/lib paths when migrating; leave classic SUBTYPE lines alone unless intentional.

---

## 6. Agent checklist (new serial or disk work)

**Serial**

- [ ] Driver class: `character_00` vs `console_01` (match peers)
- [ ] CRT m4 instantiates FILE* + FDSTRUCT; dups for err streams
- [ ] Public `stdio.h` names match what the CRT actually builds
- [ ] Multi-port: second triple uses `tty*` + `m4_file_dup` for err
- [ ] Hybrid classic console: FILE flags **18/20**, list isolation, classic include order
- [ ] Line input: newlib `getline` vs classic `fgets_cons` — **one cook layer**; no fake `ttyin` on hybrid
- [ ] Static heap: each m4 `committed` / `HEAP_SIZE` = body + 6 (map-span check)

**Disk**

- [ ] `asm_target_open_p1` / `_p2` in target lib (nm proof)
- [ ] `open_max`, `stdio_heap_size`, `fopen_max` sized for static + dynamic use
- [ ] One `open` owner; dual-stack docs if FatFs also present
- [ ] Forced lib rebuild after lst changes; app map shows driver

**Test**

- [ ] `target_io` recipe for the product (native vs host fcntl)
- [ ] Classic CP/M: default subtype only unless a specific machine is in scope
- [ ] Extend serial/disk cases only where the CRT/driver supports them
- [ ] ticks CPU model (`-m8085` when relevant)

**Dual-CPU firmware shells (Z80 newlib + 8085 classic)**

- [ ] Shared app logic; platform glue only for CRT/IOBYTE/banners
- [ ] Do **not** link newlib stdio into 8085 images
- [ ] Align line-read at one `#ifdef` call site; verify echo/BS on both

---

## Related

- Host measurement / ticks / maps / copt: **[z88dk-tooling](../z88dk-tooling/SKILL.md)**
- 8085 coding / stack rules: **[extended-usage](../extended-usage/SKILL.md)**
- Opcode map: **[opcode-reference](../opcode-reference/SKILL.md)**
- Upstream: https://github.com/z88dk/z88dk  
- Example dual-stack notes: `libsrc/_DEVELOPMENT/EXAMPLES/z80/stdio_target/readme.md`  
- Suite: `test/suites/target_io/`
