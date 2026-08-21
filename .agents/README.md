# `.agents` — 8085-skills agent material

Canonical tree for AI agent skills in this repo. **Do not duplicate** under other names.

## Layout

```text
.agents/
  README.md
  skills/
    cpu-8085/              # + references/opcodes.md
    library-classic/
    library-math32/
    compiler-sccz80/
    tool-zcc/
    tool-ticks/
    target-cpm/
    style-ste-writing/
    methodology-measure/
    …
```

Each skill is `.agents/skills/<name>/SKILL.md`. The directory name equals the YAML `name:` field.
Optional `references/` for large tables (8085 opcodes).

Claude Code and similar hosts scan **one** level under `skills/`. Do **not** add category folders (`cpu/`, `tool/`, …). Grouping lives in root `AGENTS.md`.

**CPU pack:** this repo ships **`cpu-8085` only**. Do not add `cpu-z80`, `cpu-gbz80`, `cpu-z180`, `cpu-z80n`, or `cpu-8080` here. Those live in the z88dk tree.

**Targets:** only 8085-relevant cards (`target-cpm`, `target-rc2014`). Other platforms stay in z88dk.

## Entry points

At the **repo root**:

- `.agents/` — real directory (this tree)
- `.grok` → `.agents` (symlink)
- `.claude` → `.agents` (symlink)
- `AGENTS.md` — always-on project rules + skill index

Agents must **realpath-dedupe** so following both `.grok` and `.claude` does not load the same skill twice. See root `AGENTS.md`.

## Adding a skill

1. Create `.agents/skills/<name>/SKILL.md`.
2. Set a unique `name:` that matches the directory (prefer `category-name`, e.g. `tool-foo`, `target-bar`).
3. Write a specific `description:` (triggers auto-invocation) — **narrow** so it does not fire on unrelated work.
4. Link related skills by `name` (e.g. `tool-z80asm`), or by path `.agents/skills/<name>/`.
5. Keep agent-dense: tables, commands, pitfalls — not full human manuals.
6. Do **not** add another CPU skill. Point to the z88dk checkout for Z80, gbz80, and peers.

## Source hierarchy when facts disagree

1. Built tools and sources in the **z88dk** checkout  
2. In-tree READMEs / man pages next to those tools  
3. **CPU opcode acceptance (assembler):** `src/z80asm/dev/cpu/cpu_test_8085_{ok,err}.asm` — last resort for “does z80asm accept this source line on `-m8085`?” Decode line format in skill **`tool-z80asm`**.  
4. This repo’s skill bodies  
5. External blogs — last resort  

Do not bulk-read the large fixtures; `rg` the mnemonic.

## Origin

Skill names and split match [z88dk](https://github.com/z88dk/z88dk) `.agents/skills/`. Content started in this repo as `opcode-reference`, `extended-usage`, `z88dk`, `z88dk-tooling`, and `ste-writing`.
