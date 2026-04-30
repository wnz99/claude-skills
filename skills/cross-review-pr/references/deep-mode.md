# Deep Mode

Use this reference when `cross-review-pr` is invoked with `--deep` or the user
asks for a deep, multi-area, or parallel-agent review.

Deep mode has one precise meaning: split the scope into focused review areas
and run reviewer agents in parallel. Do not satisfy deep mode with one longer
inline pass. If the current harness cannot launch parallel agents, say that
strict deep mode is unavailable and clearly label any fallback as a normal
comparative review.

## When To Use

- Critical PRs touching multiple subsystems.
- Periodic codebase-health audits over a source tree.
- Major refactors across module boundaries.
- A second pass after a default comparative review.

Skip deep mode for narrow changes such as one file or one function.

## D1. Resolve Scope

| Scope shape | How to resolve |
|-------------|----------------|
| PR number / URL | `gh pr diff "$PR_NUM" --name-only`, then read each file |
| Branch range (`main..HEAD`) | `git diff --name-only main..HEAD` |
| Directory path (`src/foo/`) | Use `find`/`rg --files` with project-relevant source extensions |
| Omitted | Default to `src/`, or the repo's source root if different |

If the resolved file list exceeds 50 files or 10,000 LOC, ask before
proceeding.

## D2. Decompose Areas

Split the file list into 4-5 areas by directory, layer, or concern. Each area
must be self-contained enough for a reviewer to analyze without reading the
whole codebase. Keep area sizes roughly balanced where possible.

State the split before launching agents:

```text
Deep review: 5 areas decomposed from src/
Area 1: parser inputs
Area 2: storage and persistence
Area 3: API contracts
Area 4: background jobs
Area 5: tests and fixtures
Launching 5 parallel reviewer agents.
```

## D3. Extract Source-Of-Truth Context

Deep-review findings should be spec-anchored or concrete-failure anchored.
Locate canonical context in this order:

1. `SPEC*.md`, `SPECS*.md`, or named product/spec docs.
2. `.planning/PROJECT.md`, `.planning/REQUIREMENTS.md`, or equivalent.
3. `CLAUDE.md`, `AGENTS.md`, or equivalent project instructions.
4. `docs/*.md`.
5. `README.md` as a last resort.

Quote only the relevant excerpts in each area prompt. Do not paraphrase a spec
into stricter requirements than it actually contains. If the spec is silent,
say so explicitly.

## D4. Launch Parallel Reviewers

Use the host agent's native parallel-agent mechanism. Each area prompt must be
self-contained and must tell the reviewer that other agents are reviewing
different areas.

Prompt skeleton:

```markdown
You are running an independent deep comparative review of one area of
<project>. You are reviewing in parallel with other agents; only review your
assigned files.

# Area
<name and responsibility>

# Files
- <absolute path 1>
- <absolute path 2>

# Project Rules
<relevant project instructions>

# Verbatim Spec Excerpts
<only relevant excerpts; say "No explicit spec found" if absent>

# Task
1. Review the assigned files for correctness, edge cases, security,
   integration bugs, and missing tests.
2. If using an external validator model, build a prompt with the same file
   list, project rules, and spec excerpts.
3. Validate every external finding by reading the cited code yourself.
4. Classify findings:
   - CONFIRMED: real bug or regression
   - STRAWMAN: model misread the spec or code
   - DEBATABLE: real ambiguity or design choice
5. For each CONFIRMED finding, identify whether an existing test would have
   caught it.

# Output
Return <=600 words:
- reviewed area
- findings table: severity / file:line / verdict / failure mode
- missing regression tests
- priority-ordered fixes
- no-bug-found categories, if any
```

Temp files must be unique per area. Do not let agents write to the same prompt
or output path.

## D5. Aggregate

As agents complete, show a one-line status per area: severity counts and
CONFIRMED/DEBATABLE/STRAWMAN counts. Trust each agent's validated
classification unless there is an obvious contradiction in the synthesis.

Master report:

```markdown
# Deep Cross-Review: <scope>

**Mode**: deep
**Areas**: N
**Primary/Validator**: <from> -> <to>
**Total findings**: M (X CONFIRMED, Y DEBATABLE, Z STRAWMAN)

## CONFIRMED - Worth Fixing

| # | Sev | Area | File | Bug | Fix | Missing test? |
|---|-----|------|------|-----|-----|---------------|

## DEBATABLE - Human Judgment Needed

## STRAWMAN - Filtered Out

## Missing Test Coverage

## Areas With No Confirmed Bugs
```

End by asking whether to implement the confirmed fixes. Every bug fix should
include a regression test that would have caught the original failure.

## Anti-Patterns

- Paraphrasing specs into stricter requirements.
- Giving identical generic prompts to every area.
- Reporting unvalidated external-model findings.
- Letting an area reviewer expand into unrelated files.
- Re-running deep mode immediately after a fix when the original area reports
  are still applicable.
