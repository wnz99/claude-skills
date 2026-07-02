---
name: code-reviewer
description:
  Use this skill to review code changes, including local staged/unstaged changes
  and remote pull requests by number or URL. Use this skill for semantic requests
  like "open a PR and do sub-agent loop reviews", "run review loops until issues
  are fixed", "review with subagents", "post review comments", or "keep reviewing
  until high and medium issues are solved." Focus on bugs, security issues,
  behavioral regressions, missing tests, maintainability, and project standards.
---

# Code Reviewer

Review code with a findings-first, evidence-backed approach. Prioritize real
bugs and regressions over style commentary.

## Terminal Awareness

Before running shell commands, inspect the active terminal environment and use
commands that are safe for that shell:

```bash
printf 'SHELL=%s\n' "${SHELL:-unknown}"
ps -p $$ -o comm=
command -v bash || true
command -v zsh || true
```

If the active shell is `zsh`, do not rely on implicit word splitting. Use
quoted variables, shell arrays, newline-safe loops, or run complex scripts
under `bash` with `set -euo pipefail`. This matters when building file lists,
reading changed files, or running verification commands.

## Workflow

### 1. Detect Operating Mode

Before choosing the review flow, classify the request:

*   **Single-pass review**: The user asks for a review, audit, PR review, or
    second pass without asking to create/update a PR or loop until fixes land.
    Use the normal workflow below.
*   **PR creation plus loop review**: The user asks to open/create a PR and run
    sub-agent reviews, loop reviews, repeated reviews, "until high/medium issues
    are solved", "until request-changes findings are gone", or similar semantic
    wording. Use the "PR Creation And Sub-Agent Loop Review" workflow.
*   **Existing PR loop review**: The user points at an existing PR and asks for
    loop/repeated/sub-agent reviews. Skip PR creation and start the loop against
    that PR.

### 2. Determine Review Target

*   **Remote PR**: If the user provides a PR number or URL (e.g., "Review PR #123"), target that remote PR.
*   **Local Changes**: If no specific PR is mentioned, or if the user asks to "review my changes", target staged and unstaged local changes.

### 3. Preparation

#### For Remote PRs:
1.  **Checkout**: Use the GitHub CLI to checkout the PR.
    ```bash
    gh pr checkout <PR_NUMBER>
    ```
2.  **Context**: Read the PR title, description, changed file list, and relevant discussion to understand the goal and history.
3.  **Project Instructions**: Read nearby project instructions (`AGENTS.md`, `CLAUDE.md`, or equivalent) before judging style or architecture.
4.  **Verification Signals**: If the project has an obvious local verification command, note it and run it only when appropriate for the review scope and environment. Do not assume `npm run preflight` exists.

#### For Local Changes:
1.  **Identify Changes**:
    *   Check status: `git status`
    *   Read diffs: `git diff` (working tree) and/or `git diff --staged` (staged).
2.  **Project Instructions**: Read nearby project instructions (`AGENTS.md`, `CLAUDE.md`, or equivalent).
3.  **Verification Signals**: Identify likely verification commands from package scripts, task runners, Makefiles, pyproject/poe tasks, Nx targets, or repo instructions. Run focused checks when useful and safe; otherwise state that verification was not run.

### PR Creation And Sub-Agent Loop Review

Use this workflow when the user asks to open a PR and run sub-agent loop reviews,
or uses similar wording. The goal is to keep the PR reviewable while converging
on zero unresolved high- or medium-severity findings.

#### A. Prepare And Open The PR

1.  Inspect `git status --short` and confirm the changed files are the intended
    scope. Do not include unrelated local changes in the PR.
2.  Read the relevant project instructions before committing or judging changes.
3.  Run focused verification that is appropriate for the changed subtree. If a
    repo-specific pre-commit/pre-push gate is required, run it before committing.
4.  Commit the intended changes with the repository's commit convention.
5.  Push the branch and open a PR with `gh pr create`. Include the verification
    evidence and known blocked checks in the PR body.
6.  Capture the PR number/URL for all later review comments.

If a PR already exists, update it instead of creating a duplicate.

#### B. Run Review Loops

Each loop has four phases: spawn independent review, post comments, fix, verify.
Every review loop must use a fresh reviewer context and the reviewer
command/model policy below, including loops run after pushed fixes.

1.  **Spawn independent sub-agent reviewers**
    *   Start from fresh context. The reviewer should see the repository, the PR
        diff, and the review instructions, not the authoring agent's reasoning.
    *   Select the reviewer sub-agent command for the active host:
        `multi_agent_v1.spawn_agent` for Codex/OpenAI when available, or the
        host's equivalent sub-agent command for Anthropic.
    *   Use an explicit reviewer model override: Codex/OpenAI reviewers use
        `gpt-5.4` with medium reasoning, and Anthropic reviewers use Sonnet 5.
        For Codex/OpenAI, pass `model: "gpt-5.4"` and
        `reasoning_effort: "medium"` when the sub-agent tool accepts those
        fields.
    *   Include the `code-reviewer` skill in the reviewer prompt or input items.
    *   Also follow any repository-specific review policy that does not
        conflict with this skill's reviewer command/model requirements.
    *   Ask each reviewer to classify findings as High, Medium, Low, or Nit.
        High and Medium are blocking. Low and Nit are optional unless the user
        explicitly says otherwise.
    *   Ask reviewers to return file/line references, impact, evidence, and a
        concrete fix suggestion for every High/Medium finding.

2.  **Post a PR comment for every loop**
    *   Post one top-level PR comment per loop, even when the loop finds no
        blocking issues.
    *   Include the loop number, reviewer identity, reviewer model, verification
        commands run, and a severity summary. If the host does not expose the
        exact model name, state the requested model from the reviewer
        command/model policy.
    *   For every High/Medium finding, include the file/line, impact, and planned
        resolution. If using inline review comments is practical, prefer inline
        comments for concrete code findings and still post the loop summary.
    *   If no High/Medium findings remain, explicitly state that the loop found
        no unresolved blocking findings.

3.  **Fix blocking findings**
    *   Resolve every substantiated High and Medium issue before starting the
        next loop.
    *   If a finding is incorrect or intentionally accepted, document the reason
        in the next loop comment and treat it as resolved only when the reasoning
        is concrete and evidence-backed.
    *   Do not churn on Low/Nit findings unless they are cheap, clearly useful,
        or requested by the user.

4.  **Verify and push**
    *   Rerun focused tests/checks relevant to the fixes.
    *   Commit and push fixes to the same PR.
    *   Start another fresh-context review loop after the push if any
        High/Medium finding was fixed, disputed, or newly introduced.

#### C. Stopping Criteria

Stop the loop only when one of these is true:

*   A fresh sub-agent review loop reports zero unresolved High/Medium findings.
*   The user explicitly stops or changes the task.
*   Progress is genuinely blocked by missing credentials, unavailable services,
    or a decision only the user can make. In that case, post a PR comment
    describing the blocker, what was already verified, and what input is needed.

Do not stop merely because one round of fixes was pushed. The final loop must be
a fresh review after the latest pushed commit.

#### D. Final User Report

Report the PR URL, loop count, final High/Medium status, verification evidence,
and any remaining Low/Nit notes or blocked checks. Keep the final response short;
the PR comments should contain the detailed loop history.

### 4. In-Depth Analysis
Analyze the code changes based on the following pillars:

*   **Correctness**: Does the code achieve its stated purpose without bugs or logical errors?
*   **Maintainability**: Is the code clean, well-structured, and easy to understand and modify in the future? Consider factors like code clarity, modularity, and adherence to established design patterns.
*   **Readability**: Is the code well-commented (where necessary) and consistently formatted according to our project's coding style guidelines?
*   **Efficiency**: Are there any obvious performance bottlenecks or resource inefficiencies introduced by the changes?
*   **Security**: Are there any potential security vulnerabilities or insecure coding practices?
*   **Edge Cases and Error Handling**: Does the code appropriately handle edge cases and potential errors?
*   **Testability**: Is the new or modified code adequately covered by tests (even if preflight checks pass)? Suggest additional test cases that would improve coverage or robustness.

#### Deep Cross-File Impact Analysis

When a change affects exported or externally consumed behavior, trace the
relevant call and dependency paths across files instead of reviewing each file
in isolation. Apply this to public functions, classes, modules, components,
handlers, CLI commands, jobs, adapters, shared types, configuration contracts,
persistence boundaries, external SDK/API calls, and documented invariants.

Build a lightweight directed map of the relevant program flow:

*   **Nodes**: Changed functions/classes/modules and important callers/callees.
*   **Edges**: Imports, direct calls, interface implementations, inheritance,
    dependency injection, factory/registry resolution, event/subscription
    wiring, routing, reflection, dynamic loading, or external API/SDK calls.

Walk the graph far enough to reach the user-facing entrypoint, persistence
boundary, external system boundary, or invariant boundary. Check whether intent
and contracts still propagate correctly across the chain, including:

*   Flags and modes such as force, dry-run, overwrite, locking, retry,
    pagination, authentication, authorization, caching, and idempotency.
*   Argument shape, return shape, nullability, error behavior, async/concurrency
    behavior, side effects, ordering assumptions, and resource ownership.
*   Semantic contract drift that type checkers may miss, especially around
    loose types, raw maps/dictionaries, JSON-like metadata, generated records,
    optional fields, erased generics, unchecked casts, or untyped external data.
*   Runtime reachability through dynamic mechanisms such as plugin loaders,
    registries, factories, service locators, reflection, string-based routing,
    dynamic imports/requires, or dependency injection containers.

For dynamic paths that cannot be proven statically, state the uncertainty and
name the runtime mechanism involved. Report concrete breakages, brittle
implicit contracts, or high-risk unverified paths; do not expand into unrelated
whole-repo review.

### 5. Provide Feedback

#### Structure

*   **Findings first**: Lead with issues, ordered by severity. Include file/line references, impact, and why the issue is real.
*   **Severity labels**: Use High for correctness, security, data corruption, or breaking-change issues that should block merge; Medium for meaningful behavioral, maintainability, reliability, or missing-test issues that should be fixed before merge; Low for useful but non-blocking improvements; Nit for small optional style comments.
*   **Open Questions / Assumptions**: Only include if they affect the verdict.
*   **Summary**: Brief change summary after findings, not before.
*   **Conclusion**: Clear recommendation (Approved / Request Changes). Request changes when any unresolved High or Medium finding remains.

#### Tone
*   Be direct, professional, and specific.
*   Explain *why* a change is requested.
*   Do not pad the review with praise.

### 6. Cleanup (Remote PRs only)
*   If you checked out a remote PR, return to the previous branch unless the user asked to stay on the PR branch.
