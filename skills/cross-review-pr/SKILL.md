---
name: cross-review-pr
description: "Cross-model comparative review of a Pull Request. Runs Claude's code-reviewer on the PR, then sends findings to Codex for independent validation and its own review. Produces a unified report with confirmed issues, false-positive filtering, and a confidence score. Trigger this skill when the user asks for a 'comparative review', 'cross-review', 'dual review', 'cross-model review', 'validated review', 'second-opinion review', or wants both Claude and Codex to review a PR together. Also trigger when the user says 'review PR with Codex', 'get a second opinion on this PR', or 'compare reviews'."
---

# Cross-Review PR

Run a comparative code review on a Pull Request using two independent models
(Claude + OpenAI Codex). The value is in cross-validation: different model
architectures have different blind spots, so issues found by both models are
high-confidence, while single-model findings get scrutinized for false positives.

## Prerequisites

- `gh` CLI installed and authenticated (for PR checkout and metadata)
- `codex` CLI installed and authenticated (`npm i -g @openai/codex`)
- Project has a CLAUDE.md with coding standards (strongly recommended)

## Arguments

- **PR identifier** (required): PR number (e.g., `47`) or URL (e.g., `https://github.com/org/repo/pull/47`)
- **--focus AREA** (optional): Narrow both reviews to a specific area
  (security, performance, concurrency, error-handling). Default: general review.
- **--post** (optional): Post the unified report as a PR comment after presenting it.

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
```

If the diff exceeds 3000 lines, warn the user and suggest using `--focus`
to narrow scope. Proceed anyway unless they stop you.

### Step 2: Claude review (code-reviewer skill)

Checkout the PR branch so the code-reviewer has full file access:

```bash
gh pr checkout "$PR_NUM"
```

Invoke the code-reviewer skill to review the PR changes. The code-reviewer
skill handles the full review workflow: it reads the diff, analyzes against
the review pillars (correctness, maintainability, security, edge cases, etc.),
and produces structured findings.

```
Skill(skill="code-reviewer", args="PR #$PR_NUM")
```

After the skill completes, capture Claude's findings. Structure them as a list,
where each finding has: severity (Critical/Improvement/Nitpick), file, location,
title, and description.

Save the findings in a structured format for the next step. If the code-reviewer
produces its output as conversation text, parse it into a list of discrete findings.

### Step 3: Codex review + validation (codex-assist skill)

Use the codex-assist skill in review mode. The codex-assist skill already handles:
- CLAUDE.md inclusion in prompts (project coding standards)
- The "Skill Preference" preamble that tells Codex to use the code-reviewer
  skill if it's installed — so Codex gets the same structured review methodology
- `codex exec` invocation, error handling, and temp file cleanup
- Structured review output schema (P0-P3 severity, file, title, description)

The key addition for cross-review is the validation task. After assembling
the standard review context per codex-assist's workflow, append Claude's
findings as a second task in the prompt.

#### 3a. Build the prompt file following codex-assist's review template

Follow the codex-assist skill's review mode instructions (see its
`references/prompt-templates.md`). The prompt structure is:

1. **Common Header** — CLAUDE.md project conventions (mandatory per codex-assist)
2. **Skill Preference** — code-reviewer detection preamble (from codex-assist)
3. **Review Target** — the PR diff
4. **Focus** — user-specified or general

Then append the cross-validation task:

```markdown
## Additional Task: Validate Another Reviewer's Findings

After completing your independent review above, evaluate these findings
from a separate reviewer (Claude). For EACH finding, give your verdict:

- **CONFIRMED**: You agree this is a real issue. Briefly explain why.
- **FALSE_POSITIVE**: You believe this is not actually an issue. Explain why.
- **UNCERTAIN**: You can see arguments both ways. Explain the ambiguity.

Important: Complete your independent review FIRST. Do not let these findings
influence your own review — they are a separate task.

<claude-findings>
[Structured list of Claude's findings from Step 2, each with:
severity, file, title, description]
</claude-findings>

### Validation Output Format

For each Claude finding:
- finding: <Claude's finding title>
- verdict: CONFIRMED / FALSE_POSITIVE / UNCERTAIN
- reasoning: <your explanation>
```

#### 3b. Run Codex via codex-assist's invocation pattern

```bash
PROMPT_FILE=$(mktemp /tmp/cross-review-codex-XXXXXX)
OUTPUT_FILE=$(mktemp /tmp/cross-review-codex-result-XXXXXX)

# Write assembled prompt to PROMPT_FILE (structure from 3a above)

codex exec \
  -s read-only \
  --ephemeral \
  -o "$OUTPUT_FILE" \
  - < "$PROMPT_FILE"
```

Set `timeout: 600000` (10 minutes) on the Bash call.

#### 3c. Handle failure gracefully

If Codex fails (not installed, auth error, timeout), present Claude's review
alone with a note that cross-validation was unavailable. The Claude review
from code-reviewer is still valuable on its own — the comparative layer
is additive, not required.

### Step 4: Synthesize unified report

Read Codex's output. Parse it into: independent findings list and validation
verdicts. Then build the unified report by cross-referencing both reviews.

Categorize every finding into one of four buckets:

| Category | Meaning | Confidence |
|----------|---------|------------|
| **Confirmed** | Both models found the same issue | High |
| **Claude-only (validated)** | Claude found it, Codex confirmed it | High |
| **Claude-only (challenged)** | Claude found it, Codex says false positive | Low - needs human judgment |
| **Claude-only (uncertain)** | Claude found it, Codex is unsure | Medium |
| **Codex-only** | Codex found it, Claude missed it | Medium |

**Matching logic**: Two findings match if they reference the same file AND
the same logical issue (even if described differently). Use semantic matching,
not string equality — "missing null check on line 42" and "potential NPE in
validateInput" are the same finding if they point to the same code.

**Confidence score**: Calculate an overall score:

```
confirmed_weight = 1.0
validated_weight = 0.8
uncertain_weight = 0.5
codex_only_weight = 0.6
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

**Claude verdict**: [Approved / Request Changes]
**Codex verdict**: [Approved / Request Changes]
**Cross-model confidence**: [score as percentage]

## Confirmed Issues (both models agree)

[For each confirmed finding:]
### [severity] [title]
**File**: [path]:[line]
**Claude**: [Claude's description]
**Codex**: [Codex's description]
**Suggested fix**: [best suggestion from either model]

## Claude Findings — Codex Validation

[For each Claude-only finding:]
### [severity] [title]
**File**: [path]:[line]
**Description**: [Claude's finding]
**Codex verdict**: [CONFIRMED / FALSE_POSITIVE / UNCERTAIN]
**Codex reasoning**: [Codex's explanation]

## Codex-Only Findings (Claude missed)

[For each Codex-only finding:]
### [severity] [title]
**File**: [path]:[line]
**Description**: [Codex's finding]

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

Examples of good regression tests:
- NaN validation fix → test that NaN input produces the safe default
- Spurious decrease from 0.0 default → test that the default value is NaN
- Dead code removal → existing tests updated to use the new API surface
- New signal/flag → test that setting and clearing the flag works

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
| `codex` not installed | Present Claude review only, suggest `npm i -g @openai/codex` |
| Codex timeout | Present Claude review only with note |
| Codex auth error | Tell user to run `codex login` or set API key |
| Diff too large (>5000 lines) | Split by file groups and run sequentially |

## Tips

- The comparative approach is most valuable for critical PRs (security changes,
  core infrastructure, public API changes) where false positives are costly
  and missed bugs are dangerous.
- For routine PRs, a single-model review (just `code-reviewer`) is faster
  and usually sufficient.
- Different models excel at different things: Claude tends to catch architectural
  issues and convention violations; Codex (GPT) tends to catch edge cases and
  off-by-one errors. Together they cover more ground.
