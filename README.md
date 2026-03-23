# clanker-discipline

Discipline AI coding agents against state explosion, grab-bag models, and mutation ambiguity.

## Install

```bash
npx skills add gbasin/clanker-discipline --all -g
```

## What it does

AI coding agents overproduce state — every bug gets one more flag, every feature one more optional field. This skill teaches agents to:

1. **Derive, don't store** — if it can be computed from existing data, don't cache it in a flag
2. **Make wrong states impossible** — discriminated unions over optional bags, null over sentinels, branded primitives
3. **Enforce function contracts** — pure functions stay pure; mutate+void or clone+return, never both
4. **Data over procedure** — if every branch returns the same shape, it's a table, not an if-chain

## Credits

Combines ideas from [theswerd/aicode](https://github.com/theswerd/aicode) ([self-documenting code](https://github.com/theswerd/aicode/blob/main/skills/self-documenting-code/SKILL.md)) and [Tommy D. Rossi's event sourcing post](https://x.com/__morse/status/2032107422525907273) on combating agent state explosion.

