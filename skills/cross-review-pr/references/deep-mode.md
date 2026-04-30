# Deep Mode

Use this reference when `cross-review-pr` is invoked with `--deep` or the user
asks for a deep, multi-area, or parallel-agent review.

Deep mode has one precise meaning: split the scope into focused review areas
and run reviewer agents in parallel. Do not satisfy deep mode with one longer
inline pass.

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
regenerate it under a known shell, preferably `bash`. Do not launch primary or
validator reviewers against empty or placeholder prompts.

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
