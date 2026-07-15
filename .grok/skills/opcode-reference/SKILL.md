---
name: opcode-reference
description: >
  Intel 8085 opcode reference using Zilog (Z80-style) mnemonics. Covers the full
  256-opcode map, byte sizes, cycle timings, flag effects (including undocumented
  K and V), and the ten undocumented 8085 instructions. Use when writing, reading,
  optimizing, or reviewing 8085 assembly; when mapping Intel↔Zilog mnemonics; when
  checking timings or flags; or when the user runs /opcode-reference.
---

# 8085 Opcode Reference (Zilog mnemonics)

## Conventions (always follow)

1. **Mnemonics are Zilog**, as in Z80 assembly — not Intel 8080/8085 names.
2. **Opcode bytes and timings are 8085**, not Z80 (many encodings differ or do not exist on Z80).
3. **Undocumented / extended** opcodes are noted in tables (column or section).
4. Immediate forms: `*` = 8-bit immediate (d8), `**` = 16-bit immediate/address (d16/a16). These `*`/`**` are operand placeholders only.
5. For **LDHI** / **LDSI** equivalents (`ld de,hl+*` / `ld de,sp+*`), the 8-bit offset is **unsigned**.
6. Conditional cycle counts use `taken/not-taken` (e.g. `12/6`, `10/7`, `18/9`).

Prefer the full tables in [references/opcodes.md](references/opcodes.md). Use this skill body for rules, flags, mnemonic mapping, and undocumented ops.

## Sources

| Topic | Source |
|-------|--------|
| Zilog mnemonics & descriptions | [feilipu/8085-opcodes](https://gitlab.com/feilipu/8085-opcodes) `8085_instructions.html` |
| Flag effects (S Z K A P V C), timings | [pastraiser i8085 opcodes](https://pastraiser.com/cpu/i8085/i8085_opcodes.html) |

When sources conflict on **flags**, trust **pastraiser** (8085-specific K and V). When they conflict on **mnemonic spelling**, trust **8085_instructions.html** (Zilog).

## Registers

```
15 ...... 8  7 ...... 0
     A            F      → AF (PSW); Zilog: af
     B            C      → BC
     D            E      → DE
     H            L      → HL
15 ............... 0
        SP
        PC
```

Memory via HL is written `(hl)` (Intel `M`). Stack grows downward; `push` stores high byte first at `sp-1`, low at `sp-2`.

## Flag register (F)

| Bit | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|-----|---|---|---|---|---|---|---|---|
|     | S | Z | K | A | 0 | P | V | C |

| Flag | Meaning |
|------|---------|
| **S** | Sign |
| **Z** | Zero |
| **K** | Undocumented; also **X5** / **UI** (underflow/overflow indicator) |
| **A** | Auxiliary carry (half-carry / AC) |
| **0** | Unused; always zero |
| **P** | Parity |
| **V** | Undocumented overflow |
| **C** | Carry |

Flag columns in tables are always **S Z K A P V C**:

- letter → that flag is affected as defined by the instruction
- `-` → unchanged
- `0` / `1` → forced clear / set

**Important:** On the Z80, P and V share one bit (P/V). On the **8085 they are separate bits** (P bit 2, V bit 1). Do not collapse them.

Condition codes for jumps/calls/returns:

| Zilog cc | Meaning |
|----------|---------|
| `nz` / `z` | Z clear / set |
| `nc` / `c` | C clear / set |
| `po` / `pe` | P odd / even (parity) |
| `p` / `m` | S clear (plus) / set (minus) |
| `nk` / `k` | K clear / set (undocumented) |
| `v` (rst v only) | V set |

## Intel → Zilog mnemonic map (primary)

Use Zilog in all generated/edited 8085 code. Intel names appear only when translating legacy sources.

| Intel | Zilog |
|-------|-------|
| `NOP` | `nop` |
| `LXI rp,d16` | `ld bc/de/hl/sp,**` |
| `STAX B/D` | `ld (bc),a` / `ld (de),a` |
| `LDAX B/D` | `ld a,(bc)` / `ld a,(de)` |
| `INX/DCX rp` | `inc/dec bc/de/hl/sp` |
| `INR/DCR r` | `inc/dec r` ; `M` → `(hl)` |
| `MVI r,d8` | `ld r,*` |
| `MOV r1,r2` | `ld r1,r2` |
| `RLC/RRC` | `rlca` / `rrca` |
| `RAL/RAR` | `rla` / `rra` |
| `DAD rp` | `add hl,bc/de/hl/sp` |
| `LDA/STA a16` | `ld a,(**)` / `ld (**),a` |
| `LHLD/SHLD a16` | `ld hl,(**)` / `ld (**),hl` |
| `DAA` | `daa` |
| `CMA` | `cpl` |
| `STC/CMC` | `scf` / `ccf` |
| `HLT` | `halt` |
| `ADD/ADC r` | `add a,r` / `adc a,r` |
| `SUB/SBB r` | `sub r` / `sbc a,r` |
| `ANA/XRA/ORA/CMP r` | `and r` / `xor r` / `or r` / `cp r` |
| `ADI/ACI/SUI/SBI` | `add a,*` / `adc a,*` / `sub *` / `sbc a,*` |
| `ANI/XRI/ORI/CPI` | `and *` / `xor *` / `or *` / `cp *` |
| `JMP/Jcc` | `jp **` / `jp cc,**` |
| `CALL/Ccc` | `call **` / `call cc,**` |
| `RET/Rcc` | `ret` / `ret cc` |
| `PCHL` | `jp (hl)` |
| `SPHL` | `ld sp,hl` |
| `XCHG` | `ex de,hl` |
| `XTHL` | `ex (sp),hl` |
| `PUSH/POP B\|D\|H\|PSW` | `push/pop bc\|de\|hl\|af` |
| `IN/OUT d8` | `in a,(*)` / `out (*),a` |
| `EI/DI` | `ei` / `di` |
| `RIM/SIM` | `rim` / `sim` (8085 only) |
| `RST n` | `rst 00h` … `rst 38h` |

### Undocumented / extended

| Op | Intel | Zilog | Bytes | Cycles | Flags (SZKAPVC) | Effect |
|----|-------|-------|-------|--------|-----------------|--------|
| `08` | DSUB | `sub hl,bc` | 1 | 10 | `SZKAPVC` | HL ← HL − BC |
| `10` | ARHL | `sra hl` | 1 | 7 | `-----0C` | Arithmetic right shift HL; V←0 |
| `18` | RDEL | `rl de` | 1 | 10 | `-----VC` | Rotate DE left through C |
| `28` | LDHI d8 | `ld de,hl+*` | 2 | 10 | `-------` | DE ← HL + unsigned * |
| `38` | LDSI d8 | `ld de,sp+*` | 2 | 10 | `-------` | DE ← SP + unsigned * |
| `CB` | RSTV | `rst v` | 1 | 12/6 | `-------` | If V set: push PC, PC←40h |
| `D9` | SHLX | `ld (de),hl` | 1 | 10 | `-------` | (DE)←L, (DE+1)←H |
| `DD` | JNK a16 | `jp nk,**` | 3 | 10/7 | `-------` | Jump if K=0 (also jnx5/jnui) |
| `ED` | LHLX | `ld hl,(de)` | 1 | 10 | `-------` | L←(DE), H←(DE+1) |
| `FD` | JK a16 | `jp k,**` | 3 | 10/7 | `-------` | Jump if K=1 (also jx5/jui) |

These are **not** standard Z80 opcodes at those encodings (Z80 uses `CB`/`DD`/`ED`/`FD` as prefixes). On 8085 they are single-byte (or 3-byte jump) instructions.

## Flag rules agents must not get wrong

| Group | Flags | Notes |
|-------|-------|-------|
| `inc`/`dec` **16-bit** (`inc bc` … `dec sp`) | `--K----` | Only **K** changes |
| `inc`/`dec` **8-bit** (incl. `(hl)`) | `SZKAPV-` | All but **C** |
| `rlca` / `rla` | `-----VC` | V and C |
| `rrca` / `rra` | `-----0C` | **V forced 0** |
| `add hl,rp` | `-----VC` | V and C (not C alone) |
| 8-bit ALU (`add`…`cp`, immediates) | `SZKAPVC` | Full set |
| `sub hl,bc` (undoc) | `SZKAPVC` | Full set |
| `sra hl` (undoc) | `-----0C` | V←0, C from bit 0 |
| `rl de` (undoc) | `-----VC` | V and C |
| `scf` | `------1` | C←1 |
| `ccf` | `------C` | C toggled |
| `pop af` | `SZKAPVC` | Restores **all** flags including K,V |

Logical ops still use the full pastraiser mask `SZKAPVC` (how individual bits are computed is instruction-defined; do not invent Z80 N-flag behavior — 8085 has **no N flag**).

## Timing notes

- Base machine cycles as in the reference tables (8085 T-states / clocks from pastraiser).
- Conditional **ret**: 12 taken / 6 not taken.
- Conditional **jp**: 10 / 7.
- Conditional **call**: 18 / 9.
- `halt` is 5 cycles (8085).

## Coding rules for this project

1. Emit **Zilog** mnemonics only (`ld a,b` not `MOV A,B`; `jp nz,label` not `JNZ`).
2. Register pairs: `bc`, `de`, `hl`, `af`, `sp` — never Intel `B`, `D`, `H`, `PSW` in new code.
3. Use `(hl)`, `(bc)`, `(de)`, `(**)` for memory; never `M`.
4. Prefer undocumented ops when they clearly win (e.g. `sub hl,bc`, `ld hl,(de)`, `ld (de),hl`, `ld de,hl+*`) **and** the target assembler/CPU path supports them.
5. Never assume Z80 instruction timings or prefix opcodes exist on 8085.
6. When optimizing, consult [references/opcodes.md](references/opcodes.md) for exact size/cycle/flag data.

## Quick lookup

Full 16×16 opcode grid, Intel cross-ref, and macro helpers:

→ **[references/opcodes.md](references/opcodes.md)**
