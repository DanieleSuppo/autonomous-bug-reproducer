# Domain Docs

How the engineering skills should consume this repo's domain documentation when exploring the codebase.

## Before exploring, read these

- **`CONTEXT.md`** at the repo root, or
- **`CONTEXT-MAP.md`** at the repo root if it exists. It points at one `CONTEXT.md` per context. Read each one relevant to the topic.
- **`docs/adr/`**. Read ADRs that touch the area you are about to work in. In multi-context repos, also check `src/<context>/docs/adr/` for context-scoped decisions.

If any of these files do not exist, **proceed silently**. Do not flag their absence or suggest creating them upfront. The `/domain-modeling` skill creates them lazily when terms or decisions actually get resolved.

## File structure

Single-context repo:

```
/
|- CONTEXT.md
|- docs/adr/
`- src/
```

## Use the glossary's vocabulary

When your output names a domain concept, use the term as defined in `CONTEXT.md`. If it is not defined, reconsider the term or note the gap for `/domain-modeling`.

## Flag ADR conflicts

If your output contradicts an existing ADR, surface it explicitly rather than silently overriding it.
