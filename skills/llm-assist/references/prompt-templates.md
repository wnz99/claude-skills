# Prompt Templates

Each mode uses a specific prompt structure. The calling agent assembles
the prompt file by combining project context with the mode-specific template.

## Prompt Assembly Safety

Render these templates into a prompt file with quote-safe shell patterns:

- Use `cat <<'EOF'` for static markdown sections.
- Append dynamic content with `printf '%s\n' "$value"` or `cat file`.
- Do not inline the full rendered prompt into a shell argument.
- Prefer stdin or attached files when handing the prompt to an external LLM.

## Common Header

**MANDATORY** - always start with this if CLAUDE.md (or equivalent project
instructions file) exists in the repo:

```markdown
# Project Coding Standards

The following are the project's coding conventions and guidelines.
You MUST apply these when analyzing code. Violations of these guidelines
are review findings, not style preferences.

<project-conventions>
[CLAUDE.md contents - prioritize coding guideline and convention sections:
language-specific guidelines, design patterns, code review standards,
architectural constraints, and quality requirements.
Trim non-guideline sections (build commands, deployment, project overview)
if over 4KB. Never omit the coding guideline sections.]
</project-conventions>
```

---

## Review

```markdown
# Code Review

## Skill Preference

Before starting the review, check if the `code-reviewer` skill is available
(look for SKILL.md at `~/.agents/skills/code-reviewer/SKILL.md` or
`~/.codex/skills/code-reviewer/SKILL.md` or `~/.claude/skills/code-reviewer/SKILL.md`).

- If `code-reviewer` is found: use its workflow to conduct the review instead
  of the generic instructions below. Pass it the diff and any focus area.
- If `code-reviewer` is NOT found: use the generic review instructions below.

## Review Target

Review the following diff for bugs, correctness issues, performance
problems, and style violations against the project conventions above.

## Focus
[User-specified focus, or "General review - check for correctness,
error handling, performance, security, and style."]

## Diff

<diff>
[git diff output]
</diff>

## Instructions (fallback if code-reviewer skill is not available)

For each finding, provide:
1. Severity: P0 (critical bug), P1 (significant), P2 (moderate), P3 (minor/style)
2. File path and line range
3. Clear description of the issue
4. Suggested fix

**Guideline compliance check (REQUIRED for every review):**
After checking for bugs and correctness, verify the changed code against the
project coding standards provided in the Project Coding Standards section above.
For each guideline that applies to the changed files, check whether the new code
complies. Flag violations as findings (typically P2 or P3). If no project
standards were provided, skip this step.

Be precise. Only report real issues — avoid speculative or low-confidence findings.
If something looks intentional, note it but don't flag it as a bug.
```

---

## Debug

```markdown
# Debug Investigation

Investigate this bug independently. Do NOT anchor on the theories below -
form your own hypothesis first, then compare.

## Bug Description
[Error message, symptoms, reproduction steps]

## Relevant Files
[File paths and key code sections]

## Current Theories (from another investigator)
[Calling agent's theories - provided for comparison AFTER the external LLM forms its own]

## Instructions

1. Read the relevant code paths
2. Form your own hypothesis about the root cause
3. Identify evidence for and against your hypothesis
4. Compare your conclusion with the theories above
5. State your diagnosis with confidence level (high/medium/low)
6. Suggest a specific fix if confident
```

---

## Plan

```markdown
# Architecture/Implementation Review

Review this proposed plan and provide an independent assessment.

## Plan
[The plan or approach being evaluated]

## Constraints
[Known constraints, requirements, deadlines]

## Uncertainties
[What the planner is unsure about]

## Instructions

1. Assess whether the plan achieves its stated goal
2. Identify risks or failure modes the planner may have missed
3. Suggest alternatives if you see a clearly better approach
4. Rate the plan: APPROVE / APPROVE-WITH-CONCERNS / RETHINK
5. If RETHINK, explain what specifically should change and why
```

---

## Verify

```markdown
# Fix Verification

Verify that this code change correctly resolves the described issue
without introducing regressions.

## Original Issue
[Bug description and reproduction steps]

## Fix (diff)

<diff>
[git diff of the fix]
</diff>

## Instructions

1. Confirm the fix addresses the root cause (not just symptoms)
2. Check for regressions - does the fix break any existing behavior?
3. Check for edge cases the fix might miss
4. Check that error handling is complete
5. Verdict: VERIFIED / CONCERNS / INSUFFICIENT
6. If CONCERNS or INSUFFICIENT, explain what's missing
```

---

## RCA (Root Cause Analysis)

```markdown
# Root Cause Analysis

Multiple theories exist for this bug. Investigate independently
and determine the most likely root cause.

## Bug Description
[Symptoms, affected code paths, when it occurs]

## Existing Theories

[Numbered list of theories with evidence for/against each]

## Instructions

1. Read the relevant code and form your own understanding
2. Evaluate each existing theory against the code
3. If you identify a different root cause, present it with evidence
4. Rank all theories (including any new ones) by likelihood
5. Recommend the specific code path to investigate or fix
```

---

## Rescue

```markdown
# Rescue - Fresh Approach Needed

The current investigator is stuck after multiple failed attempts.
Approach this problem from scratch.

## Problem
[What needs to be accomplished]

## Failed Attempts
[What was tried and why each failed]

## Constraints
[What must be preserved, what can change]

## Instructions

1. Do NOT repeat the failed approaches
2. Start from the requirements and work forward
3. Read the relevant code yourself - don't trust summaries
4. If you find a solution, implement it
5. If you also get stuck, explain what you learned and what
   the next investigator should try
```

---

## Ask

```markdown
# Question

[The question, with relevant code context]

## Instructions

Answer based on the actual codebase and current documentation.
If the answer depends on runtime behavior you can't verify,
say so and explain what to check.
```
