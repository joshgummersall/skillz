---
name: cmux-split-agent
description: Open a new instance of whichever coding agent CLI (claude, codex, ...) is active in the current cmux surface, in a new split pane, immediately running a given prompt or skill with the syntax appropriate to that agent. Use when the user asks to "open claude/codex in a split" or "run X in a new cmux pane".
---

# cmux Split + Agent

Launches a fresh coding-agent CLI session in a new cmux split, pre-loaded with a prompt. Mirrors whatever agent is currently running the caller's session (claude, codex, ...) rather than assuming `claude`.

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

## Why this needs four calls, not one

cmux has no single call that creates a split and runs a command in it:

- `new-split`/`new-pane --command` is documented in the `cmux` skill's reference docs, but as of cmux 0.64.22 that flag is only wired up for `new-workspace` — `new-split --help` and `new-pane --help` don't list it, and passing it errors with `unknown flag`.
- The socket-level `initial_input` param (on `surface.split` etc.) races the new shell's startup — text sent this way can be silently dropped before the shell is ready to read it. Verified via `read-screen` showing no output after using it.

So the reliable sequence is: create an empty split, write the prompt to a file, `send` a short command that reads that file, then `send-key enter`, as separate steps once the surface exists.

## Why the prompt goes through a file, not the command line

`cmux send` types its argument as literal keystrokes into the destination shell (zsh, bash, ...), exactly as if the user had typed it — so the prompt text gets parsed twice: once by whatever quoting you used to pass it to `cmux send`, and again by the destination shell reading it off the terminal. There is no single escaping scheme that survives both. Concretely, `is there a way to avoid casting?` typed into zsh trips glob expansion on the bare `?` (`zsh: no matches found: ...casting?`) even though the outer single-quoting was correct for the calling shell; content with embedded `"`, `&`, backticks, or `$(...)` has the same problem at the destination shell regardless of how carefully it was escaped for the caller.

Sidestep this by never typing the prompt text itself: write it to a file with the `Write` tool (a tool parameter, not shell text — no escaping applies) and have the destination shell pull it in via command substitution. Only a short, fixed, character-safe wrapper is ever typed:

```
claude "$(cat /tmp/cmux-agent-prompt.ab12cd.md)"
```

The file path is the only variable part of that wrapper. Generate it with `mktemp` so it can't collide with a concurrent invocation, and it will contain no spaces or quote characters, so the wrapper never needs prompt-specific escaping.

## Steps

1. Determine the agent command: `echo "$CMUX_AGENT_LAUNCH_KIND"` (see above).
2. Get the caller's current surface: `cmux identify --json` → `caller.surface_ref`.
3. Split off that surface (default direction `right`, override if the user says otherwise):
   ```bash
   cmux new-split right --surface <caller_surface_ref> --focus true
   ```
   This returns the new surface's ref, e.g. `OK surface:8 workspace:3`.
4. Normalize the skill prefix as above, then create a unique temp file and write the exact agent invocation text into it with the `Write` tool (not a shell heredoc/echo — see above for why):
   ```bash
   mktemp "${TMPDIR:-/tmp}/cmux-agent-prompt.XXXXXX.md"
   ```
   Write the invocation text verbatim as the file's entire content, e.g. `/thermo-nuclear-code-quality-review` (Claude), `$thermo-nuclear-code-quality-review` (Codex), or the raw free-text prompt.
5. Type the wrapper command into the new surface. Only the fixed wrapper and the mktemp'd path appear in this command, so plain single-quoting for the calling shell is always sufficient — no per-prompt escaping is needed:
   ```bash
   cmux send --surface <new_surface_ref> 'claude "$(cat /tmp/cmux-agent-prompt.ab12cd.md)"'
   cmux send --surface <new_surface_ref> 'codex "$(cat /tmp/cmux-agent-prompt.ab12cd.md)"'
   ```
   Use only the invocation matching the detected agent, with the real path from step 4 substituted in.
6. Press enter to execute it:
   ```bash
   cmux send-key --surface <new_surface_ref> enter
   ```

Do not try to collapse this into one `new-split --command` call — see "Why this needs four calls" above. Do not go back to typing the prompt text directly into the `send` call — see "Why the prompt goes through a file" above.
