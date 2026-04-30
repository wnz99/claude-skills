---
name: cross-review-pr
description: "Cross-model comparative review of a Pull Request, branch, or codebase scope. Runs one LLM as primary reviewer and another as validator. Default: Claude<->Codex. Supports any combination of Claude, Codex, and OpenCode via --from/--to flags. Deep means parallel-agent fanout: if the user asks for a 'deep PR review', 'deep comparative review', 'deep cross-review', 'multi-area review', or uses --deep, treat that as a request to spawn 4–5 parallel reviewer agents, each focused on a distinct review area with strict spec-anchored prompts, finding validation, and missing-test gap analysis. Trigger this skill when the user asks for a 'comparative review', 'cross-review', 'dual review', 'cross-model review', 'validated review', 'second-opinion review', 'deep review', 'multi-area review', 'parallel-agent review', or wants two LLMs to review a PR together. Also trigger when the user says 'review PR with Codex', 'review PR with OpenCode', 'get a second opinion on this PR', 'compare reviews', or 'run a deep cross-review'."
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

- **Scope** (required for default mode): PR number (e.g., `47`) or URL
  (e.g., `https://github.com/org/repo/pull/47`). In `--deep` mode the
  scope can also be a branch range (`main..HEAD`), a directory path
  (`src/`), or omitted (review the entire `src/` tree).
- **--from PRIMARY** (optional): LLM that performs the initial review.
  Values: `claude` (default), `codex`, `opencode`.
- **--to VALIDATOR** (optional): LLM that validates and adds its own findings.
  Values: `codex` (default), `claude`, `opencode`.
- **--focus AREA** (optional): Narrow both reviews to a specific area
  (security, performance, concurrency, error-handling). Default: general review.
- **--deep** (optional): Run the multi-agent multi-area review described
  in the **Deep Mode** section below. The word "deep" in the user's
  request is equivalent to passing `--deep`; do not interpret it as a
  generic request for extra thoroughness. Each area is reviewed by an
  independent parallel agent that internally runs the same primary/validator
  cross-review on its slice of code. Default areas: 5. Override with
  `--areas N`.
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

## Deep Mode

Activated by `--deep` or by the user using the word "deep" for the PR
review. In this skill, **deep has one precise meaning**: spawn parallel
reviewer agents, each focused on a specific review area. The standard
mode runs one primary + one validator over the full scope. Deep mode
decomposes the scope into 4–5 distinct **focus areas** and runs an
independent reviewer agent per area in parallel. Each area-agent then
runs its own primary/validator cross-review on its slice, so the user
gets a multi-agent multi-area synthesis instead of a single global
review.

Do not satisfy a deep review by doing one longer inline pass. The point
of Deep Mode is to split attention across multiple focused reviewers so
bugs, correctness issues, integration gaps, and missing tests buried in
a large corpus are less likely to be missed by one agent carrying the
entire diff in a single context.

If the current harness cannot spawn parallel agents, say that strict
Deep Mode is unavailable in this environment before continuing. Then
either ask for explicit permission/alternative tooling if the harness
requires it, or clearly label the fallback as a non-deep comparative
review.

The empirical reason this exists: a single reviewer pass over a large
diff produces shallow generic findings; the reviewer's attention is
spread thin and it tends to flag stylistic concerns and miss
correctness bugs deep in specific subsystems. A focused reviewer with
verbatim spec excerpts and a tight scope produces concrete bugs
anchored to file:line. Run 4–5 of those in parallel and you cover the
codebase breadth without losing depth.

### When to use

- Critical PRs that touch multiple subsystems (parser + storage +
  classifier in one PR).
- Periodic codebase-health audits — point at `src/` with `--deep`,
  let the agents fan out.
- After major refactors that crossed module boundaries.
- When you've already done a default-mode review and want a second,
  deeper pass.

Skip deep mode for narrow PRs (one file, one function). The fan-out
overhead isn't worth it.

### Step D1: Determine scope

Resolve the scope argument into a concrete file list:

| Scope shape | How to resolve |
|---|---|
| PR number / URL | `gh pr diff "$PR_NUM" --name-only` then read each file |
| Branch range (`main..HEAD`) | `git diff --name-only main..HEAD` |
| Directory path (`src/foo/`) | `find <path> -name '*.py' -o -name '*.ts' …` |
| Omitted | Default to `src/` (or repo source root if different) |

If the resolved file list exceeds 50 files or 10,000 LOC, ask the user
to confirm before proceeding (deep mode will spawn many parallel
agents).

### Step D2: Decompose into focus areas

Split the file list into 4–5 areas (or `--areas N`). The split must
satisfy:

1. Each area is **self-contained**: a reviewer can analyze it without
   needing simultaneous context on the other areas.
2. Each area is **roughly balanced** in LOC (within 3x of the others).
3. Areas are grouped by **layer or directory**, not arbitrarily.

Heuristics to pick areas:

- **Directory tree**: top-level subdirectories under the source root
  often map cleanly onto areas (`parsers/`, `data/`, `metrics/`,
  `models/`, `cli/`). Use this when the project is well-organized.
- **Layer**: parsing → fetching → processing → classification →
  presentation. Use this for layered architectures.
- **Concern**: types/contracts, side-effecting code, pure math,
  presentation. Use this when the directory tree is flat.
- **Touched-by-the-diff**: in `--deep` over a PR, prefer the changed
  files clustered by directory; pull in adjacent unchanged files only
  if the change crosses an interface.

State the area split explicitly to the user before launching, so they
can correct it. This is also the point where you make clear that Deep
Mode is about to spawn parallel review agents:

```
Deep review: 5 areas decomposed from src/
  Area 1: parsers/ + models.py (~507 LOC) — input layer
  Area 2: data/ (4 files, ~858 LOC) — reference-data fetch
  Area 3: metrics/markout.py (~523 LOC) — decay math
  Area 4: metrics/{holding,uniformity,clustering,brackets}.py + _* (~1488 LOC)
  Area 5: profile.py (~1839 LOC) — classifier
Spawning 5 parallel reviewer agents.
```

### Step D3: Extract canonical spec excerpts (CRITICAL)

The most common deep-review failure mode is paraphrasing the spec in
the reviewer prompt. The reviewer flags "deviations" from the
paraphrase, not from the actual spec — every finding is then a
strawman caused by the prompt. **Always quote spec excerpts verbatim
from the source-of-truth document(s).**

Locate canonical specs in this order, stopping at the first that
exists:

1. `<repo>/SPEC*.md`, `<repo>/SPECS*.md`, or named like
   `TOXIC_TRADES_LAB_SPECS.md`
2. `<repo>/.planning/PROJECT.md` and `<repo>/.planning/REQUIREMENTS.md`
3. `<repo>/CLAUDE.md` and `<repo>/AGENTS.md`
4. `<repo>/docs/*.md` if present
5. `<repo>/README.md` as a last resort

For each area, extract the relevant verbatim sections (e.g., for a
markout area, copy SPEC §7.1 verbatim into the prompt; for a parser
area, copy the broker-log format spec). Do not paraphrase. If the
spec is silent on a point, say so explicitly in the prompt — that
gives the reviewer permission to flag the silence as INFO without
manufacturing bug findings.

### Step D4: Spawn parallel reviewer agents

For each area, launch an agent in parallel via the Agent tool with
`subagent_type: "general-purpose"` (or whatever your harness calls it),
`run_in_background: true`. Each agent's prompt should be self-contained
and follow this skeleton:

```markdown
You are running an independent cross-review of one area of the
<project name> codebase using <validator LLM> as the second reviewer.
You are running in parallel with N–1 other reviewers covering different
areas.

# What you're reviewing

Area: <descriptive name>
Files:
- <absolute path 1>
- <absolute path 2>
…

Scope: <one-paragraph description of what this area is responsible for
and why it matters to the project>.

# Project context

<Paste relevant CLAUDE.md / project rules: tone, no-sycophancy,
preferred tools, anything load-bearing for review judgment>.

# Verbatim spec excerpts (only source of truth)

<Paste the relevant SPEC section(s) verbatim. Do NOT paraphrase.>

# Your job

1. Build a focused <validator LLM> prompt at /tmp/llm-review-<area>-prompt.md:
   - Include the verbatim spec above.
   - Include the full source of all in-scope files (use cat to append).
   - Ask <validator LLM> for correctness/edge-case bugs anchored to the spec.
   - Specify deep-dive categories tailored to this area.
   - Output format: severity + spec-line + file:line + bug + concrete
     trigger input + suggested fix.
   - Hard rule: spec-anchored OR concrete-failure findings only;
     no preference findings.

2. Run <validator LLM>: e.g.,
   `codex exec -s read-only --ephemeral -o /tmp/llm-review-<area>-output.txt - < /tmp/llm-review-<area>-prompt.md`
   (timeout 600000)

3. Read /tmp/llm-review-<area>-output.txt.

4. Validate every finding by reading the cited code yourself.
   Classify each as:
   - CONFIRMED: code does what the LLM says, and that's a real bug
   - STRAWMAN: LLM misread the spec or the code
   - DEBATABLE: real ambiguity or judgment call

5. Identify missing tests: for each CONFIRMED finding, check whether
   an existing test would have caught it. If no, note "missing
   regression test for <case>".

6. Return a concise synthesis (≤600 words) with:
   - 1-line description of what you reviewed
   - Findings table: severity / area / file:line / verdict / failure mode
   - Missing-test list (per CONFIRMED finding)
   - Priority-ordered fix recommendation
   - No-bug-found areas the LLM explicitly confirmed clean

Clean up the temp files after extracting findings.

# Important

- Validate every finding before confirming. Garbage findings are
  worse than no findings — they erode trust in the rest of the report.
- Do not invent bugs. If everything checks out, say so plainly —
  that is a real signal worth surfacing.
- Temp file paths above are unique to this area; they will not
  collide with other parallel agents.
```

The temp file paths must be **unique per area** (e.g.,
`/tmp/llm-review-parser-prompt.md`, `/tmp/llm-review-data-prompt.md`,
…) so parallel agents don't clobber each other.

If a project doesn't have one of the artifacts the prompt references
(no SPEC.md, no CLAUDE.md), tell the agent that explicitly so it
falls back to general best-practices reasoning rather than
hallucinating spec content.

### Step D5: Aggregate as agents complete

Each agent returns a self-contained synthesis. As they complete:

1. Show the user a one-line summary per agent (severity counts +
   verdict counts) so they see incremental progress.
2. Don't try to re-validate findings the agent already validated.
   Trust its CONFIRMED/STRAWMAN/DEBATABLE classification.
3. Once all agents return, build the master synthesis.

### Step D6: Master synthesis

Aggregate all area-agent reports into a single document with:

```markdown
# Deep Cross-Review: <scope name>

**Mode**: deep | **Areas**: N | **Primary↔Validator**: <claude>↔<codex>
**Total findings**: M (X CONFIRMED, Y DEBATABLE, Z STRAWMAN)
**Areas with no bugs found**: <list>

## CONFIRMED — worth fixing (priority order)

| # | Sev | Area | file:line | Bug | Fix | Missing test? |
|---|---|---|---|---|---|---|
| 1 | HIGH | Parser | ... | ... | ... | yes — <case> |
| 2 | MED | … | … | … | … | yes/no |
…

## DEBATABLE — design choice or low reachability

…

## STRAWMAN — LLM misread the spec or code

(brief, mostly for transparency about what was filtered out)

## Missing test coverage summary

For each CONFIRMED finding without a regression test, list the test
that should be added (file + minimal scenario).

## Per-area no-bug-found summary

| Area | Categories the LLM explicitly cleared |
|---|---|
| Parser | UTC enforcement, frozen invariants, … |
| Data layer | URL construction, atomic writes, … |
| … | … |
```

End with a recommended next-step block:

```
Recommended next step:
- Bundle the M CONFIRMED fixes (HIGH + MED) into one commit with
  regression tests per finding. Want me to implement them now?
- DEBATABLE items: leave as-is unless you want to revisit the design.
- STRAWMAN items: filtered out, no action.
```

### Step D7: Clean up (deep mode)

Each area-agent cleaned up its own temp files. Verify nothing leaked:

```bash
ls /tmp/llm-review-*-prompt.md /tmp/llm-review-*-output.txt 2>/dev/null
# If anything remains, rm it.
```

### Anti-patterns (deep mode specifically)

- **Paraphrasing spec content**: the most common cause of strawman
  findings. Always quote verbatim.
- **Identical prompts across areas**: each area's deep-dive categories
  must be specific to its concerns. Generic prompts produce generic
  findings.
- **Skipping validation**: an unvalidated agent report is worth less
  than a single inline review. The validation step is the entire point
  of deep mode — do not skip it to save time.
- **Letting agents review files they don't own**: parser agent should
  not touch markout files, even if they're "related". Each agent owns
  its slice.
- **Re-running deep mode immediately after a fix commit**: the second
  pass overlaps heavily with the first. Wait until the codebase has
  meaningfully changed before re-running.

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
