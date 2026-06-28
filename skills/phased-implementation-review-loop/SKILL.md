---
name: phased-implementation-review-loop
description: Use this skill whenever the user asks to implement a multi-step code change with a plan, execute it step by step, run sub-agent review loops after each step, or keep fixing High/Medium review issues until they are solved. This skill enforces phased planning, implementation, verification, independent code review, and rereading the plan before moving to the next step.
---

# Phased Implementation Review Loop

Use this skill for substantial implementation work where correctness depends on
controlled sequencing and independent review. The workflow exists to prevent the
authoring agent from drifting away from the plan or accepting its own blind
spots as proof.

## Preconditions

Before editing code:

1. Read the relevant project instructions, including the nearest `AGENTS.md`.
2. Read the issue, todo, plan, PR description, or runbook that defines the
   target behavior.
3. Identify the smallest owning subtree for the change.
4. Inspect the current implementation and tests enough to make a concrete plan.
5. If the user requested sub-agent reviews, delegate to a fresh-context
   sub-agent running the `code-reviewer` skill. Pick the reviewer model to match
   the environment this skill is running in — neither is authoritative, they are
   just the option available in each:
   - **Running via Codex:** Codex `gpt-5.4`, medium reasoning, fresh context.
   - **Running via Claude:** `sonnet`, fresh context.

## Output Plan

Create a written plan before implementation. Keep it concrete enough that each
step can be independently verified.

Use this shape:

```markdown
**Plan**
1. Step name
   Goal:
   Files likely touched:
   Tests/verification:
   Review target:
2. Step name
   Goal:
   Files likely touched:
   Tests/verification:
   Review target:
```

Make each step small enough that:

- The diff has a clear behavioral boundary.
- Focused tests can prove the step.
- A reviewer can understand the change without reading unrelated future work.

Do not include broad cleanup, opportunistic refactors, or unrelated style churn
unless they are necessary for the step.

## Step Loop

For every planned step, repeat this loop.

### 1. Re-read The Plan

Before changing code for the step:

- Re-read the full plan and the current step.
- Re-check the relevant code path to confirm the step still makes sense.
- If discoveries invalidate the plan, update the plan before editing and explain
  why.

### 2. Implement The Step

Use the repo's established patterns. Prefer TDD when the change has observable
behavior. Keep edits scoped to the current step.

When the step changes public behavior, update the relevant docs or runbooks in
the same step unless the plan intentionally separates documentation.

### 3. Verify Locally

Run focused verification for the changed behavior after the last code edit.

Verification should include, as appropriate:

- Unit tests for new pure logic and edge cases.
- CLI/help or dry-run tests for operator-facing commands.
- Type/lint checks for the touched subtree.
- Live provider or integration checks only when explicitly requested or already
  required by the task.

Do not claim the step is complete until fresh verification output exists after
the latest edit.

### 4. Review In Code Against The Plan

Before delegating review:

- Re-read the plan's current step.
- Inspect the diff and the implemented code path.
- Confirm every promised behavior for the step is present in code.
- Confirm tests exercise the behavior, not just implementation details.
- If anything is missing, fix it before asking for sub-agent review.

### 5. Run Independent Sub-agent Review

Delegate a fresh-context review of only the current step's diff and relevant
repository context.

Reviewer instructions:

- Use the `code-reviewer` skill.
- Run the reviewer on the model for the current environment (see Preconditions
  §5): via Codex → `gpt-5.4`, medium reasoning; via Claude → `sonnet`. Both are
  equally valid options; use whichever matches where this skill is running.
- Review for High and Medium issues first: correctness, data corruption,
  security, behavioral regressions, missing tests, broken contracts, and
  maintainability risks.
- Include file/line references, impact, evidence, and concrete fixes.
- Do not review the authoring agent's reasoning. Review the diff and codebase.

### 6. Fix And Loop

For every substantiated High or Medium finding:

1. Fix the issue in the current step.
2. Re-run focused verification.
3. Re-read the plan and inspect code again.
4. Run another independent review loop.

Stop the loop only when a fresh review after the latest fixes reports zero
unresolved High/Medium findings.

Low and Nit findings are optional unless they are cheap, useful, or explicitly
requested. Do not let optional polish expand the step.

### 7. Step Completion Record

After review is clean, record:

- What changed.
- Verification commands and result.
- Review loop count.
- Any remaining Low/Nit findings or accepted risks.
- Whether the plan needs adjustment before the next step.

Then move to the next planned step.

## Final Completion

After the last step:

1. Re-read the original plan and acceptance criteria.
2. Inspect the final code paths end to end.
3. Run the broadest appropriate local verification gate for the owning subtree.
4. Run a final independent review over the aggregate diff if the change spans
   multiple modules, contracts, or operator workflows.
5. Fix and loop until no High/Medium findings remain.
6. Summarize the result with verification evidence and known residual risks.

## Blockers

If a step cannot proceed because of missing credentials, unavailable services,
or a product decision, stop the step loop and report:

- The exact blocker.
- What was verified before the blocker.
- What remains unverified.
- The smallest decision or access needed to continue.

Do not mark the step complete while blocked.
