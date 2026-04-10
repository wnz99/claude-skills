---
name: codex-assist
description: "Spawn OpenAI Codex CLI as a cross-model thinking partner for debugging, code review, planning, root cause analysis, fix verification, and rescue when stuck. Use this skill whenever you need a second opinion from a different LLM, want to cross-validate findings, are stuck after multiple failed attempts, need independent verification of a fix, want to compare review results to filter false positives, or need a fresh perspective on architecture/planning decisions. Also trigger proactively when: you've failed the same approach 3+ times, a code review would benefit from cross-model validation, you're unsure about platform-specific behavior you can't test, or the user explicitly asks for a second opinion, codex review, or external help."
---

# Codex Assist

Spawn OpenAI Codex CLI as an independent thinking partner. Codex runs
headlessly via `codex exec`, investigates the problem with its own
tools, and returns findings for you to synthesize with your own analysis.

This gives cross-model validation — different models catch different
blind spots, eliminating sycophancy bias and local minima.

## Prerequisites

Codex CLI must be installed and authenticated. If a command fails with
"command not found", tell the user:

```
npm i -g @openai/codex
```

Then they need a ChatGPT subscription or OpenAI API key configured.

## Modes

| Mode | Command | Sandbox | When to use |
|------|---------|---------|-------------|
| review | `/codex-assist review` | read-only | Code review with false-positive filtering |
| debug | `/codex-assist debug` | read-only | Independent bug investigation |
| plan | `/codex-assist plan` | read-only | Second opinion on architecture/approach |
| verify | `/codex-assist verify` | read-only | Confirm a fix resolves the issue |
| rca | `/codex-assist rca` | read-only | Root cause analysis with competing theories |
| rescue | `/codex-assist rescue` | workspace-write | Delegate when stuck after 3+ failures |
| ask | `/codex-assist ask` | read-only | Freeform question about code/libraries/platforms |

## Invocation Flow

### 1. Detect or collect context

Parse what the user provided. If invoked proactively (no user prompt),
explain why you're calling for help before proceeding.

For each mode, gather the minimum context needed:

- **review**: Determine scope (uncommitted, branch diff, specific commit).
  Ask the user for focus area if not specified.
- **debug**: Collect error message, reproduction steps, relevant file paths.
  Include your own theories so Codex can independently validate or reject them.
- **plan**: Summarize the plan or decision. Include constraints, trade-offs
  you've identified, and what you're uncertain about.
- **verify**: Generate the diff of your fix. Include the original issue
  description so Codex can check the fix addresses it.
- **rca**: Describe the bug and list your theories with evidence for/against
  each. Ask Codex to investigate independently.
- **rescue**: Summarize what you tried, why each attempt failed, and what
  constraints exist. Give Codex full freedom to approach differently.
- **ask**: Pass the question with relevant code context.

### 2. Assemble the prompt

Build the prompt in a temp file. **ALWAYS include project coding standards
from CLAUDE.md** — this ensures Codex applies the same rules as the project.

```bash
PROMPT_FILE=$(mktemp /tmp/codex-assist-XXXXXX)
OUTPUT_FILE=$(mktemp /tmp/codex-result-XXXXXX)
```

**CRITICAL: CLAUDE.md inclusion is mandatory, not optional.** If a CLAUDE.md
(or equivalent project instructions file) exists in the repo, read it and
include the coding guidelines and conventions sections in the prompt's
`## Project Context` section. Prioritize sections about: coding standards,
language guidelines, design patterns, review standards, and architectural
constraints. For review mode specifically, instruct Codex to check each
finding against these project-specific guidelines and flag violations as
review findings.

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

### 3. Run Codex

All modes except rescue use read-only sandbox:

```bash
codex exec \
  -s read-only \
  --ephemeral \
  -o "$OUTPUT_FILE" \
  - < "$PROMPT_FILE"
```

For **rescue** mode (Codex may need to edit files):

```bash
codex exec \
  -s workspace-write \
  --ephemeral \
  -o "$OUTPUT_FILE" \
  - < "$PROMPT_FILE"
```

Set `timeout: 600000` (10 minutes) on the Bash call.

If the command fails, check:

| Error | Action |
|-------|--------|
| `codex: command not found` | Tell user: `npm i -g @openai/codex` |
| Auth/API error | Tell user to check `codex login` or API key |
| Timeout | Report partial results if output file has content |

### 4. Read and synthesize results

Read the output file. Do NOT just pass through raw Codex output.
Instead:

**For review mode:**
- Parse Codex findings
- Compare each finding against your own analysis
- Label each as: CONFIRMED (both agree), CODEX-ONLY (Codex found, you missed),
  CLAUDE-ONLY (you found, Codex missed), or DISAGREEMENT (conflicting conclusions)
- Present a unified table with verdicts

**For debug mode:**
- Compare Codex's diagnosis with yours
- If they agree, confidence is high — present the shared conclusion
- If they disagree, present both theories with evidence and let the user decide

**For plan mode:**
- Highlight where Codex agrees with your approach (validation)
- Surface any risks or alternatives Codex identified that you missed
- If Codex suggests a fundamentally different approach, present both with trade-offs

**For verify mode:**
- If Codex confirms the fix: report as verified with Codex's reasoning
- If Codex finds issues: present them and ask whether to iterate

**For rca mode:**
- Map Codex's conclusion to your theories
- If Codex converges on the same root cause: high confidence
- If Codex identifies a different cause: present both with evidence

**For rescue mode:**
- If Codex produced working code: review it for correctness and style
  before applying. Check it follows project conventions.
- If Codex also failed: report what it tried and combine learnings

**For ask mode:**
- Synthesize Codex's answer with your own knowledge
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
"I'm going to get a second opinion from Codex on this because [reason]."

## Review Mode Details

### code-reviewer skill integration

When running review mode, the prompt MUST instruct Codex to check for the
`code-reviewer` skill and use it if available:

1. **Before assembling the review prompt**, add this preamble to the prompt file:

```markdown
## Skill Preference

Before starting the review, check if the `code-reviewer` skill is available
(look for SKILL.md at `~/.agents/skills/code-reviewer/SKILL.md` or
`~/.codex/skills/code-reviewer/SKILL.md` or `.claude/skills/code-reviewer/SKILL.md`).

- If `code-reviewer` is found: use its workflow to conduct the review instead
  of the generic instructions below. Pass it the diff and any focus area.
- If `code-reviewer` is NOT found: use the generic review instructions below,
  and at the END of your review output, append this note:

  ---
  **Tip:** Install the `code-reviewer` skill for richer, structured reviews:
  ```
  codex install https://skills.sh/google-gemini/gemini-cli/code-reviewer
  ```
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
used with `--output-schema` to get machine-parseable review findings.

## Tips

- Codex uses GPT models (currently gpt-5.4 by default). The cross-model
  benefit comes from architectural differences — the same bug can be
  invisible to one model and obvious to another.
- For large diffs, consider splitting into focused chunks rather than
  sending everything at once.
- Session IDs from `codex exec` are ephemeral by default. If you need
  multi-round interaction, drop `--ephemeral` and use
  `codex exec resume --last "follow-up instructions"`.
- Keep prompt files under 100KB — very large contexts degrade quality.
