---
name: doc-review
description: Review and refresh an existing documentation file (runbook, README, AGENTS.md/CLAUDE.md, architecture doc, ADR, setup guide, API doc, etc.) against the live codebase to remove stale info, correct inaccurate claims, fill in missing pieces, and reorganize sections. Use whenever the user asks to "review docs", "update this doc", "check if this doc is still accurate", "audit a runbook", "is the README up to date?", "bring this doc in line with the code", "verify documentation against the codebase", or points at any markdown/MDX/rst file in `docs/`, `README.md`, `AGENTS.md`, `CLAUDE.md`, `.planning/`, `runbooks/`, or similar and asks for a freshness/correctness pass. Strongly prefer this skill over ad hoc rewrites — it enforces an evidence-based review where every change is backed by a citation to the live code. For authoring a brand-new doc from scratch, prefer `write-docs` instead. For regenerating a fixed set of project-template docs (GSD), prefer `gsd-docs-update`.
---

# Doc Review

A structured workflow for auditing one documentation file against the live codebase, then producing a precise, evidence-backed update.

The goal is not to rewrite the doc in your own voice. The goal is to make the doc match reality with the smallest, best-justified set of changes — so it stays trustworthy for the next person who relies on it.

## Announce on trigger

When this skill fires, post a single short line so the user can confirm the right workflow engaged:

> "Using **doc-review** on `<relative/path/to/doc>` — I'll inventory checkable claims, verify against the codebase, and report findings before editing."

Then proceed.

## When to use this — and when not to

**Use it for:** auditing/refreshing an existing documentation file. Typical triggers:

- "review `docs/runbooks/<name>.md`"
- "is this README still accurate?"
- "audit this doc, the code has moved on"
- "bring `AGENTS.md` up to date"
- "what's stale in `CONTRIBUTING.md`?"

**Don't use it for:**

- **Authoring a brand-new doc from scratch** — use `write-docs` (or just write it). This skill assumes there's existing content to audit.
- **Regenerating a fixed template-driven doc set** (e.g. GSD's `PROJECT.md`/`KNOWLEDGE.md`/etc.) — use `gsd-docs-update` which knows those templates.
- **Whole-repo doc sync** (extracting OpenAPI from routes, regenerating `features.md` from components) — use a repo-wide audit tool. This skill is per-document and surgical.

If the user has not named a specific file, ask which one — this skill is per-document, not project-wide.

## Operating principles

1. **Evidence before edits.** Never change a sentence based on memory. For each claim worth verifying, find the corresponding code, config, command, or schema and cite it (`path/to/file.ext:42` or a `Glob`/`Grep` hit). If you can't verify a claim, that itself is a finding — flag it, don't silently rewrite it.
2. **Smallest faithful diff.** Prefer surgical edits over wholesale rewrites. Preserve the doc's voice, structure, and existing examples when they're still correct. Reorganize only when the existing structure actively obscures the content.
3. **Treat omissions as findings too.** Missing steps, undocumented flags, new commands the codebase now exposes, removed services still mentioned — all count as staleness. Don't only look for what's wrong; look for what's missing.
4. **Distinguish stale from opinionated.** A claim that contradicts the code is stale. A claim that's a design recommendation or convention is opinion — leave it unless the user signals otherwise. When in doubt, ask.
5. **Respect repo-local conventions.** Before editing, read the nearest `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, or root `README.md` for any house rules (commit format, branch model, doc style, allowed commands). The doc's own repo is the source of truth — your generic instincts are not.
6. **Don't break the contract silently.** If the doc is referenced from indexes, sibling docs, CI, or other tooling, note any structural changes (renamed sections, removed anchors) so callers can be updated.

## Workflow

### Step 1 — Read the target doc end-to-end

Read the full file (no offset/limit unless it's enormous). Build a working list of *checkable claims*:

- Commands (build, test, deploy, install, lint, migrate, etc.)
- File paths and directory layouts
- Environment variables, secrets, config keys
- Function/class/module/package names
- URLs, account IDs, project names, resource identifiers
- CI workflow names, task-runner targets, package manager scripts
- Version pins, model IDs, SDK versions, language/runtime versions
- Step-by-step procedures with implicit ordering or preconditions
- Diagrams or tables that encode structure (e.g. service maps, env→domain tables)

Capture them as a checklist (use a TODO list for non-trivial docs). Each item will be verified or flagged.

Also note the doc's **conventions in passing**: heading depth, code-fence languages used, list markers (`-` vs `*`), callout style, tone. You'll match these when editing.

### Step 2 — Read repo-local guidance

Before grepping the codebase, read:

- The nearest `AGENTS.md` / `CLAUDE.md` (root and any subtree closer to the doc)
- `CONTRIBUTING.md` if present
- The repo's `README.md` if you don't already know the project layout

These often encode rules that override your defaults — preferred commands, branch model, supported package managers, doc style, what's out of scope. Do not skip this for unfamiliar repos.

### Step 3 — Map the doc to the codebase

For each checkable claim, find the source of truth. Prefer cheap, exact lookups first:

- `Glob` to confirm a file/dir still exists
- `Grep` to confirm a command, env var, symbol, or script is still defined where the doc says
- `Read` for the few files that own the behaviour (e.g. `package.json` / `pyproject.toml` / `Cargo.toml` / `Makefile` / build config / deploy manifest / the actual implementation file)
- `SemanticSearch` only when the doc describes a concept and you don't know where it lives

For large or unfamiliar areas, you may delegate to an exploration subagent — but stay strict that it returns concrete `file:line` evidence, not summaries.

### Step 4 — Classify each claim

Bucket every checkable claim into one of:

- **CORRECT** — verified against code; no change needed
- **STALE** — code disagrees; the doc is wrong
- **OUTDATED** — was correct, but a newer/preferred path now exists (e.g. a script was renamed, a workflow moved)
- **AMBIGUOUS** — partially right, or right but missing important nuance
- **UNVERIFIABLE** — can't find the code/config; flag for the user
- **MISSING** — code exposes something material that the doc never mentions

Only **STALE**, **OUTDATED**, **AMBIGUOUS**, and **MISSING** items become edits. **UNVERIFIABLE** items become questions for the user, not silent edits.

### Step 5 — Produce a review report *before* editing

Show the user a compact report and get explicit go-ahead before changing the file. Use this exact structure so it's predictable:

```markdown
# Doc Review: <relative/path/to/doc>

## Summary
- N claims checked
- N stale, N outdated, N missing, N unverifiable
- Overall verdict: <fresh | mostly fresh, minor drift | significant drift | substantially out of date>

## Findings

### 1. [STALE] <one-line claim, with section/line reference>
- **Doc says:** "<short quote>" (line 42)
- **Code says:** <citation, e.g. `src/foo.ts:88` or `package.json` `scripts.build`>
- **Proposed fix:** <one or two sentences describing the edit>

### 2. [MISSING] <topic the doc doesn't cover>
- **Evidence:** <citation>
- **Proposed addition:** <where it should live in the doc and what it should say>

… etc …

## Structural suggestions (optional)
<Only if reorganization would materially improve the doc. Otherwise omit this section.>

## Open questions
<UNVERIFIABLE items and any judgment calls that need the user's input.>
```

If the doc is essentially fresh (zero or near-zero findings), say so plainly and stop. Don't manufacture changes.

### Step 6 — Apply edits

Once the user approves (explicitly or by saying "go ahead"), apply the edits with surgical replacements. Rules:

- **One finding, one edit** where possible — keeps the diff reviewable.
- **Quote real commands and paths verbatim** — copy them from the source files you cited, don't retype from memory.
- **Update cross-references.** If you rename a heading the doc links to, fix the anchor. If another doc links to this one, mention it in the final summary so the user can decide whether to follow up.
- **Preserve formatting conventions.** Match the doc's existing style for code fences, list markers, callouts, and heading levels. If the doc uses `bash` fences, don't switch to `sh`.
- **Don't add filler.** No "this document describes…" intros, no "in conclusion" outros, no decorative emoji, no AI-tells. Match the doc's existing tone.
- **Do not touch unrelated content.** Whitespace-only churn and tone rewrites are out of scope unless the user asked for them.

For larger structural changes (reordering sections, splitting a doc), prefer to make them in a separate, clearly-labeled second pass after the correctness pass lands — and confirm with the user first.

### Step 7 — (Optional) Cold-read pass

For docs whose value is procedural — runbooks, setup guides, contributor onboarding, READMEs — do one final read top-to-bottom *as if you'd never seen the doc and had no other context*. Ask:

- Is the **goal** of this document clear in the first paragraph?
- Could a fresh reader execute the documented procedure end-to-end without getting stuck?
- Are prerequisites named before they're needed?
- Does any step assume tribal knowledge not stated anywhere?

Each gap is a new finding. Either fix it (small, obvious) or list it under "Open questions" for the user.

Skip this step for reference docs (API specs, schema dumps) where readability isn't the value proposition.

### Step 8 — Final summary

After edits, post a short summary:

- Files changed (with line counts if useful)
- One-line description per finding that was fixed
- Anything still open: unverifiable claims, deferred reorganization, downstream docs that may need follow-up
- Suggested next step (e.g. `git diff <file>`, run a smoke test the doc describes, update a sibling index)

Do not commit the changes. Leave that to the user unless they explicitly asked for a commit (then follow the repo's commit-message conventions from `CONTRIBUTING.md` / `AGENTS.md` — usually Conventional Commits).

## Adapting to the repo

This skill is intentionally generic. The repo's own files tell you how to behave:

| Look here | For |
|---|---|
| `AGENTS.md` / `CLAUDE.md` (root + nearest subtree) | House rules, allowed commands, routing, anti-patterns |
| `CONTRIBUTING.md` | Commit/PR conventions, branch model |
| `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod` / `Gemfile` | Canonical commands, dependency versions |
| `Makefile` / `justfile` / `Taskfile.yml` / `nx.json` | Task-runner truth |
| `.github/workflows/` / `.gitlab-ci.yml` / `.circleci/` | CI claims to verify |
| Infra-as-code (`terraform/`, `pulumi/`, `cdk/`, sibling infra repos) | Resource names, env mappings |
| Existing sibling docs in the same folder | Tone, structure, cross-link targets |

If a repo has rules that conflict with this skill (e.g. forbids file paths in trunk docs, requires a specific report template, mandates the doc be regenerated rather than edited), the **repo's rules win**. Acknowledge the conflict and adapt.

## Anti-patterns to avoid

- Rewriting prose because it "could be clearer" without an underlying correctness issue.
- Making changes without citing a file (you'll regress on the next pass).
- Bundling correctness fixes with structural reorganization in the same edit — the user can't tell what's a fact change vs. a layout change.
- Inventing commands, flags, or env vars that "ought to" exist. If it's not in the code, it doesn't go in the doc.
- Removing a section because you didn't find evidence for it in 30 seconds — escalate as UNVERIFIABLE first.
- Silently dropping warnings or caveats. Outdated warnings are still warnings until proven obsolete.
- Editing before reading the repo's `AGENTS.md` / `CONTRIBUTING.md`.
- Skipping the report step and going straight to edits — the report is what makes this trustworthy.

## A worked example

User: "Review `docs/runbooks/deploy-workers.md`."

1. **Announce** the skill, then read the runbook end-to-end. Extract claims: env→domain table, the deploy commands it shows, a note about manually-managed secrets, a per-env provisioning checklist.
2. Read the repo's root `AGENTS.md` — note that infra lives in a sibling repo and that secrets are out-of-scope for IaC there.
3. Verify each claim:
   - `Glob` for `apps/**/wrangler.jsonc` (or whatever the deploy config is) — confirm the env names the doc lists.
   - `Grep` for the exact deploy invocations the doc recommends — do `package.json` scripts still match?
   - Cross-check the sibling-infra-repo claim — flag UNVERIFIABLE if that repo isn't accessible.
4. Produce the report. Suppose three findings: a renamed env, a removed deprecated flag, and a new resource the runbook doesn't mention.
5. After approval, apply three small edits, each citing the config line that proves it.
6. Optional cold-read pass — confirm a fresh reader could follow the runbook end-to-end.
7. Summarize, note that `docs/runbooks/README.md` references this doc and might want a freshness-date bump.
