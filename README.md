# skillz

A grab-bag of Claude Code skills. No unifying theme — each one solves a
specific, one-off problem I wanted to reuse.

## Layout

Each skill is a directory under `skills/` containing a `SKILL.md`:

```
skills/
  <skill-name>/
    SKILL.md    # required: frontmatter (name, description) + instructions
    *           # optional supporting scripts/assets
```

Skill directory names are kebab-case and match the `name:` field in the
`SKILL.md` frontmatter.

## Skills

- [`cmux-split-agent`](skills/cmux-split-agent/SKILL.md) — open a new
  coding-agent CLI instance in a cmux split pane, running a given prompt or
  skill.

## Adding a skill

1. Create a new directory: `mkdir skills/<skill-name>`
2. Add `skills/<skill-name>/SKILL.md` with YAML frontmatter (`name`,
   `description`) followed by instructions in Markdown.
3. Add a link to it under Skills above.
