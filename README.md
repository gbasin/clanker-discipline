# clanker-discipline

Discipline AI coding agents against state explosion, grab-bag models, and mutation ambiguity.

## Install

```bash
npx skills add gbasin/clanker-discipline
```

## What it does

AI coding agents overproduce state — every bug gets one more flag, every feature one more optional field. This skill teaches agents to:

1. **Derive, don't store** — compute state from existing data instead of caching it in flags; encapsulate what you can't eliminate
2. **Make wrong states impossible** — discriminated unions, null over sentinels, branded primitives, phased composition
3. **Enforce function contracts** — never add side effects to a pure function; pick a mutation contract (mutate+void or clone+return)
4. **Data over procedure** — lookup tables over if-chains

## Credits

Combines ideas from [theswerd/aicode](https://github.com/theswerd/aicode) ([self-documenting code](https://github.com/theswerd/aicode/blob/main/skills/self-documenting-code/SKILL.md)) and [Tommy D. Rossi's event sourcing post]([https://x.com/niclofreon/status/1902782543620178324](https://x.com/__morse/status/2032107422525907273)) on combating agent state explosion.
