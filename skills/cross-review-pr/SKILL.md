---
name: cross-review-pr
description: "Cross-model comparative review of a Pull Request. Runs one LLM as primary reviewer and another as validator. Default: Claude<->Codex. Supports any combination of Claude, Codex, and OpenCode via --from/--to flags. Trigger this skill when the user asks for a 'comparative review', 'cross-review', 'dual review', 'cross-model review', 'validated review', 'second-opinion review', or wants two LLMs to review a PR together. Also trigger when the user says 'review PR with Codex', 'review PR with OpenCode', 'get a second opinion on this PR', or 'compare reviews'."
---

# Cross-Review PR

Run a comparative code review on a Pull Request using two independent models.
The value is in cross-validation: different model architectures have different
blind spots, so issues found by both models are high-confidence, while
single-model findings get scrutinized for false positives.

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
- If you are **Codex**: `codex` roles run inline (do the review/validation
  yourself). `claude` and `opencode` roles shell out via their CLIs.
- If you are **OpenCode**: `opencode` roles run inline. `claude` and `codex`
  roles shell out via their CLIs.

**CRITICAL: Never shell out to yourself.** If `--from` or `--to` matches
your own identity, you perform that step directly — no CLI subprocess.
If the role is a *different* LLM, you invoke it via its CLI.

## Prerequisites

- `gh` CLI installed and authenticated (for PR checkout and metadata)
- External LLM CLIs installed for whichever models are selected
  (not needed when both `--from` and `--to` are `claude`)
- Project has a CLAUDE.md with coding standards (strongly recommended)

## Arguments

- **PR identifier** (required): PR number (e.g., `47`) or URL
  (e.g., `https://github.com/org/repo/pull/47`)
- **--from PRIMARY** (optional): LLM that performs the initial review.
  Values: `claude` (default), `codex`, `opencode`.
- **--to VALIDATOR** (optional): LLM that validates and adds its own findings.
  Values: `codex` (default), `claude`, `opencode`.
- **--focus AREA** (optional): Narrow both reviews to a specific area
  (security, performance, concurrency, error-handling). Default: general review.
- **--post** (optional): Post the unified report as a PR comment after
  presenting it.

`--from` and `--to` must not be the same LLM.

**Common combinations:**

| Shorthand | Equivalent |
|-----------|------------|
| (default) | `--from claude --to codex` |
| `--from codex` | `--from codex --to claude` |
| `--from opencode --to codex` | OpenCode reviews, Codex validates |

When only one of `--from`/`--to` is provided, the other defaults to the
opposite side of the Claude<->Codex pair. If the provided value is `claude`,
the other defaults to `codex` and vice versa. If the provided value is
`opencode`, the other defaults to `codex`.

## Workflow

### Step 1: Gather PR context

Extract the PR number from the argument. If it's a URL, parse the number from it.

```bash
PR_NUM="<extracted number>"
```

Fetch PR metadata and diff:

```bash
gh pr view "$PR_NUM" --json title,body,baseRefName,headRefName,files > /tmp/cross-review-pr-meta.json
gh pr diff "$PR_NUM" > /tmp/cross-review-pr-diff.patch
```

Read the PR title, description, base branch, and changed file list from the
metadata. Show the user a brief summary before proceeding:

```
Comparative review: PR #47 — "feat(27): backpressure pipeline"
Base: develop <- gsd/phase-27-backpressure-frame-dropping
Files changed: 12
Primary reviewer: claude | Validator: codex
```

If the diff exceeds 3000 lines, warn the user and suggest using `--focus`
to narrow scope. Proceed anyway unless they stop you.

### Step 2: Primary review

Run the primary review using whichever LLM is selected via `--from`.
Apply the **self-awareness rule**: if `--from` matches your own identity,
you are the primary reviewer — do it inline. Otherwise, shell out.

#### If primary is YOU (inline)

Checkout the PR branch so you have full file access:

```bash
gh pr checkout "$PR_NUM"
```

If you have a `code-reviewer` skill installed, invoke it:

```
Skill(skill="code-reviewer", args="PR #$PR_NUM")
```

Otherwise, review the diff directly. Produce a structured list where each
finding has: severity (Critical/Improvement/Nitpick), file, location,
title, and description. End with an overall verdict: Approved or Request Changes.

#### If primary is a DIFFERENT LLM (external CLI)

Build a review prompt file following the `llm-assist` skill's review mode
template. The prompt structure is:

1. **Common Header** — CLAUDE.md project conventions (mandatory per llm-assist)
2. **Skill Preference** — code-reviewer detection preamble (from llm-assist)
3. **Review Target** — the PR diff
4. **Focus** — user-specified or general

```markdown
## Task: Code Review

Review the following Pull Request diff. For each issue found, output:
- severity: Critical / Improvement / Nitpick
- file: <path>
- location: <line or range>
- title: <short title>
- description: <explanation>

At the end, give an overall verdict: Approved or Request Changes.

<pr-diff>
[contents of /tmp/cross-review-pr-diff.patch]
</pr-diff>
```

Run via the appropriate CLI for the `--from` LLM:

```bash
PROMPT_FILE=$(mktemp /tmp/cross-review-primary-XXXXXX)
OUTPUT_FILE=$(mktemp /tmp/cross-review-primary-result-XXXXXX)

# Write assembled prompt to PROMPT_FILE

# If --from is codex:
codex exec -s read-only --ephemeral -o "$OUTPUT_FILE" - < "$PROMPT_FILE"

# If --from is opencode:
opencode run "Follow the instructions in the attached file" \
  -f "$PROMPT_FILE" > "$OUTPUT_FILE" 2>&1

# If --from is claude (and you are NOT Claude):
claude -p "$PROMPT_FILE" --output-file "$OUTPUT_FILE"
```

Set `timeout: 600000` (10 minutes) on the Bash call.

Parse the output into the same structured findings format.

### Step 3: Validation review

Send the primary review's findings to the validator LLM, along with the
PR diff for its own independent review. Apply the **self-awareness rule**:
if `--to` matches your own identity, you are the validator — do it inline.
Otherwise, shell out.

#### If validator is YOU (inline)

You already have the PR diff and branch checked out. Perform the
validation directly:

1. Do an independent review of the diff, producing your own findings.
2. For each finding from the primary reviewer, give a verdict:
   - **CONFIRMED**: Agree this is a real issue. Briefly explain why.
   - **FALSE_POSITIVE**: Not actually an issue. Explain why.
   - **UNCERTAIN**: Arguments both ways. Explain the ambiguity.

Important: Complete the independent review FIRST before evaluating the
primary reviewer's findings, to avoid anchoring bias.

#### If validator is a DIFFERENT LLM (external CLI)

Build the validation prompt following `llm-assist`'s review template,
then append the cross-validation task:

1. **Common Header** — CLAUDE.md project conventions
2. **Skill Preference** — code-reviewer detection preamble
3. **Review Target** — the PR diff
4. **Focus** — user-specified or general
5. **Validation Task** (appended):

```markdown
## Additional Task: Validate Another Reviewer's Findings

After completing your independent review above, evaluate these findings
from a separate reviewer. For EACH finding, give your verdict:

- **CONFIRMED**: You agree this is a real issue. Briefly explain why.
- **FALSE_POSITIVE**: You believe this is not actually an issue. Explain why.
- **UNCERTAIN**: You can see arguments both ways. Explain the ambiguity.

Important: Complete your independent review FIRST. Do not let these findings
influence your own review — they are a separate task.

<primary-findings>
[Structured list of primary reviewer's findings, each with:
severity, file, title, description]
</primary-findings>

### Validation Output Format

For each finding from the primary reviewer:
- finding: <primary reviewer's finding title>
- verdict: CONFIRMED / FALSE_POSITIVE / UNCERTAIN
- reasoning: <your explanation>
```

Run via the appropriate CLI for the `--to` LLM:

```bash
PROMPT_FILE=$(mktemp /tmp/cross-review-validator-XXXXXX)
OUTPUT_FILE=$(mktemp /tmp/cross-review-validator-result-XXXXXX)

# Write assembled prompt to PROMPT_FILE

# If --to is codex:
codex exec -s read-only --ephemeral -o "$OUTPUT_FILE" - < "$PROMPT_FILE"

# If --to is opencode:
opencode run "Follow the instructions in the attached file" \
  -f "$PROMPT_FILE" > "$OUTPUT_FILE" 2>&1

# If --to is claude (and you are NOT Claude):
claude -p "$PROMPT_FILE" --output-file "$OUTPUT_FILE"
```

Set `timeout: 600000` (10 minutes) on the Bash call.

#### Handle failure gracefully

If the validator LLM fails (not installed, auth error, timeout), present
the primary review alone with a note that cross-validation was unavailable.
The primary review is still valuable on its own — the comparative layer is
additive, not required.

### Step 4: Synthesize unified report

Read both reviews. Parse the validator's output into: independent findings
list and validation verdicts. Then build the unified report by
cross-referencing both reviews.

Categorize every finding into one of five buckets:

| Category | Meaning | Confidence |
|----------|---------|------------|
| **Confirmed** | Both models found the same issue independently | High |
| **Primary-only (validated)** | Primary found it, validator confirmed it | High |
| **Primary-only (challenged)** | Primary found it, validator says false positive | Low - needs human judgment |
| **Primary-only (uncertain)** | Primary found it, validator is unsure | Medium |
| **Validator-only** | Validator found it, primary missed it | Medium |

**Matching logic**: Two findings match if they reference the same file AND
the same logical issue (even if described differently). Use semantic matching,
not string equality — "missing null check on line 42" and "potential NPE in
validateInput" are the same finding if they point to the same code.

**Confidence score**: Calculate an overall score:

```
confirmed_weight = 1.0
validated_weight = 0.8
uncertain_weight = 0.5
validator_only_weight = 0.6
challenged_weight = 0.2

score = weighted_sum / total_findings
```

This isn't a pass/fail metric — it indicates how much agreement exists between
the two models. A score above 0.7 means strong consensus; below 0.5 means
significant disagreement worth investigating.

### Step 5: Present the report

Display the unified report to the user:

```markdown
# Comparative Review: PR #47

**Primary reviewer**: [claude/codex/opencode] | **Validator**: [claude/codex/opencode]
**Primary verdict**: [Approved / Request Changes]
**Validator verdict**: [Approved / Request Changes]
**Cross-model confidence**: [score as percentage]

## Confirmed Issues (both models agree)

[For each confirmed finding:]
### [severity] [title]
**File**: [path]:[line]
**Primary reviewer**: [description]
**Validator**: [description]
**Suggested fix**: [best suggestion from either model]

## Primary Findings — Validator Verdicts

[For each primary-only finding:]
### [severity] [title]
**File**: [path]:[line]
**Description**: [primary reviewer's finding]
**Validator verdict**: [CONFIRMED / FALSE_POSITIVE / UNCERTAIN]
**Validator reasoning**: [explanation]

## Validator-Only Findings (primary missed)

[For each validator-only finding:]
### [severity] [title]
**File**: [path]:[line]
**Description**: [validator's finding]

## Summary

[Narrative combining both perspectives. Highlight areas of agreement
and disagreement. Call out any findings where the models disagree
that the user should pay special attention to.]
```

### Step 5b: Regression tests for fixes

When the user asks to fix confirmed issues after the review, every bug fix
must include a regression test that would have caught the original bug.
This is non-negotiable — a fix without a test is incomplete.

For each fix, add a test that:
1. Reproduces the invalid input or bad state that triggered the bug
2. Asserts the corrected behavior
3. Lives alongside the existing tests for that module

Present a summary of added tests in the report so the user can verify coverage.

### Step 6: Optional — Post as PR comment

If `--post` flag was provided, ask the user to confirm before posting:

```
Post this report as a comment on PR #47? (yes/no)
```

If confirmed:

```bash
gh pr comment "$PR_NUM" --body-file /tmp/cross-review-report.md
```

### Step 7: Clean up

```bash
rm -f /tmp/cross-review-pr-meta.json /tmp/cross-review-pr-diff.patch
rm -f "$PROMPT_FILE" "$OUTPUT_FILE"
```

Switch back to the previous branch:

```bash
git checkout - 2>/dev/null || true
```

## Error Handling

| Failure | Recovery |
|---------|----------|
| `gh` not installed | Tell user to install GitHub CLI |
| PR not found | Check PR number/URL and repo |
| `--from` and `--to` are the same | Error: primary and validator must be different LLMs |
| External LLM not installed | If it's the primary, abort with install instructions. If it's the validator, present primary review alone. |
| External LLM timeout | Same as not installed — degrade gracefully if validator, abort if primary |
| External LLM auth error | Tell user to check auth config for the selected provider |
| Diff too large (>5000 lines) | Split by file groups and run sequentially |

## Tips

- The comparative approach is most valuable for critical PRs (security changes,
  core infrastructure, public API changes) where false positives are costly
  and missed bugs are dangerous.
- For routine PRs, a single-model review (just `code-reviewer`) is faster
  and usually sufficient.
- Different models excel at different things: Claude tends to catch architectural
  issues and convention violations; Codex often catches edge cases and
  off-by-one errors. Together they cover more ground.
- Running `--from codex --to claude` is useful when you want Codex's review
  validated by Claude's deeper architectural understanding.
- Running `--from claude --to opencode` gives you a different second opinion
  if Codex is unavailable.
