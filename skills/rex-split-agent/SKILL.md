---
name: rex-split-agent
description: Open a new instance of whichever coding agent CLI (claude, codex, ...) is running in the current Rex pane, in a new split pane, immediately running a given prompt or skill with the syntax appropriate to that agent. Use when the user asks to "open claude/codex in a split" or "run X in a new rex pane".
---

# Rex Split + Agent

Launches a fresh coding-agent CLI session in a new Rex split, pre-loaded with a prompt. Mirrors the agent running the caller's session (claude, codex, ...) rather than assuming `claude`. Uses the `rex` CLI; see the `/rex` skill for general usage.

`ARGUMENTS` is the skill invocation or free-text prompt to run (e.g. `$thermo-nuclear-code-quality-review`, `/thermo-nuclear-code-quality-review`, or `review this diff for security issues`).

## Detecting the active agent

Rex does not tag terminals with the agent in use. Infer it from the caller's own environment:

- `CLAUDECODE=1` means `claude`.
- Otherwise, if `CODEX_*` variables are set (`env | grep ^CODEX`), use `codex`.
- Otherwise ask the user which agent to open rather than guessing.

Do not hardcode `claude`.

## Adapting skill invocations

Normalize a requested skill invocation for the detected agent, regardless of the prefix the user supplied:

- Claude: `/skill-name`
- Codex: `$skill-name`

Preserve the skill name and any trailing arguments. Pass free-text prompts through unchanged; do not rewrite arbitrary dollar signs, paths, or built-in slash commands as skills. For other agents, preserve the supplied invocation unless their skill syntax is known.

## Passing the prompt

Rex starts the command directly from argv (everything after `--`, as separate words). The prompt is never typed into a shell, so it is parsed only once, by the calling shell. Capture it literally with a quoted heredoc, then pass it as one quoted word:

```bash
PROMPT=$(cat <<'PROMPT_EOF'
<verbatim prompt text goes here, completely unescaped>
PROMPT_EOF
)
rex split -s "$REX_SESSION" -b "$REX_BLOCK" --split=right -c "<cwd>" --label "<title>" --json -- claude "$PROMPT"
```

The quoted delimiter disables all expansion, so `?`, `"`, `&`, backticks, and `$(...)` survive intact. Pick a delimiter unlikely to collide with the prompt.

## Working directory

Optional. By default use the caller's working directory (`$PWD`). Use a different directory only when the user names one (for example, "in ~/src/other-repo") or the task clearly belongs to another repo or worktree.

- Resolve `~` and relative paths to an absolute path first. Confirm it exists with `test -d`; if not, ask the user instead of guessing.
- `ARGUMENTS` stays the prompt only. Do not parse a directory out of free text unless the user clearly stated one.

Always pass it with `-c`. No `cd` is needed.

## Model

Optional. By default the new agent uses its own default model. Pass a model only when the user names one (for example, "with opus" or "using gpt-5").

- Claude: `claude --model <model> "$PROMPT"`.
- Codex: `codex -m <model> "$PROMPT"`.
- Pass the user's model name or alias through unchanged. Do not guess or expand it.
- `ARGUMENTS` stays the prompt only. Do not parse a model out of free text unless the user clearly stated one.
- For other agents, ask the user for the flag rather than guessing.

## Titling

Pass `--label "<title>"` to name the new pane. Infer the title from the prompt: 2-4 words, Title Case, no quotes or trailing punctuation (for example, `Security Review`, `Fix Login Redirect`). For a bare skill invocation, derive it from the skill name (`/thermo-nuclear-code-quality-review` becomes `Code Quality Review`).

## Steps

1. Determine the agent command (see above).
2. Pick the working directory, the model flag if any, and the title.
3. Normalize the skill prefix, then run the heredoc and `rex split` in one `Bash` call (default direction `right`; override with `--split` if the user says otherwise):
   ```bash
   PROMPT=$(cat <<'PROMPT_EOF'
   /thermo-nuclear-code-quality-review
   PROMPT_EOF
   )
   rex split -s "$REX_SESSION" -b "$REX_BLOCK" --split=right -c "/abs/path/to/cwd" --label "Code Quality Review" --json -- claude "$PROMPT"
   ```
   Use only the invocation matching the detected agent. The new pane takes focus; pass `--focus=false` to keep the caller focused. The JSON result's `.block_ids[0]` identifies the new pane.

Do not create a shell pane and then `rex send` the prompt into it. Do not interpolate the prompt text directly into the command line.
