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
- [`cmux-workspace-agent`](skills/cmux-workspace-agent/SKILL.md) — same as
  `cmux-split-agent`, but opens a new cmux workspace instead of a split.
- [`rex-split-agent`](skills/rex-split-agent/SKILL.md) — Rex equivalent of
  `cmux-split-agent`: open a coding-agent CLI in a Rex split pane.
- [`rex-tab-agent`](skills/rex-tab-agent/SKILL.md) — Rex equivalent of
  `cmux-workspace-agent`: open a coding-agent CLI in a new Rex tab.
- [`rex-session-agent`](skills/rex-session-agent/SKILL.md) — open a
  coding-agent CLI in a new Rex session and switch the app to it.

## Install

Add the marketplace, then install the plugin you want:

```
/plugin marketplace add joshgummersall/skillz
/plugin install cmux@skillz
/plugin install rex@skillz
```

- `cmux@skillz`: `cmux-split-agent`, `cmux-workspace-agent`
- `rex@skillz`: `rex-split-agent`, `rex-tab-agent`, `rex-session-agent`

## Adding a skill

1. Create a new directory: `mkdir skills/<skill-name>`
2. Add `skills/<skill-name>/SKILL.md` with YAML frontmatter (`name`,
   `description`) followed by instructions in Markdown.
3. Add a link to it under Skills above.
4. Add its path to the matching plugin's `skills` list in
   `.claude-plugin/marketplace.json`.
