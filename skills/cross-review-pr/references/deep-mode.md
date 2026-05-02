# Deep Mode

Use this reference when `cross-review-pr` is invoked with `--deep` or the user
asks for a deep, multi-area, or parallel-agent review.

Deep mode has one precise meaning: split the scope into focused review areas,
run Reviewer A and Reviewer B independently on every area, validate findings
area-by-area in both directions, then synthesize. Do not satisfy deep mode with
one longer inline pass, one fanned-out reviewer plus one whole-scope pass, or
area agents that optionally call a validator model.

In hosts with restrictive delegation policy, "deep review" requests this
workflow but may not be explicit permission to spawn sub-agents. If the host
requires explicit permission for sub-agents, delegation, or parallel agent
work, ask for that permission before spawning. Do not silently use a fallback
such as external processes plus one local validation pass.

If the current harness cannot launch parallel agents, or if the user declines
sub-agents, say that strict deep mode is unavailable and ask whether to run a
normal comparative fallback instead.

## When To Use

- Critical PRs touching multiple subsystems.
- Periodic codebase-health audits over a source tree.
- Major refactors across module boundaries.
- A second pass after a default comparative review.

Skip deep mode for narrow changes such as one file or one function.

## D0. Shell And Command Preflight

Before generating prompts or running review commands, inspect the terminal
environment and choose command patterns that are safe for the active shell.
Do this before any command that builds file lists, loops over files, or embeds
diff/source content in prompt files.

Run a quick preflight:

```bash
printf 'SHELL=%s\n' "${SHELL:-unknown}"
ps -p $$ -o comm=
command -v bash || true
command -v zsh || true
```

If the current shell is `zsh`, do not rely on zsh word splitting. Either:

- run prompt-generation scripts under `bash`, or
- use shell-agnostic newline-safe loops and arrays.

Preferred pattern for generated review prompts:

```bash
bash <<'BASH'
set -euo pipefail

prompt_file="$1"
file_list="$2"

cat > "$prompt_file" <<'EOF'
# Deep Review Area
EOF

while IFS= read -r file; do
  [ -n "$file" ] || continue
  {
    printf '\n## Source: %s\n\n```text\n' "$file"
    sed -n '1,240p' "$file"
    printf '\n```\n'
  } >> "$prompt_file"
done < "$file_list"
BASH
```

Avoid patterns that depend on unquoted expansion or implicit splitting:

```bash
# Unsafe in zsh: may iterate once over the entire string or produce empty input.
for file in $FILES; do
  cat "$file" >> "$PROMPT_FILE"
done

# Unsafe for arbitrary file names and multiline values.
echo "$DIFF" >> "$PROMPT_FILE"
```

After generating every area prompt, validate it before launching reviewers:

```bash
wc -l "$PROMPT_FILE"
rg -n '^(diff --git|## Source:|<diff>)' "$PROMPT_FILE" | head
test "$(wc -l < "$PROMPT_FILE")" -gt 50
```

If a prompt is unexpectedly short or lacks source/diff markers, stop and
regenerate it under a known shell, preferably `bash`. Do not launch independent
or validation reviewers against empty or placeholder prompts.

## D0b. Delegation Authorization

Before decomposing areas or launching reviewers, decide whether strict deep
mode is authorized in the current host.

Strict deep mode requires parallel reviewer agents. If the host policy says
sub-agents/delegation/parallel agent work require explicit user authorization,
check the user's wording:

- Explicit enough: "spawn sub-agents", "use parallel reviewer agents",
  "delegate to agents", "parallel agent review".
- Not explicit enough by itself: "deep review", "deep comparative review",
  "`--deep`", "be thorough".

If authorization is missing, ask:

```text
Strict deep mode requires spawning parallel reviewer sub-agents. Do you want me
to spawn parallel reviewer sub-agents for this review?
```

To bypass this checkpoint, the user must explicitly ask for sub-agents or
parallel agent work in the original request, for example:

- "Run a deep parallel-agent PR review."
- "Run `cross-review-pr --deep` and spawn parallel reviewer sub-agents."
- "Delegate each deep-review area to a separate reviewer agent."

Only after the user says yes should you spawn area reviewer agents. If the user
says no, ask whether to run a clearly labeled non-deep comparative fallback.
Do not call a fallback "deep mode".

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
Launching 10 parallel reviewer agents: 5 Reviewer A agents and 5 Reviewer B agents.
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

## D4. Launch Symmetric Parallel Reviewers

Use the host agent's native parallel-agent mechanism. For each area, launch one
Reviewer A agent and one Reviewer B agent. Both reviewers receive the same
area scope, file list, project rules, and spec excerpts, but neither receives
the other reviewer's findings during the independent review pass.

For N areas, strict deep mode launches 2N independent reviewer agents. Example:
a six-area Claude <-> Codex review launches six Claude area reviewers and six
Codex area reviewers. Do not replace this with one model fanned out by area
while the other model performs one whole-PR pass or only validates.

Each area prompt must be self-contained and must tell the reviewer that other
agents are reviewing different areas.

Prompt skeleton:

```markdown
You are Reviewer <A-or-B> running an independent deep comparative review of one
area of <project>. The paired Reviewer <B-or-A> will review the same area
independently. Other reviewer pairs are reviewing other areas. Only review your
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
2. Do not ask another model to validate findings during this independent pass.
3. Do not read or infer the paired reviewer's findings.
4. For each finding, include severity, file:line, failure mode, concrete
   evidence, suggested fix, and whether an existing test would catch it.
5. End with an area verdict: Approved or Request Changes.

# Output
Return <=600 words:
- reviewed area
- reviewer: <A-or-B>
- findings table: severity / file:line / failure mode / evidence / fix
- missing regression tests
- no-bug-found categories, if any
```

Temp files must be unique per area. Do not let agents write to the same prompt
or output path.

## D5. Reciprocal Area Validation

After both independent reviews for an area complete, run reciprocal validation
for that same area:

1. Reviewer A validates Reviewer B's area findings against the same area scope.
2. Reviewer B validates Reviewer A's area findings against the same area scope.
3. The invoking agent records validation results before aggregating.

Validation may be inline when the reviewer role matches the invoking agent, or
through that reviewer's external CLI otherwise. The validating reviewer must
receive its own independent review, the other reviewer's findings, and the same
area context. It must not redo the full review.

Validation output for each other-reviewer finding:

- verdict: CONFIRMED / FALSE_POSITIVE / UNCERTAIN
- reasoning: concrete code/spec evidence
- missing test: yes / no / unclear

Do not aggregate an area as complete until both independent reviews are done.
If one validation direction fails, keep the independent reviews and successful
validation, but label the missing validation explicitly.

## D6. Aggregate

As agents complete, show a one-line status per area: severity counts and
validation counts: confirmed independently, A-only confirmed by B, B-only
confirmed by A, challenged, uncertain, and unvalidated.

If any area reviewer invokes an external CLI such as Claude Code or OpenCode,
do not stop that process just because it is taking a long time. Monitor process
state, elapsed time, output-file size, and recent output. Slow-but-progressing
external review work is not a hang. If progress is ambiguous, ask the user
whether to keep waiting or stop, and include enough detail for an informed
decision: elapsed time, process state, output size, recent output summary, and
what would be lost by stopping.

Master report:

```markdown
# Deep Cross-Review: <scope>

**Mode**: deep
**Areas**: N
**Reviewer A / Reviewer B**: <from> / <to>
**Reviewer agents**: 2N independent area reviewers
**Total findings**: M (X independently confirmed, Y cross-confirmed, Z challenged/uncertain/unvalidated)

## CONFIRMED - Worth Fixing

| # | Sev | Area | File | Bug | Fix | Missing test? |
|---|-----|------|------|-----|-----|---------------|

## DEBATABLE - Human Judgment Needed

## CHALLENGED / UNCERTAIN / UNVALIDATED

## FALSE POSITIVES - Filtered Out

## Missing Test Coverage

## Areas With No Confirmed Bugs
```

End by asking whether to implement the confirmed fixes. Every bug fix should
include a regression test that would have caught the original failure.

## Anti-Patterns

- Paraphrasing specs into stricter requirements.
- Giving identical generic prompts to every area.
- Fanning out one reviewer by area while the other reviewer performs one
  whole-scope pass or only validates.
- Letting area reviewers optionally invoke external validators instead of
  running symmetric independent Reviewer A and Reviewer B passes.
- Reporting unvalidated external-model findings as confirmed.
- Letting an area reviewer expand into unrelated files.
- Re-running deep mode immediately after a fix when the original area reports
  are still applicable.
