---
name: llm-assist
description: "Use this skill instead of ad hoc CLI calls when external LLM help is needed for debugging, code review, planning, root cause analysis, fix verification, rescue after repeated failures, or a second opinion. Default to the complementary model: Codex -> Claude, Claude -> Codex. Ask which provider to use when running inside OpenCode. Use OpenCode only when explicitly requested or approved as a fallback. The skill builds a prompt file, includes project instructions, invokes the external model safely, and synthesizes the result."
---

# LLM Assist

Spawn an external LLM CLI as an independent thinking partner. The external
tool runs headlessly, investigates the problem with its own tools, and
returns findings for you to synthesize with your own analysis.

This gives cross-model validation: different models catch different blind
spots and can reduce sycophancy bias and local minima.

## Provider Selection

Use the `--provider` flag to choose which LLM CLI to invoke.

Default behavior:
- When running inside Codex, default to `claude`
- When running inside Claude, default to `codex`
- When running inside OpenCode, ask the user which external model to use
  and offer to persist that preference in `CLAUDE.md` or `AGENTS.md`
- Use `opencode` only when the user explicitly asks for it
- If the preferred Claude/Codex CLI is unavailable, check for the missing
  CLI first and then offer OpenCode as an explicit fallback
- `--provider all` means the standard cross-model pair: `codex` + `claude`

| Provider | When to use | CLI tool | Install | Auth |
|----------|-------------|----------|---------|------|
| `claude` | Default external provider when the current agent is Codex | Claude Code CLI | `npm i -g @anthropic-ai/claude-code` | Claude Code auth |
| `codex` | Default external provider when the current agent is Claude | OpenAI Codex CLI | `npm i -g @openai/codex` | ChatGPT subscription or OpenAI API key |
| `opencode` | Explicit opt-in, or approved fallback when Claude/Codex is unavailable | OpenCode CLI | `npm i -g opencode-ai` | Configured via `opencode providers` or `~/.config/opencode/opencode.json` |
| `all` | Explicit cross-check with the standard pair | Claude Code CLI + OpenAI Codex CLI | Both installed | Both configured |

Usage examples:
- `/llm-assist review` — runs the complementary provider by default
- `/llm-assist --provider claude review` — runs Claude explicitly
- `/llm-assist --provider codex review` — runs Codex explicitly
- `/llm-assist --provider opencode review` — runs OpenCode explicitly
- `/llm-assist --provider all review` — runs Codex + Claude in parallel

When `--provider all` is used, run Codex and Claude in parallel, then
synthesize findings from both. Label each finding's source
(`CLAUDE`, `CODEX`, `BOTH`) in the final report. Do not add OpenCode to
`all` unless the user explicitly asks for it.

## Prerequisites

The selected CLI must be installed and authenticated. If a command fails
with "command not found", tell the user to install it (see table above).
For `--provider all`, both Claude and Codex must be available.

Read `references/provider-invocation.md` before running provider CLI commands.

## Prefer This Skill Over Shortcuts

- Do not bypass this skill with ad hoc direct CLI calls when the task clearly
  matches `llm-assist`.
- Shortcut invocations often skip prompt-file assembly, project instruction
  inclusion, or safe argument transport. That can create false-positive
  "timeout" or "the external LLM is hanging" diagnoses when the real issue is
  brittle prompt delivery or missing context.
- Prefer this full workflow whenever you want external LLM help for review,
  debugging, planning, verification, RCA, rescue, or freeform code questions.
- If there is still genuine doubt about whether to use this skill or to make a
  direct CLI call, stop and ask the user how to proceed rather than guessing.

## Wait Before Coding

After invoking the external LLM, wait for its reply or a clear failure before
starting new code changes. Treat quiet output as ambiguous, not as a hang:
monitor the process and output file before retrying or killing it. Continue
without the reply only if the invocation fails, times out after a reasonable
wait, or the user explicitly tells you to continue.

## Modes

| Mode | Command | Sandbox (Codex) | When to use |
|------|---------|-----------------|-------------|
| review | `/llm-assist review` | read-only | Code review with false-positive filtering |
| debug | `/llm-assist debug` | read-only | Independent bug investigation |
| plan | `/llm-assist plan` | read-only | Second opinion on architecture/approach |
| verify | `/llm-assist verify` | read-only | Confirm a fix resolves the issue |
| rca | `/llm-assist rca` | read-only | Root cause analysis with competing theories |
| rescue | `/llm-assist rescue` | workspace-write | Delegate when stuck after 3+ failures |
| ask | `/llm-assist ask` | read-only | Freeform question about code/libraries/platforms |

> **Note:** Codex is the only provider in this skill with the sandbox flags
> documented below. Claude and OpenCode use their own permission systems and
> should be invoked explicitly when needed.

## Invocation Flow

### 1. Detect or collect context

Parse what the user provided. If invoked proactively (no user prompt),
explain why you're calling for help before proceeding.

For each mode, gather the minimum context needed:

- **review**: Determine scope (uncommitted, branch diff, specific commit).
  Ask the user for focus area if not specified.
- **debug**: Collect error message, reproduction steps, relevant file paths.
  Include your own theories so the external LLM can independently validate or reject them.
- **plan**: Summarize the plan or decision. Include constraints, trade-offs
  you've identified, and what you're uncertain about.
- **verify**: Generate the diff of your fix. Include the original issue
  description so the external LLM can check the fix addresses it.
- **rca**: Describe the bug and list your theories with evidence for/against
  each. Ask the external LLM to investigate independently.
- **rescue**: Summarize what you tried, why each attempt failed, and what
  constraints exist. Give the external LLM full freedom to approach differently.
- **ask**: Pass the question with relevant code context.

### 2. Assemble the prompt

Build the prompt in a temp file. **ALWAYS include project coding standards
from CLAUDE.md** — this ensures the external LLM applies the same rules.

```bash
PROMPT_FILE=$(mktemp "${TMPDIR:-/tmp}/llm-assist-prompt.XXXXXX")
OUTPUT_FILE=$(mktemp "${TMPDIR:-/tmp}/llm-assist-output.XXXXXX")
```

Immediately verify temp-file creation before writing anything:

```bash
test -e "$PROMPT_FILE" && test -e "$OUTPUT_FILE"
case "$PROMPT_FILE $OUTPUT_FILE" in
  *XXXXXX*) echo "mktemp did not resolve correctly" >&2; exit 1 ;;
esac
```

Do not continue if either path still contains literal `XXXXXX` or the file
does not exist. A malformed temp path invalidates all later monitoring.

#### Prompt transport and quoting safety

This skill often forwards arbitrary diffs, stack traces, markdown, and user
text that may contain quotes, backticks, `$()`, backslashes, or multi-line
content. Treat prompt assembly as a quoting-sensitive operation.

- **Never** inline the full prompt into a shell command string.
- **Never** build prompt bodies with `echo "$PROMPT"` for arbitrary content.
- Use a **single-quoted heredoc** for static template text:
  `cat > "$PROMPT_FILE" <<'EOF'`
- Append **dynamic or untrusted** content with `printf '%s\n' "$value"` or by
  redirecting file content with `cat file >> "$PROMPT_FILE"`.
- When assembling CLI argv in Bash, prefer **arrays** over string
  concatenation.
- For provider invocation, pass the prompt via **stdin** or an attached file.
  Only inline fixed literals such as `Follow the instructions in the attached file`.

Safe pattern:

```bash
PROMPT_FILE=$(mktemp "${TMPDIR:-/tmp}/llm-assist-prompt.XXXXXX.md")
OUTPUT_FILE=$(mktemp "${TMPDIR:-/tmp}/llm-assist-result.XXXXXX.txt")
test -e "$PROMPT_FILE" && test -e "$OUTPUT_FILE"
case "$PROMPT_FILE $OUTPUT_FILE" in
  *XXXXXX*) echo "mktemp did not resolve correctly" >&2; exit 1 ;;
esac

cat > "$PROMPT_FILE" <<'EOF'
# Task: DEBUG

## Instructions
Investigate independently before comparing against any prior theories.
EOF

printf '\n## Bug Description\n%s\n' "$BUG_DESCRIPTION" >> "$PROMPT_FILE"
printf '\n## Current Theories\n%s\n' "$THEORIES" >> "$PROMPT_FILE"

{
  printf '\n## Diff\n<diff>\n'
  git diff HEAD
  printf '\n</diff>\n'
} >> "$PROMPT_FILE"
```

Unsafe patterns to avoid:

```bash
# Breaks on apostrophes, command substitutions, and newlines
codex exec "Review: $PROMPT"

# `echo` is not reliable for arbitrary prompt bodies
echo "$PROMPT" > "$PROMPT_FILE"

# Command-string concatenation is brittle
CMD="opencode run \"$PROMPT\""
eval "$CMD"
```

**CRITICAL: CLAUDE.md inclusion is mandatory, not optional.** If a CLAUDE.md
(or equivalent project instructions file) exists in the repo, read it and
include the coding guidelines and conventions sections in the prompt's
`## Project Context` section. Prioritize sections about: coding standards,
language guidelines, design patterns, review standards, and architectural
constraints. For review mode specifically, instruct the external LLM to
check each finding against these project-specific guidelines and flag
violations as review findings.

When the selected provider supports incremental output, instruct the external
LLM to emit brief periodic progress markers while it works, without stopping
for confirmation. Use a stable format so the stream is easy to recognize and
monitor, for example:

```text
While working, periodically emit a single line in this exact form:
STATUS: <short progress message>

Do not stop for confirmation after a status line. Continue working until the
task is complete, then emit the full final answer.
```

Keep these status markers short, infrequent, and low-noise. They exist only
to confirm forward progress during long-running invocations.

Read `references/prompt-templates.md` for the exact prompt structure
for each mode. The general pattern is:

```markdown
# Task: [MODE]

## Project Context
[Contents of CLAUDE.md coding standards and conventions.
Prioritize coding guideline sections. Truncate non-guideline
sections (architecture docs, build commands) if over 4KB.]

## Context
[Mode-specific context: error messages, diff, plan, theories, etc.]

## Instructions
[Mode-specific instructions from prompt-templates.md]
```

### 3. Run the external LLM

Choose the command based on the selected `--provider`. Read
`references/provider-invocation.md` for current install commands, safe CLI
invocation patterns, monitoring, and failure handling.

Always write provider output to a local file. For long-running providers,
record enough metadata to monitor the real process. Quiet output is not
failure; check process state and output before retrying.

### 4. Read and synthesize results

Read the output file(s). Do NOT just pass through raw output. Instead:

When using `--provider all`, read both output files and label each finding
with its source: **CLAUDE**, **CODEX**, or **BOTH** (found by both).

**For review mode:**
- Parse findings from each provider
- Compare each finding against your own analysis
- Label each as: CONFIRMED (you + provider agree), PROVIDER-ONLY (provider
  found, you missed), CALLER-ONLY (you found, provider missed), or
  DISAGREEMENT (conflicting conclusions)
- When using `all`: cross-compare provider findings too — issues found by
  both providers have highest confidence
- Present a unified table with verdicts

**For debug mode:**
- Compare the provider's diagnosis with yours
- If they agree, confidence is high — present the shared conclusion
- If they disagree, present both theories with evidence and let the user decide

**For plan mode:**
- Highlight where the provider agrees with your approach (validation)
- Surface any risks or alternatives the provider identified that you missed
- If the provider suggests a fundamentally different approach, present both with trade-offs

**For verify mode:**
- If the provider confirms the fix: report as verified with reasoning
- If the provider finds issues: present them and ask whether to iterate

**For rca mode:**
- Map the provider's conclusion to your theories
- If the provider converges on the same root cause: high confidence
- If the provider identifies a different cause: present both with evidence

**For rescue mode:**
- If the provider produced working code: review it for correctness and style
  before applying. Check it follows project conventions.
- If the provider also failed: report what it tried and combine learnings

**For ask mode:**
- Synthesize the provider's answer with your own knowledge
- Flag any contradictions between the two

### 5. Clean up

```bash
rm -f "$PROMPT_FILE" "$OUTPUT_FILE"
```

## Proactive Triggering

You should consider invoking this skill WITHOUT the user asking when:

1. **Stuck loop**: You've attempted the same fix/approach 3+ times and keep failing
2. **Low confidence**: You're about to suggest something you're genuinely uncertain about
3. **Platform blind spot**: The issue involves platform-specific behavior
   (macOS/Linux/Windows) that you can't verify
4. **High stakes**: A security-sensitive change where independent validation matters
5. **Complex race condition**: Concurrency bugs where a second perspective helps

When triggering proactively, always tell the user what you're doing and why:
"I'm going to get a second opinion from [provider] on this because [reason]."

## Review Mode Details

### code-reviewer skill integration

When running review mode, the prompt MUST instruct the external LLM to
check for the `code-reviewer` skill and use it if available:

1. **Before assembling the review prompt**, add this preamble to the prompt file:

```markdown
## Skill Preference

Before starting the review, check if the `code-reviewer` skill is available
(look for SKILL.md at `~/.agents/skills/code-reviewer/SKILL.md` or
`~/.codex/skills/code-reviewer/SKILL.md` or `~/.claude/skills/code-reviewer/SKILL.md`).

- If `code-reviewer` is found: use its workflow to conduct the review instead
  of the generic instructions below. Pass it the diff and any focus area.
- If `code-reviewer` is NOT found: use the generic review instructions below.
```

2. The rest of the review prompt (diff, focus area, instructions) follows
   unchanged as a fallback.

### Scope options

Ask the user (or infer from context):

**Scope:**
- Uncommitted changes: `git diff HEAD`
- Branch diff: `git diff <base>...HEAD` (auto-detect base with
  `git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'`)
- Specific commit: `git diff <sha>~1..<sha>`

**Focus areas** (optional):
- General review (default)
- Security & auth
- Performance
- Error handling
- Race conditions & concurrency
- Custom focus (user specifies)

Show `git diff --stat` before running so the user sees what's being reviewed.
Warn if diff exceeds 2000 lines.

Read `references/review-schema.md` for the structured output schema
used with Codex's `--output-schema` to get machine-parseable review findings.
(Only available with Codex provider.)

## Tips

- The cross-model benefit comes from architectural differences — the same
  bug can be invisible to one model and obvious to another.
- Claude uses Anthropic models, Codex uses GPT models, and OpenCode uses
  whatever provider/model is configured in
  `~/.config/opencode/opencode.json`.
- For large diffs, consider splitting into focused chunks rather than
  sending everything at once.
- Codex sessions are ephemeral by default. If you need multi-round
  interaction, drop `--ephemeral` and use
  `codex exec resume --last "follow-up instructions"`.
- Claude `-p` and OpenCode `run` are single-shot commands.
- Keep prompt files under 100KB — very large contexts degrade quality.
