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

## Why the prompt goes through a quoted heredoc, not inline quoting or a file

`--command`'s value gets typed as literal keystrokes into the new workspace's shell (zsh, bash, ...), exactly as if the user had typed it — so the prompt text is parsed twice: once by whatever quoting got it into the `cmux new-workspace` call, and again by that destination shell reading it off the terminal. No single escaping scheme survives both. Concretely, `is there a way to avoid casting?` breaks zsh's glob expansion on the bare `?` (`zsh: no matches found: ...casting?`) no matter how correctly the outer quoting was done for the calling shell; embedded `"`, `&`, backticks, or `$(...)` fail the same way.

A temp file avoids that but trades it for its own portability trap: BSD/macOS `mktemp` only substitutes a trailing `XXXXXX` run when it is the very last thing in the template, so `mktemp foo.XXXXXX.md` silently creates a file named literally `foo.XXXXXX.md` instead of a random one (GNU `mktemp` handles this fine, so it works on Linux and breaks silently on macOS) — plus it leaves a file to clean up.

Avoid both problems with a quoted heredoc, built in a single `Bash` tool call so no intermediate file or escaping is ever needed. A heredoc whose delimiter is single-quoted (`<<'PROMPT_EOF'`) disables **all** expansion inside its body — `$`, backticks, `"`, `!`, everything is captured completely literally, which is exactly what arbitrary prompt text needs. Wrap the whole thing in an outer quoted heredoc to capture it into a shell variable in one shot, then pass that variable (double-quoted, to preserve its embedded newlines) as `--command`:

```bash
PAYLOAD=$(cat <<'OUTER_EOF'
claude "$(cat <<'PROMPT_EOF'
<verbatim prompt text goes here, completely unescaped>
PROMPT_EOF
)"
OUTER_EOF
)
cmux new-workspace --cwd "$PWD" --command "$PAYLOAD" --focus true
```

The outer heredoc is pure text capture at the calling layer — the inner `<<'PROMPT_EOF'` inside it is never executed there, only captured as literal characters. When `--command`'s value is typed into the new workspace's shell, *that* shell is the one that actually runs the inner heredoc, parsing it exactly once. Pick delimiters unlikely to collide with the prompt content (a fixed distinctive string is normally enough; append the caller's `$$` if you want extra safety) — the only failure mode is a prompt that happens to contain a line identical to the delimiter.

## Steps

1. Determine the agent command: `echo "$CMUX_AGENT_LAUNCH_KIND"` (see above).
2. Normalize the skill prefix as above, then build `$PAYLOAD` in one `Bash` call using the quoted double-heredoc pattern above, substituting `claude` or `codex` for the agent and the normalized invocation text (verbatim, no escaping) for `<verbatim prompt text goes here...>`:
   ```bash
   PAYLOAD=$(cat <<'OUTER_EOF'
   claude "$(cat <<'PROMPT_EOF'
   /thermo-nuclear-code-quality-review
   PROMPT_EOF
   )"
   OUTER_EOF
   )
   ```
3. Create the workspace with `$PAYLOAD` as its initial input, keeping the caller's working directory so the agent starts with the same project context:
   ```bash
   cmux new-workspace --cwd "$PWD" --command "$PAYLOAD" --focus true
   ```
   Use only the invocation matching the detected agent (`claude`/`codex`) inside `$PAYLOAD`. `--focus true` switches to the new workspace immediately; omit it (or pass `false`) to leave the caller's workspace focused.

Do not fall back to create-then-`send`-then-`send-key enter` — see "Why this is one call" above. Do not go back to typing the prompt text directly into `--command`, or through an intermediate file — see "Why the prompt goes through a quoted heredoc" above.
