---
name: cross-review-pr
description: "Cross-model comparative PR / Pull Request review of a PR, branch, commit, or codebase scope. Runs two LLMs through independent reviews and reciprocal validation, then synthesizes cross-checked findings. Default: Claude <-> Codex. Supports Claude, Codex, and OpenCode via --from/--to. Trigger for comparative PR review, comparative review, PR cross-review, cross-review, dual review, cross-model review, validated review, second-opinion review, deep PR review, deep review, multi-area review, parallel-agent PR review, or when the user wants two LLMs to review a PR together. Deep review requires explicit permission to spawn sub-agents/parallel agents where the host policy requires it; see references/deep-mode.md."
---

# Cross-Review PR

Run a comparative code review on a Pull Request using two independent models.
The value is in reciprocal cross-validation: each model reviews independently,
then checks the other model's findings. Issues found or confirmed by both
models are high-confidence, while single-model findings get scrutinized for
false positives before the final report.

## Supported LLMs

| ID | Description |
|----|-------------|
| `claude` | Claude Code |
| `codex` | OpenAI Codex CLI |
| `opencode` | OpenCode CLI |

## Self-awareness rule

**You (the agent executing this skill) must identify which LLM you are.**
This determines which roles you handle inline vs. which require shelling
out to an external CLI.

- If you are **Claude**: `claude` roles run inline (use `code-reviewer` skill
  or direct analysis). `codex` and `opencode` roles shell out via their CLIs.
- If you are **Codex**: `codex` roles run inline (do the review and validation
  yourself). `claude` and `opencode` roles shell out via their CLIs.
- If you are **OpenCode**: `opencode` roles run inline. `claude` and `codex`
  roles shell out via their CLIs.

**CRITICAL: Never shell out to yourself.** If `--from` or `--to` matches
your own identity, you perform that step directly — no CLI subprocess.
If the role is a *different* LLM, you invoke it via its CLI.

## Prerequisites

- `gh` CLI installed and authenticated (for PR checkout and metadata)
- External LLM CLIs installed for whichever selected reviewer is not the
  invoking LLM
- Project has a CLAUDE.md with coding standards (strongly recommended)

## Arguments

- **Scope** (required for default mode): PR number (e.g., `47`) or URL
  (e.g., `https://github.com/org/repo/pull/47`). In `--deep` mode the
  scope can also be a branch range (`main..HEAD`), a directory path
  (`src/`), or omitted (review the entire `src/` tree).
- **--from REVIEWER_A** (optional): LLM that performs the first independent review.
  Values: `claude` (default), `codex`, `opencode`.
- **--to REVIEWER_B** (optional): LLM that performs the second independent review
  and participates in reciprocal validation.
  Values: `codex` (default), `claude`, `opencode`.
- **--focus AREA** (optional): Narrow both reviews to a specific area
  (security, performance, concurrency, error-handling). Default: general review.
- **--deep** (optional): Run the multi-agent multi-area review described
  in the **Deep Mode** section below. The word "deep" requests deep mode, but
  it is not automatically explicit permission to spawn sub-agents in hosts
  with restrictive delegation policy. If the host requires explicit permission
  for sub-agents or parallel agent work, ask for that permission before
  launching strict deep mode. Default areas: 5. Override with `--areas N`.
- **--areas N** (optional, deep mode only): Number of focus areas to
  decompose the scope into. Default: `5`. Range: 2–8. Smaller scopes
  may use fewer; the skill will downscale automatically.
- **--post** (optional): Post the unified report as a PR comment after
  presenting it (only meaningful when scope is a PR).

`--from` and `--to` must not be the same LLM.

**Common combinations:**

| Shorthand | Equivalent |
|-----------|------------|
| (default) | `--from claude --to codex` |
| `--from codex` | `--from codex --to claude` |
| `--from opencode --to codex` | OpenCode and Codex cross-review |

When only one of `--from`/`--to` is provided, the other defaults to the
opposite side of the Claude<->Codex pair. If the provided value is `claude`,
the other defaults to `codex` and vice versa. If the provided value is
`opencode`, the other defaults to `codex`.

## Workflow

### Step 1: Terminal Awareness

Before gathering diffs or generating prompt files, inspect the active terminal
environment and choose commands that are safe for that shell:

```bash
printf 'SHELL=%s\n' "${SHELL:-unknown}"
ps -p $$ -o comm=
command -v bash || true
command -v zsh || true
```

If the current shell is `zsh`, do not rely on zsh word splitting. Prefer one
of these patterns:

- Run prompt-generation scripts under `bash` with `set -euo pipefail`.
- Use newline-safe loops: `while IFS= read -r file; do ...; done < "$file_list"`.
- Use shell arrays for command argv.
- Append dynamic content with `printf '%s\n' "$value"` or `cat file`, not
  `echo`.

Avoid unquoted list expansion such as `for file in $FILES; do ...; done`.
That can produce empty or malformed prompt sections under zsh.

After generating any prompt file, validate it before invoking reviewers:

```bash
wc -l "$PROMPT_FILE"
rg -n '^(diff --git|## Source:|<pr-diff>|<diff>)' "$PROMPT_FILE" | head
test "$(wc -l < "$PROMPT_FILE")" -gt 50
```

If the prompt is unexpectedly short or lacks source/diff markers, stop and
regenerate it under a known-safe shell before running Claude, Codex, or
OpenCode. Do not launch reviewers against empty or placeholder prompts.

Prompt files may contain proprietary source code. Before writing any prompt,
metadata, or diff files, create a dedicated temporary directory per review and
clean it up even when the workflow fails:

```bash
CROSS_REVIEW_TMPDIR=$(mktemp -d /tmp/cross-review-pr-XXXXXX)
cleanup_cross_review_tmpdir() {
  rm -rf "$CROSS_REVIEW_TMPDIR"
}
trap cleanup_cross_review_tmpdir EXIT
```

### Step 2: Deep-Mode Delegation Authorization

If `--deep` is active, read `references/deep-mode.md` before proceeding.
Strict deep mode requires symmetric independent parallel reviewer agents:
Reviewer A and Reviewer B must each review every decomposed area independently,
then validate each other's findings for that same area before synthesis. Do
not satisfy deep mode by fanning out only one reviewer while the other performs
one whole-PR pass or only validates. In environments that only allow
sub-agents, delegation, or parallel agent work after explicit user
authorization, a request like "deep review" or "deep comparative PR review" is
not enough by itself.

If explicit authorization is missing, stop and ask:

```text
Strict deep mode requires spawning parallel reviewer sub-agents. Do you want me
to spawn parallel reviewer sub-agents for this review?
```

To avoid this checkpoint, the user can explicitly request strict deep mode with
phrases like "deep parallel-agent review", "spawn parallel reviewer
sub-agents", or "`--deep` with sub-agents".

Do not silently fall back to one inline pass, local-only validation, or only
external CLI processes. If the user declines sub-agents, ask whether they want
a non-deep comparative fallback and label that fallback clearly.

### Step 3: Gather PR Context

Extract the PR number from the argument. If it's a URL, parse the number from it.

```bash
PR_NUM="<extracted number>"
```

Fetch PR metadata and diff:

```bash
gh pr view "$PR_NUM" --json title,body,baseRefName,headRefName,files > "$CROSS_REVIEW_TMPDIR/pr-meta.json"
gh pr diff "$PR_NUM" > "$CROSS_REVIEW_TMPDIR/pr-diff.patch"
```

Read the PR title, description, base branch, and changed file list from the
metadata. Show the user a brief summary before proceeding:

```text
Comparative review: PR #47 — "feat(27): backpressure pipeline"
Base: develop <- gsd/phase-27-backpressure-frame-dropping
Files changed: 12
Reviewer A: claude | Reviewer B: codex
Mode: independent reviews + reciprocal validation
```

If the diff exceeds 3000 lines, warn the user and suggest using `--focus`
to narrow scope. If it exceeds 5000 lines, split by file groups and run the
workflow sequentially per group.

### Step 4: Reviewer A Independent Review

Run the first independent review using whichever LLM is selected via `--from`.
Apply the **self-awareness rule**: if `--from` matches your own identity, do it
inline. Otherwise, shell out.

If the reviewer is YOU, check out the PR branch so you have full file access:

```bash
git status --short
test -z "$(git status --short)" || {
  printf '%s\n' "Working tree is dirty; inspect before checking out the PR."
  exit 1
}
gh pr checkout "$PR_NUM"
```

If the working tree is dirty, do not check out the PR branch until you have
confirmed the changes are unrelated and checkout will not overwrite them. If
checkout would disturb user changes, stop and ask the user how to proceed.

If you have a `code-reviewer` skill installed, use that skill's workflow.
Otherwise, review the diff directly. Produce a structured list where each
finding has: severity (Critical/Improvement/Nitpick), file, location, title,
description, and suggested fix. End with an overall verdict: Approved or
Request Changes.

If the reviewer is a DIFFERENT LLM, build a review prompt file following the
`llm-assist` skill's review mode template:

1. **Common Header** — CLAUDE.md project conventions (mandatory per llm-assist)
2. **Skill Preference** — code-reviewer detection preamble (from llm-assist)
3. **Review Target** — the PR diff
4. **Focus** — user-specified or general

```markdown
## Task: Independent Code Review

Review the following Pull Request diff independently. Do not assume another
reviewer will catch issues. For each issue found, output:
- severity: Critical / Improvement / Nitpick
- file: <path>
- location: <line or range>
- title: <short title>
- description: <explanation>
- suggested_fix: <concrete remediation>

At the end, give an overall verdict: Approved or Request Changes.

<pr-diff>
[contents of "$CROSS_REVIEW_TMPDIR/pr-diff.patch"]
</pr-diff>
```

Run via the appropriate CLI for the reviewer LLM:

```bash
PROMPT_FILE=$(mktemp "$CROSS_REVIEW_TMPDIR/reviewer-a-prompt-XXXXXX")
OUTPUT_FILE=$(mktemp "$CROSS_REVIEW_TMPDIR/reviewer-a-result-XXXXXX")

# Write assembled prompt to PROMPT_FILE

# If reviewer is codex:
codex exec -s read-only --ephemeral -o "$OUTPUT_FILE" - < "$PROMPT_FILE"

# If reviewer is opencode:
opencode run "Follow the instructions in the attached file" \
  -f "$PROMPT_FILE" > "$OUTPUT_FILE" 2>&1

# If reviewer is claude and you are NOT Claude:
claude -p "Follow the instructions provided on stdin." \
  --verbose \
  --output-format stream-json \
  --include-partial-messages \
  < "$PROMPT_FILE" > "$OUTPUT_FILE" 2>&1
```

Use a generous wait budget for external reviewer CLIs, but do not treat a
long-running process as hung merely because it is slow or quiet. Monitor
progress before deciding what to do:

```bash
ps -o pid=,etime=,pcpu=,state=,command= -p "$REVIEWER_PID"
wc -c "$OUTPUT_FILE"
tail -n 40 "$OUTPUT_FILE"
```

If output size is increasing, tool-use events are appearing, CPU is non-zero, or
the process is otherwise doing work, keep waiting and tell the user what
progress you see. If progress is ambiguous, ask the user whether to keep waiting
or stop and include elapsed time, output-file size, recent output summary,
process state, and what result would be lost by stopping. Only kill an external
reviewer without asking when it has clearly exited badly, is an obvious orphan,
or the user explicitly instructs you to stop it.

Parse the output into the same structured findings format.
If Reviewer A fails because the CLI is unavailable, auth is broken, or the
process exits badly, stop the comparative workflow and report the failure. Do
not continue to reciprocal validation with only one completed review.

### Step 5: Reviewer B Independent Review

Run the second independent review using whichever LLM is selected via `--to`.
Apply the **self-awareness rule** exactly as in Step 4.

Reviewer B must not receive Reviewer A's findings yet. Give it the same PR
metadata, project conventions, focus, and diff, but no cross-validation task.
This preserves independence and avoids anchoring.

Use the same output schema as Step 4:

- severity: Critical / Improvement / Nitpick
- file: <path>
- location: <line or range>
- title: <short title>
- description: <explanation>
- suggested_fix: <concrete remediation>
- overall verdict: Approved or Request Changes

Use a distinct prompt/output path so the two reviews do not overwrite each
other:

```bash
PROMPT_FILE=$(mktemp "$CROSS_REVIEW_TMPDIR/reviewer-b-prompt-XXXXXX")
OUTPUT_FILE=$(mktemp "$CROSS_REVIEW_TMPDIR/reviewer-b-result-XXXXXX")
```

Monitor external CLIs with the same slow-is-not-hung rule from Step 4. If
Reviewer B fails because the CLI is unavailable, auth is broken, or the process
exits badly, present Reviewer A's review with a clear note that the comparative
layer could not be completed. Do not fabricate a second review.

### Step 6: Reciprocal Validation

After both independent reviews are complete, cross-check findings in both
directions.

Reviewer A validates Reviewer B's findings:

- If Reviewer A is YOU, evaluate each Reviewer B finding inline.
- If Reviewer A is external, send Reviewer B's findings plus the PR diff back
  to that LLM, along with Reviewer A's own independent review for context. Do
  not ask it to redo the full independent review.

Reviewer B validates Reviewer A's findings:

- If Reviewer B is YOU, evaluate each Reviewer A finding inline.
- If Reviewer B is external, send Reviewer A's findings plus the PR diff back
  to that LLM, along with Reviewer B's own independent review for context. Do
  not ask it to redo the full independent review.

Validation prompt:

```markdown
## Task: Validate Another Reviewer's Findings

You previously produced the independent review included below. Use it as
context, but do not redo the full review. Now evaluate the findings from the
other reviewer against the PR diff. For EACH finding, give your verdict:

- **CONFIRMED**: You agree this is a real issue. Briefly explain why.
- **FALSE_POSITIVE**: You believe this is not actually an issue. Explain why.
- **UNCERTAIN**: You can see arguments both ways. Explain the ambiguity.

<your-independent-review>
[Structured review previously produced by this same reviewer]
</your-independent-review>

<other-reviewer-findings>
[Structured list of findings, each with severity, file, location, title,
description, and suggested_fix]
</other-reviewer-findings>

<pr-diff>
[contents of "$CROSS_REVIEW_TMPDIR/pr-diff.patch"]
</pr-diff>

### Validation Output Format

For each finding from the other reviewer:
- finding: <finding title>
- verdict: CONFIRMED / FALSE_POSITIVE / UNCERTAIN
- reasoning: <your explanation>
```

Use distinct prompt/output paths for each direction:

```bash
PROMPT_FILE=$(mktemp "$CROSS_REVIEW_TMPDIR/validation-a-prompt-XXXXXX")
OUTPUT_FILE=$(mktemp "$CROSS_REVIEW_TMPDIR/validation-a-result-XXXXXX")

PROMPT_FILE=$(mktemp "$CROSS_REVIEW_TMPDIR/validation-b-prompt-XXXXXX")
OUTPUT_FILE=$(mktemp "$CROSS_REVIEW_TMPDIR/validation-b-result-XXXXXX")
```

Run external validation through the same CLI command pattern and monitoring
rules used for independent reviews in Step 4.

If either reverse-validation step fails, keep the available independent review
and partial validation results, but label the missing validation explicitly in
the final report.

### Step 7: Synthesize Unified Report

Read both independent reviews and both validation passes. Build the unified
report by cross-referencing findings semantically.

Categorize every finding into one of six buckets:

| Category | Meaning | Confidence |
|----------|---------|------------|
| **Confirmed independently** | Both models found the same issue independently | High |
| **A-only, confirmed by B** | Reviewer A found it; Reviewer B did not find it independently but confirmed it during validation | High |
| **B-only, confirmed by A** | Reviewer B found it; Reviewer A did not find it independently but confirmed it during validation | High |
| **Challenged** | One reviewer found it; the other says false positive | Low - needs human judgment |
| **Uncertain** | One reviewer found it; the other is unsure or validation is missing | Medium |
| **Unvalidated** | Comparative validation failed for this finding | Medium-low |

**Matching logic**: Two findings match if they reference the same file AND the
same logical issue, even if described differently. Use semantic matching, not
string equality.

**Confidence score**: Calculate an overall score:

```text
confirmed_independently_weight = 1.0
confirmed_by_other_weight = 0.85
uncertain_weight = 0.5
unvalidated_weight = 0.4
challenged_weight = 0.2

weighted_sum =
  confirmed_independently_count * confirmed_independently_weight
  + confirmed_by_other_count * confirmed_by_other_weight
  + uncertain_count * uncertain_weight
  + unvalidated_count * unvalidated_weight
  + challenged_count * challenged_weight

score = weighted_sum / total_finding_count
```

This is not a pass/fail metric. It indicates how much agreement exists between
the two models. A score above 0.7 means strong consensus; below 0.5 means
significant disagreement worth investigating.

### Step 8: Present The Report

Display the unified report to the user:

```markdown
# Comparative Review: PR #47

**Reviewer A**: [claude/codex/opencode]
**Reviewer B**: [claude/codex/opencode]
**Reviewer A verdict**: [Approved / Request Changes]
**Reviewer B verdict**: [Approved / Request Changes]
**Cross-model confidence**: [score as percentage]

## Confirmed Issues

[For each independently confirmed or cross-confirmed finding:]
### [severity] [title]
**File**: [path]:[line]
**Reviewer A**: [description or validation reasoning]
**Reviewer B**: [description or validation reasoning]
**Suggested fix**: [best suggestion from either model]

## Challenged Or Uncertain Findings

[For each challenged, uncertain, or unvalidated finding:]
### [severity] [title]
**File**: [path]:[line]
**Found by**: [Reviewer A / Reviewer B]
**Description**: [finding]
**Other reviewer verdict**: [FALSE_POSITIVE / UNCERTAIN / missing]
**Reasoning**: [other reviewer reasoning when available]

## Summary

[Narrative combining both perspectives. Highlight agreement, disagreement,
and any findings that need human judgment before acting.]
```

### Step 9: Regression Tests For Fixes

When the user asks to fix confirmed issues after the review, every bug fix
must include a regression test that would have caught the original bug.
This is non-negotiable; a fix without a test is incomplete.

For each fix, add a test that:
1. Reproduces the invalid input or bad state that triggered the bug
2. Asserts the corrected behavior
3. Lives alongside the existing tests for that module

Present a summary of added tests in the report so the user can verify coverage.

### Step 10: Optional Post As PR Comment

If `--post` flag was provided, ask the user to confirm before posting:

```text
Post this report as a comment on PR #47? (yes/no)
```

If confirmed:

```bash
gh pr comment "$PR_NUM" --body-file /tmp/cross-review-report.md
```

### Step 11: Clean Up

```bash
rm -rf "$CROSS_REVIEW_TMPDIR"
```

Switch back to the previous branch:

```bash
git checkout - 2>/dev/null || true
```

## Deep Mode

Activated by `--deep` or by a request for a deep, multi-area, or
parallel-agent review. Deep mode means symmetric focused parallel reviewer
fanout: for each decomposed area, run Reviewer A and Reviewer B independently
on the same scope, then validate findings area-by-area before the final
synthesis. It is not one longer inline pass, one fanned-out model plus one
whole-PR pass, or optional validator calls from area agents.

Read `references/deep-mode.md` before running deep mode. If the current
harness cannot launch parallel agents, say that strict deep mode is unavailable
and clearly label any fallback as a normal comparative review.

## Error Handling

| Failure | Recovery |
|---------|----------|
| `gh` not installed | Tell user to install GitHub CLI |
| PR not found | Check PR number/URL and repo |
| `--from` and `--to` are the same | Error: the two reviewers must be different LLMs |
| External LLM not installed | If an independent review cannot run, stop and report that comparative review is unavailable. If only reciprocal validation fails after both reviews completed, present both reviews and label missing validation clearly. |
| External LLM appears slow | Monitor process/output. If progressing, keep waiting. If ambiguous, ask the user whether to wait or stop with elapsed time, output size, recent output summary, and process state. Applies to Claude Code, Codex, and OpenCode. |
| External LLM timeout with no progress | Ask the user before killing unless the process clearly failed or they already instructed you to stop. If stopped, present partial output only if clearly marked incomplete. |
| External LLM auth error | Tell user to check auth config for the selected provider |
| Diff too large (>3000 lines) | Warn the user and suggest `--focus`; above 5000 lines, split by file groups and run sequentially |
| Dirty working tree before PR checkout | Stop before checkout if user changes could be disturbed; ask how to proceed |

## Tips

- The comparative approach is most valuable for critical PRs (security changes,
  core infrastructure, public API changes) where false positives are costly
  and missed bugs are dangerous.
- For routine PRs, a single-model review (just `code-reviewer`) is faster
  and usually sufficient.
- Different models excel at different things: Claude tends to catch architectural
  issues and convention violations; Codex often catches edge cases and
  off-by-one errors. Together they cover more ground.
- Running `--from codex --to claude` starts with Codex's independent pass, then
  has Claude independently review and cross-check it.
- Running `--from claude --to opencode` pairs Claude with OpenCode when Codex is
  unavailable or when a third implementation perspective is useful.
