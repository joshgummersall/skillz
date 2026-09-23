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

## Why the prompt goes through a file, not the command line

`--command`'s value gets typed as literal keystrokes into the new workspace's shell (zsh, bash, ...), exactly as if the user had typed it — so the prompt text is parsed twice: once by whatever quoting got it into the `cmux new-workspace` call, and again by that destination shell reading it off the terminal. No single escaping scheme survives both. Concretely, `is there a way to avoid casting?` breaks zsh's glob expansion on the bare `?` (`zsh: no matches found: ...casting?`) no matter how correctly the outer quoting was done for the calling shell; embedded `"`, `&`, backticks, or `$(...)` fail the same way.

Avoid this by never typing the prompt text itself: write it to a file with the `Write` tool (a tool parameter, not shell text — no escaping applies), and have `--command` be a short, fixed, character-safe wrapper that reads that file via command substitution:

```
claude "$(cat /tmp/cmux-agent-prompt.ab12cd.md)"
```

The file path is the only variable part. Generate it with `mktemp` so concurrent invocations can't collide, and it will contain no spaces or quote characters, so the wrapper never needs prompt-specific escaping.

## Steps

1. Determine the agent command: `echo "$CMUX_AGENT_LAUNCH_KIND"` (see above).
2. Normalize the skill prefix as above, then create a unique temp file and write the exact agent invocation text into it with the `Write` tool (not a shell heredoc/echo — see above for why):
   ```bash
   mktemp "${TMPDIR:-/tmp}/cmux-agent-prompt.XXXXXX.md"
   ```
   Write the invocation text verbatim as the file's entire content, e.g. `/thermo-nuclear-code-quality-review` (Claude), `$thermo-nuclear-code-quality-review` (Codex), or the raw free-text prompt.
3. Create the workspace with the wrapper command as its initial input, keeping the caller's working directory so the agent starts with the same project context. Only the fixed wrapper and the mktemp'd path appear here, so plain single-quoting for the calling shell is always sufficient:
   ```bash
   cmux new-workspace --cwd "$PWD" --command 'claude "$(cat /tmp/cmux-agent-prompt.ab12cd.md)"' --focus true
   ```
   or, for Codex:
   ```bash
   cmux new-workspace --cwd "$PWD" --command 'codex "$(cat /tmp/cmux-agent-prompt.ab12cd.md)"' --focus true
   ```
   Use only the invocation matching the detected agent, with the real path from step 2 substituted in. `--focus true` switches to the new workspace immediately; omit it (or pass `false`) to leave the caller's workspace focused.

Do not fall back to create-then-`send`-then-`send-key enter` — see "Why this is one call" above. Do not go back to typing the prompt text directly into `--command` — see "Why the prompt goes through a file" above.
