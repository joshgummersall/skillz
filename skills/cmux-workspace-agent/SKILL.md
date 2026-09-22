---
name: cmux-workspace-agent
description: Open a new instance of whichever coding agent CLI (claude, codex, ...) is active in the current cmux surface, in a new cmux workspace, immediately running a given prompt or skill with the syntax appropriate to that agent. Use when the user asks to "open claude/codex in a new workspace/tab" or "run X in a new cmux workspace".
---

# cmux Workspace + Agent

Launches a fresh coding-agent CLI session in a new cmux workspace, pre-loaded with a prompt. Mirrors whatever agent is currently running the caller's session (claude, codex, ...) rather than assuming `claude`. Same idea as `cmux-split-agent`, but opens a new workspace (tab) instead of splitting the current one.

`ARGUMENTS` is the skill invocation or free-text prompt to run (e.g. `$thermo-nuclear-code-quality-review`, `/thermo-nuclear-code-quality-review`, or `review this diff for security issues`).

## Detecting the active agent

cmux tags every terminal it launched an agent CLI in with `CMUX_AGENT_LAUNCH_KIND` (and `CMUX_AGENT_LAUNCH_EXECUTABLE` for the resolved binary path) — the same value it uses for `new-surface --provider`. Read it from the current shell:

```bash
echo "$CMUX_AGENT_LAUNCH_KIND"       # e.g. "claude" or "codex"
```

- If set, use that value as the command name (`claude`, `codex`, ...).
- If unset (not a cmux-launched agent terminal — e.g. a plain shell), fall back: `CLAUDECODE=1` implies `claude`; otherwise ask the user which agent to open rather than guessing.

Do not hardcode `claude`.

## Adapting skill invocations

When the user requests a skill, normalize its invocation for the detected agent, regardless of which prefix the user supplied:

- Claude: `/skill-name` (for example, `/thermo-nuclear-code-quality-review`).
- Codex: `$skill-name` (for example, `$thermo-nuclear-code-quality-review`).

Preserve the skill name and any trailing arguments. Pass free-text prompts through unchanged; do not rewrite arbitrary dollar signs, paths, or built-in slash commands as skills. For other agents, preserve the supplied invocation unless their skill syntax is known.

## Why this is one call, not three

`cmux-split-agent` needs a create-then-`send`-then-`send-key enter` dance because `--command` on `new-split` wasn't reliably wired up as of cmux 0.64.22. `new-workspace --command` has always worked: it starts the workspace's regular interactive shell and types the text plus one Enter into it at spawn time, so the command runs immediately and the shell stays alive after. One call is enough — don't split it into create-then-`send`.

## Steps

1. Determine the agent command: `echo "$CMUX_AGENT_LAUNCH_KIND"` (see above).
2. Normalize the skill prefix as above, then build the agent invocation string. Quote it for the destination shell: single-quote the prompt, escaping each embedded `'` as `'\''`.
3. Create the workspace with that command as its initial input, keeping the caller's working directory so the agent starts with the same project context:
   ```bash
   cmux new-workspace --cwd "$PWD" --command 'claude '\''/thermo-nuclear-code-quality-review'\''' --focus true
   ```
   or, for Codex:
   ```bash
   cmux new-workspace --cwd "$PWD" --command 'codex '\''$thermo-nuclear-code-quality-review'\''' --focus true
   ```
   Use only the invocation matching the detected agent. `--focus true` switches to the new workspace immediately; omit it (or pass `false`) to leave the caller's workspace focused. When building the command programmatically, shell-quote the invocation text explicitly — JSON string encoding is not shell quoting.

Do not fall back to create-then-`send`-then-`send-key enter` — see "Why this is one call" above.
