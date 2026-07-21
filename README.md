# 8085-skills

These AI skills documents are designed to improve the 8085 code produced by AI agents, like Claude or Grok-build, when used with the z88dk code base.

Although the Intel mnemonics are usually used with the 8085, the z88dk code base is wholely z80 (and derivatives) based. Therefore it is easier to read and maintain if consistent mnemonics are used.

The 8085 has very useful extension opcodes and flags, which were incorporated by Intel at launch and are therefore available on any 8085 CPU. These opcodes and flags are used extensively for stack access, and for comparison operations, within the z88dk libraries.

The z88dk-z80asm assembler has a MACRO capability, and it has the capability to support "synthetic" opcodes. Synthetic opcodes are usually created by a series of useful operations that can be chained together without side effects. For example **`ld bc,de`** which is a 16-bit emulation by two 8-bit operations, or **`ld a,(hl+)`** which is an indirect load followed by an increment of the index register. These operations can make code more concise or allow an operation supported by one CPU to be used across multiple CPUs without deploying conditional assembly.

## Skills (Grok / Claude)

| Skill | Path | Use when |
|-------|------|----------|
| **opcode-reference** | `.grok/skills/opcode-reference/SKILL.md` | Full 256-opcode map, Zilog mnemonics, flags (S Z K A P V C), undocumented ops, timings |
| **extended-usage** | `.grok/skills/extended-usage/SKILL.md` | Stack-only locals, `ld de,sp+*`, frame rebuild, K-flag loops, mul/div building blocks, pitfalls |
| **z88dk-tooling** | `.grok/skills/z88dk-tooling/SKILL.md` | `z88dk-ticks`, hotspots, maps/nm/disasm, math suite, benchmarks, A/B measurement pitfalls, **`z88dk-copt` / `lib/z80rules.*`** (compiler peephole vs hand-written library asm; match target-file whitespace) |

Full opcode tables: `.grok/skills/opcode-reference/references/opcodes.md`

## Related

- Repo: https://github.com/feilipu/8085-skills
- Design notes: https://feilipu.me/2021/09/27/8085-software/

