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

## Why this needs three calls, not one

cmux has no single call that creates a split and runs a command in it:

- `new-split`/`new-pane --command` is documented in the `cmux` skill's reference docs, but as of cmux 0.64.22 that flag is only wired up for `new-workspace` — `new-split --help` and `new-pane --help` don't list it, and passing it errors with `unknown flag`.
- The socket-level `initial_input` param (on `surface.split` etc.) races the new shell's startup — text sent this way can be silently dropped before the shell is ready to read it. Verified via `read-screen` showing no output after using it.

So the reliable sequence is: create an empty split, then `send` the command text, then `send-key enter`, as separate steps once the surface exists.

## Steps

1. Determine the agent command: `echo "$CMUX_AGENT_LAUNCH_KIND"` (see above).
2. Get the caller's current surface: `cmux identify --json` → `caller.surface_ref`.
3. Split off that surface (default direction `right`, override if the user says otherwise):
   ```bash
   cmux new-split right --surface <caller_surface_ref> --focus true
   ```
   This returns the new surface's ref, e.g. `OK surface:8 workspace:3`.
4. Normalize the skill prefix as above, then type the agent invocation into the new surface. Quote for both shells: first single-quote the prompt for the destination shell, escaping each embedded `'` as `'\''`; then shell-quote the entire command text for the calling shell. Inner single quotes inside outer double quotes do not protect `$`, backticks, or command substitutions from the calling shell. These examples preserve the literal Codex `$`:
   ```bash
   cmux send --surface <new_surface_ref> 'codex '\''$thermo-nuclear-code-quality-review'\'''
   cmux send --surface <new_surface_ref> 'claude '\''/thermo-nuclear-code-quality-review'\'''
   ```
   Use only the invocation matching the detected agent. When building commands programmatically, use a shell-quoting function for each layer; JSON string encoding is not shell quoting.
5. Press enter to execute it:
   ```bash
   cmux send-key --surface <new_surface_ref> enter
   ```

Do not try to collapse this into one `new-split --command` call — see "Why this needs three calls" above.
