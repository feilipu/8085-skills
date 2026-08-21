# 8085-skills

These AI skill documents improve work done by agents (Claude, Grok Build, and similar) on **Intel 8085** assembly in the z88dk code base. They also cover the **general** z88dk library, compiler, and host-tool paths that 8085 work uses.

This repo ships **`cpu-8085` only**. It does not include Z80, gbz80, Z180, Z80N, or 8080 CPU skills. Those live in the [z88dk](https://github.com/z88dk/z88dk) tree.

Although Intel mnemonics are usual for the 8085, the z88dk code base uses **Zilog** mnemonics everywhere. Use Zilog in new and edited sources.

The 8085 has useful extension opcodes and flags. Intel shipped them at launch, so they are available on any 8085 CPU. The z88dk libraries use them heavily for stack access and for comparison operations.

The z88dk-z80asm assembler supports MACROs and synthetic opcodes. Synthetics are usually chains of operations without harmful side effects. For example **`ld bc,de`** is a 16-bit copy as two 8-bit loads, and **`ld a,(hl+)`** is an indirect load then an index increment.

## Layout

Canonical tree: **`.agents/`**. Discovery links: **`.grok`** and **`.claude`** → `.agents`. Index: **`AGENTS.md`**.

Each skill is `.agents/skills/<name>/SKILL.md`. The directory name matches the YAML `name:` field.

Full 8085 opcode tables: `.agents/skills/cpu-8085/references/opcodes.md`

## Skills

| Skill | Use when |
|-------|----------|
| **cpu-8085** | 8085 asm, opcode map, flags (S Z K A P V C), stack-only locals, extended ops |
| **library-classic** | Classic clib, hybrid 8085 consoles, isolation from newlib |
| **library-newlib** | CRT m4, FILE*, `asm_target_open`, dual-stack FCB vs FatFs |
| **library-math32** | IEEE float cores, 8085 vs z80 products, restoring div / NR inv |
| **library-math16** | Half float |
| **library-am9511** | Am9511A APU float |
| **compiler-sccz80** | sccz80 codegen and `libsrc/l/sccz80/7-8085` runtime |
| **compiler-zsdcc** | zsdcc / sdcc_ix / sdcc_iy (Z80-class; not the 8085 classic product) |
| **compiler-80cc** | 80cc; no `-fframe-pointer` on 8085 |
| **tool-zcc** | `zcc` front end |
| **tool-ticks** | `z88dk-ticks` (CPU flag before the binary) |
| **tool-z80asm** | assembler, synthetics, 8085 ok/err fixtures |
| **tool-copt** | peephole rules; library asm is never copt’d |
| **tool-z80nm** / **tool-dis** | link proof and disassembly |
| **tool-appmake** | tape/disk/ROM/hex packaging |
| **tool-zobjcopy** / **tool-z88dk-lib** / **tool-gdb** | object, third-party lib, gdb |
| **tool-zx0** / **tool-zx7** | compressors |
| **tool-ucpp** / **tool-zpragma** | usually via zcc |
| **tool-asmpp** / **tool-asmstyle** / **tool-basck** | niche helpers |
| **target-cpm** | `+cpm` (8085 is classic) |
| **target-rc2014** | `+rc2014` (`acia85` / `uart85` / `basic85`) |
| **style-libsrc-layout** | one major function per `libsrc` file |
| **style-ste-writing** | prose only (docs, PR text, comments — not code) |
| **methodology-measure** | TIMER, hotspots, A/B, benches, wiki numbers |
| **methodology-sdcc-vanilla** | stock `sdcc` benches, not zsdcc |

Official STE standard (copyrighted; do not paste in full): https://asd-ste100.org

## Old names

The 2026 split matches z88dk `.agents/skills/`. These names no longer exist as skill directories:

| Old | Now |
|-----|-----|
| `opcode-reference` | `cpu-8085` |
| `extended-usage` | `cpu-8085` |
| `z88dk` | `library-classic`, `library-newlib`, `library-math32`, `style-libsrc-layout` |
| `z88dk-tooling` | `methodology-measure` plus the `tool-*` cards |
| `ste-writing` | `style-ste-writing` |

## Related

- Repo: https://github.com/feilipu/8085-skills
- z88dk (full multi-CPU skill set): https://github.com/z88dk/z88dk
- Design notes: https://feilipu.me/2021/09/27/8085-software/
