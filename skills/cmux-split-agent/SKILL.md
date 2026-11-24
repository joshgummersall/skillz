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

## Finding the caller's surface

`cmux identify --json` reads `CMUX_SURFACE_ID` from the environment. A running process cannot update its environment, so after the user drags the pane into another workspace, or restores the session, the ids go stale. Then `caller` is `null` (or `caller.surface_ref` is `null`), and `--surface null` would fail or hit the wrong pane. The `focused` block is where the user is looking, not necessarily where this agent runs, so do not use it as a substitute.

1. Run `cmux identify --json`. If `caller.surface_ref` is a non-null ref, use it.
2. Otherwise find the surface by terminal device, which a stale environment cannot affect. Walk up the process tree to the first process with a real tty, then match it against `cmux tree --all`:
   ```bash
   pid=$$; tty=
   while [ "$pid" -gt 1 ]; do
     tty=$(ps -o tty= -p "$pid" | tr -d ' ')
     if [ -n "$tty" ] && [ "$tty" != "??" ]; then break; fi
     pid=$(ps -o ppid= -p "$pid" | tr -d ' ')
   done
   cmux tree --all 2>/dev/null | grep -E "tty=$tty( |$)" | sed -E 's/.*(surface:[0-9]+).*/\1/'
   ```
   Use the single ref this prints. If it prints nothing or more than one line, stop.
3. If neither works, stop and tell the user the caller surface could not be resolved (likely a stale `CMUX_SURFACE_ID` after moving the pane). Do not split off the focused surface, and do not strip `CMUX_*` variables from the new agent's command: cmux hooks may need them.

Observed with a stale environment (cmux 0.65.2): an explicit `--surface <ref>` worked for `cmux read-screen`, but `cmux open <file>` failed with `not_found: Source surface not found` even with `--surface`. `cmux open` worked under `env -u CMUX_SURFACE_ID -u CMUX_PANEL_ID -u CMUX_TAB_ID cmux open <file>`. `new-split --surface` was not tested with a stale environment.

## One call, not three

`new-split` accepts `--command <text>`: cmux starts the new pane's normal interactive shell and delivers the text plus one Enter at spawn time, so the command runs immediately and no follow-up `send`/`send-key enter` is needed — confirmed working directly (`cmux new-split right --surface <ref> --command "echo hi" --focus true` prints `hi` with no further calls). An older version of this skill claimed `--command` was only wired up for `new-workspace` and that `new-split`/`new-pane` rejected it with `unknown flag`; that is no longer true (and is contradicted by the `cmux` skill's own docs) — don't resurrect the create-then-`send`-then-`send-key enter` dance.

## Why the prompt is built via a quoted heredoc, not inline-quoted directly

`--command`'s value is delivered as literal keystrokes into the new pane's shell (zsh, bash, ...), exactly as if the user had typed it — so the prompt text is parsed twice: once by whatever quoting gets it into the `cmux new-split` call, and again by that destination shell reading it off the terminal. There is no single escaping scheme that survives both. Concretely, `is there a way to avoid casting?` typed into zsh trips glob expansion on the bare `?` (`zsh: no matches found: ...casting?`) even though the outer quoting was correct for the calling shell; content with embedded `"`, `&`, backticks, or `$(...)` has the same problem at the destination shell regardless of how carefully it was escaped for the caller.

A temp file avoids that but trades it for its own portability trap: BSD/macOS `mktemp` only substitutes a trailing `XXXXXX` run when it is the very last thing in the template, so `mktemp foo.XXXXXX.md` silently creates a file named literally `foo.XXXXXX.md` instead of a random one (GNU `mktemp` handles this fine, so it works on Linux and breaks silently on macOS) — plus it leaves a file to clean up.

Avoid both problems with a quoted heredoc, built in a single `Bash` tool call so no intermediate file or escaping is ever needed. A heredoc whose delimiter is single-quoted (`<<'PROMPT_EOF'`) disables **all** expansion inside its body — `$`, backticks, `"`, `!`, everything is captured completely literally, which is exactly what arbitrary prompt text needs. Wrap the whole thing in an outer quoted heredoc to capture it into a shell variable in one shot, then pass that variable (double-quoted, to preserve its embedded newlines) as `--command`:

```bash
PAYLOAD=$(cat <<'OUTER_EOF'
cd '<cwd>' && claude "$(cat <<'PROMPT_EOF'
<verbatim prompt text goes here, completely unescaped>
PROMPT_EOF
)"
OUTER_EOF
)
cmux new-split right --surface <caller_surface_ref> --command "$PAYLOAD" --focus true
```

The outer heredoc is pure text capture at the calling layer — the inner `<<'PROMPT_EOF'` inside it is never executed there, only captured as literal characters. When `--command`'s value is typed into the new pane's shell, *that* shell is the one that actually runs the inner heredoc, parsing it exactly once. Pick delimiters unlikely to collide with the prompt content (a fixed distinctive string is normally enough; append the caller's `$$` if you want extra safety) — the only failure mode is a prompt that happens to contain a line identical to the delimiter.

## Working directory

Optional. By default the new agent starts in the caller's working directory (the directory this session is running in). Claude may instead pass a distinct directory when the user names one (for example, "in ~/src/other-repo") or the task clearly belongs to another repo or worktree.

- Resolve `~` and relative paths to an absolute path first. Confirm it exists with `test -d`; if not, ask the user instead of guessing.
- `ARGUMENTS` stays the prompt only. Do not parse a directory out of free text unless the user clearly stated one.

`new-split` has no `--cwd` flag, so the directory is applied by prefixing the command with `cd <dir> &&`. Always include the `cd`, even for the default, so the result does not depend on what the split inherits.

## Model

Optional. By default the new agent uses its own default model. Pass a model only when the user names one (for example, "with opus" or "using gpt-5").

- Claude: `claude --model <model> "$(...)"`.
- Codex: `codex -m <model> "$(...)"`.
- Pass the user's model name or alias through unchanged. Do not guess or expand it.
- `ARGUMENTS` stays the prompt only. Do not parse a model out of free text unless the user clearly stated one.
- For other agents, ask the user for the flag rather than guessing.

## Titling

`new-split` has no name flag, so rename the new surface's tab after creating it. Infer the title from the prompt: 2-4 words, Title Case, no quotes or trailing punctuation (for example, `Security Review`, `Fix Login Redirect`). For a bare skill invocation, derive it from the skill name (`/thermo-nuclear-code-quality-review` becomes `Code Quality Review`). The title is a plain short string, so inline quoting is safe.

## Steps

1. Determine the agent command: `echo "$CMUX_AGENT_LAUNCH_KIND"` (see above).
2. Pick the working directory (see above): the session's own directory by default, or the distinct one requested.
   If the user named a model, add its flag after the agent command (see "Model" above).
3. Get the caller's current surface (see "Finding the caller's surface" above). Do not continue with a null or guessed ref.
4. Normalize the skill prefix as above, then build `$PAYLOAD` and split off the caller's surface in one `Bash` call, using the quoted double-heredoc pattern above (default direction `right`, override if the user says otherwise):
   ```bash
   PAYLOAD=$(cat <<'OUTER_EOF'
   cd '/abs/path/to/cwd' && claude "$(cat <<'PROMPT_EOF'
   /thermo-nuclear-code-quality-review
   PROMPT_EOF
   )"
   OUTER_EOF
   )
   cmux new-split right --surface <caller_surface_ref> --command "$PAYLOAD" --focus true
   ```
   Use only the invocation matching the detected agent (`claude`/`codex`) inside `$PAYLOAD`. This returns the new surface's ref, e.g. `OK surface:8 workspace:3`.
5. Title the new tab with the surface ref from step 4's output:
   ```bash
   cmux rename-tab --surface surface:8 "<title>"
   ```

Do not fall back to create-then-`send`-then-`send-key enter` — see "One call, not three" above. Do not go back to typing the prompt text directly into `--command`, or through an intermediate file — see "Why the prompt is built via a quoted heredoc" above.
