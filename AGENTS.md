# 8085-skills — agent instructions

Project rules for AI agents working from this skill pack. These skills target **Intel 8085** work in [z88dk](https://github.com/z88dk/z88dk) plus the **general** z88dk library, compiler, tool, and methodology cards that 8085 library work needs.

This repo does **not** ship other CPU packs. For Z80, gbz80, Z180, Z80N, or 8080 coding rules, use the z88dk tree.

## Canonical skill root (do not double-read)

| Path | Role |
|------|------|
| **`.agents/`** | **Only** real tree of agent skills |
| **`.grok`** | Symlink → `.agents` (Grok discovery) |
| **`.claude`** | Symlink → `.agents` (Claude discovery) |

**Rules**

1. Prefer paths under **`.agents/skills/...`** in prose and links.
2. Before loading a skill file, **resolve realpath**. If that realpath was already loaded via `.grok` or `.claude`, **do not read it again**.
3. Skill `name` values are globally unique (`cpu-8085`, `tool-ticks`, …). Hosts that dedupe by name still must not re-apply the same body under another alias.
4. Do **not** add a second full instruction set as root `CLAUDE.md` or `.agents/CLAUDE.md` that duplicates this file or the skills.
5. This **`AGENTS.md`** is the only always-on project rules file at the repo root.

Skills load **on demand** when the task matches their `description`.

**Context budget (mandatory)**

1. Do **not** bulk-read every skill under `.agents/skills/`.
2. Do **not** open every tool skill “just in case.”
3. Open **one** target skill when the task names that machine (`+cpm` or `+rc2014`). Other platforms: z88dk `lib/config/<name>.cfg`.
4. The index tables in this file are enough to *choose* a skill; open a `SKILL.md` only when that topic is in scope.

## Environment (when used with a z88dk tree)

```bash
export PATH=/path/to/z88dk/bin:$PATH
export ZCCCFG=/path/to/z88dk/lib/config
```

Assume a built z88dk tree (`bin/` tools present) unless the task is to build the toolchain.

Tree paths in the skills (`libsrc/`, `src/z80asm/dev/cpu/`, `support/benchmarks/`, `.agents/scripts/`) refer to the **z88dk checkout**, not this repo.

## Hard house rules (always)

1. **`libsrc/`**: one major function / logical operation per source file (`style-libsrc-layout`).
2. **Hand-written library asm** is assembled as-is — **`z88dk-copt` never runs on it** (`tool-copt`).
3. **Classic vs newlib**: do not merge stdio/fcntl cores in one link without a designed bridge (`library-classic`, `library-newlib`).
4. **Mnemonics:** **Zilog everywhere** in z88dk sources. z80asm may accept **Intel** forms on 8080/8085 for **external-code compatibility only** — do not write Intel mnemonics in z88dk sources (`cpu-8085`, `tool-z80asm`).
5. **8085:** extended ops and **stack-only** locals (`cpu-8085`).
6. **Measure** with `z88dk-ticks`; put **CPU flag before the binary** (`tool-ticks`, `methodology-measure`).
7. **math32 / math16**: `div` = restoring; `inv` = Newton–Raphson (`library-math32`, `library-math16`).
8. **CPU opcode capability (last resort):** fixtures in z88dk `src/z80asm/dev/cpu/` (`cpu_test_8085_ok.asm` / `_err.asm`). **ok** may be native, multi-byte synthetic, or `call __z80asm__*`. **`_strict_`** = strict mode (**synthetics forbidden**). Fixtures may list Intel spellings for compat tests; emit **Zilog** in tree work. How to read lines: skill **`tool-z80asm`**. `rg` one mnemonic; never bulk-load huge `*_err.asm` files.

## Commit hygiene (this repo)

1. **One subject line. No body.** Commit with a single `-m`.
2. **No attribution trailers.** No `Co-Authored-By:`, no “generated with” footer, no tool or model credit.
3. Commit only when asked, and never push unasked.

## Skill index

Load only what the task needs. Paths are under `.agents/skills/`.

Each skill lives at `.agents/skills/<name>/SKILL.md`. The directory name is the `name:` field. Hosts that scan only one level under `skills/` need this flat layout. Do **not** add category folders.

### CPU

| Skill | When |
|-------|------|
| `cpu-8085` | 8085 asm, opcodes, stack rules, extended ops |

This repo has **no** `cpu-z80`, `cpu-gbz80`, `cpu-z180`, `cpu-z80n`, or `cpu-8080` skills.

### Library

| Skill | When |
|-------|------|
| `library-classic` | Classic clib, multi-CPU, hybrid consoles |
| `library-newlib` | CRT m4, FILE*, open, dual-stack |
| `library-math32` | IEEE float cores, multi-CPU rebuild |
| `library-math16` | Half float |
| `library-am9511` | Am9511A APU float, status/specials, techdocs |

### Compiler

| Skill | When |
|-------|------|
| `compiler-sccz80` | sccz80 + runtime + copt interaction |
| `compiler-zsdcc` | zsdcc / sdcc_ix / sdcc_iy / patch pin |
| `compiler-80cc` | 80cc; 8085 has no IX — do not pass `-fframe-pointer` |

### Tools

| Skill | Binary / topic |
|-------|----------------|
| `tool-zcc` | `zcc` |
| `tool-ticks` | `z88dk-ticks` |
| `tool-z80asm` | `z88dk-z80asm` |
| `tool-copt` | `z88dk-copt` / hand-asm hygiene |
| `tool-z80nm` | `z88dk-z80nm` |
| `tool-dis` | `z88dk-dis` |
| `tool-appmake` | `z88dk-appmake` |
| `tool-zobjcopy` | `z88dk-zobjcopy` |
| `tool-z88dk-lib` | `z88dk-lib` |
| `tool-gdb` | `z88dk-gdb` |
| `tool-zx0` / `tool-zx7` | compressors |
| `tool-ucpp` / `tool-zpragma` | usually via zcc only |
| `tool-asmpp` / `tool-asmstyle` / `tool-basck` | niche helpers |

### Targets (load only the named `+target`)

| Skill | `zcc` |
|-------|-------|
| `target-cpm` | `+cpm` |
| `target-rc2014` | `+rc2014` (8085 subtypes `acia85` / `uart85` / `basic85`) |

Any other target: z88dk `lib/config/<name>.cfg`. Host TIMER / suites: `+test` + `methodology-measure` / `tool-ticks`.

### Style and methodology

| Skill | When |
|-------|------|
| `style-libsrc-layout` | New or split library files |
| `style-ste-writing` | Human prose only (not code) |
| `methodology-measure` | A/B, hotspots, z88dk benches, suites, wiki numbers |
| `methodology-sdcc-vanilla` | Stock `sdcc` benches (`*/sdcc/`), not zsdcc |

## Old skill names (removed)

| Old name | Now |
|----------|-----|
| `opcode-reference` | `cpu-8085` (opcode map + `references/opcodes.md`) |
| `extended-usage` | `cpu-8085` (extended instruction usage) |
| `z88dk` | `library-classic`, `library-newlib`, `library-math32`, `style-libsrc-layout` |
| `z88dk-tooling` | `methodology-measure`, `tool-ticks`, `tool-copt`, and the other `tool-*` cards |
| `ste-writing` | `style-ste-writing` |

## Human docs vs agent skills

| Audience | Location |
|----------|----------|
| Agents | `.agents/skills/<name>/SKILL.md`, this file |
| Humans (z88dk wiki drafts) | z88dk `wiki/` |
| Product library notes | e.g. z88dk `libsrc/math/float/math32/readme.md` |

When wiki and the z88dk tree disagree, **the tree wins**.
