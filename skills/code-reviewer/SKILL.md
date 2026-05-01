---
name: code-reviewer
description:
  Use this skill to review code changes, including local staged/unstaged changes
  and remote pull requests by number or URL. Focus on bugs, security issues,
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

### 1. Determine Review Target

*   **Remote PR**: If the user provides a PR number or URL (e.g., "Review PR #123"), target that remote PR.
*   **Local Changes**: If no specific PR is mentioned, or if the user asks to "review my changes", target staged and unstaged local changes.

### 2. Preparation

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

### 3. In-Depth Analysis
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

### 4. Provide Feedback

#### Structure

*   **Findings first**: Lead with issues, ordered by severity. Include file/line references, impact, and why the issue is real.
*   **Severity labels**: Use Critical for bugs, security issues, or breaking changes; Improvement for meaningful quality or performance fixes; Nitpick only for small optional style comments.
*   **Open Questions / Assumptions**: Only include if they affect the verdict.
*   **Summary**: Brief change summary after findings, not before.
*   **Conclusion**: Clear recommendation (Approved / Request Changes).

#### Tone
*   Be direct, professional, and specific.
*   Explain *why* a change is requested.
*   Do not pad the review with praise.

### 5. Cleanup (Remote PRs only)
*   If you checked out a remote PR, return to the previous branch unless the user asked to stay on the PR branch.
