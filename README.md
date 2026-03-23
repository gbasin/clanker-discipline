# clanker-discipline

Discipline AI coding agents against state explosion, grab-bag models, and mutation ambiguity.

## Install

```bash
npx skills add gbasin/clanker-discipline
```

## What it does

AI coding agents overproduce state. Every bug gets one more flag. Every feature gets one more optional field. This skill teaches agents to:

1. **Store evidence, not conclusions** — derive state from events instead of caching it in flags
2. **Make wrong states impossible** — discriminated unions over optional field bags
3. **Brand primitives** — prevent type-identical domain concepts from being interchangeable
4. **Know what your functions are** — keep semantic functions pure, let pragmatic functions be messy
5. **Pick a mutation contract** — mutate-and-return-void or clone-and-return-new, never both
6. **Encapsulate state** — closures over class fields, local over global
7. **Data over procedure** — lookup tables over if-chains
8. **Debug with data** — export events, write pure-function tests, no mocking

## Credits

Combines ideas from [theswerd/aicode](https://github.com/theswerd/aicode) ([self-documenting code](https://github.com/theswerd/aicode/blob/main/skills/self-documenting-code/SKILL.md)) and [swyxio's event sourcing post](https://x.com/niclofreon/status/1902782543620178324) on combating agent state explosion.
