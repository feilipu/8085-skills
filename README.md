# 8085-skills

These AI skill documents improve work done by agents (Claude, Grok Build, and similar) on the z88dk code base and related technical prose.

Most skills focus on **Intel 8085** assembly and tooling. The **ste-writing** skill covers documentation style for the same projects.

Although the Intel mnemonics are usually used with the 8085, the z88dk code base is wholly Z80 (and derivatives) based. Therefore it is easier to read and maintain if consistent mnemonics are used.

The 8085 has useful extension opcodes and flags. Intel shipped them at launch, so they are available on any 8085 CPU. The z88dk libraries use them heavily for stack access and for comparison operations.

The z88dk-z80asm assembler supports MACROs and "synthetic" opcodes. Synthetics are usually chains of operations without harmful side effects. For example **`ld bc,de`** is a 16-bit copy as two 8-bit loads, and **`ld a,(hl+)`** is an indirect load then an index increment. These forms keep code short and can share one source across CPUs without conditional assembly.

## Skills (Grok / Claude)

| Skill | Path | Use when |
|-------|------|----------|
| **opcode-reference** | `.grok/skills/opcode-reference/SKILL.md` | Full 256-opcode map, Zilog mnemonics, flags (S Z K A P V C), undocumented ops, timings |
| **extended-usage** | `.grok/skills/extended-usage/SKILL.md` | Stack-only locals, `ld de,sp+*`, frame rebuild, K-flag loops, mul/div building blocks, pitfalls |
| **z88dk-tooling** | `.grok/skills/z88dk-tooling/SKILL.md` | `z88dk-ticks`, hotspots, maps/nm/disasm, math suite, benchmarks, A/B measurement pitfalls, **`z88dk-copt` / `lib/z80rules.*`** (compiler peephole vs hand-written library asm; match target-file whitespace) |
| **z88dk** | `.grok/skills/z88dk/SKILL.md` | Newlib vs classic trees, CRT m4 **serial/FILE\*** instantiation, **disk fcntl** (`asm_target_open`, `open_max`/`fopen_max`, FCB vs FatFs), dual-stack policy, **`target_io`** testing, migration isolation, **`libsrc` one major function per file** |
| **ste-writing** | `.grok/skills/ste-writing/SKILL.md` | Prose only (docs, READMEs, PR text, release notes, comments — **not** code). ASD-STE100 Simplified Technical English. Modes: **strict** (procedures, safety, errors) and **STE-flavored** (general docs). Removes "AI slop" form without changing technical truth |

Full opcode tables: `.grok/skills/opcode-reference/references/opcodes.md`

Official STE standard (copyrighted; do not paste in full): https://asd-ste100.org

## Related

- Repo: https://github.com/feilipu/8085-skills
- Design notes: https://feilipu.me/2021/09/27/8085-software/
