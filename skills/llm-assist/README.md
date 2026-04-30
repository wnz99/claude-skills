# llm-assist

An agent skill that invokes an external LLM CLI as an independent thinking
partner for code review, debugging, planning, root cause analysis, fix
verification, and rescue when an agent is stuck.

Unlike the older Codex-only version of this skill, `llm-assist` is now
provider-aware:

- Running inside Codex defaults to Claude
- Running inside Claude defaults to Codex
- Running inside OpenCode asks which external provider to use
- OpenCode is opt-in, or an approved fallback when Claude or Codex is missing
- `--provider all` runs the standard cross-model pair: Claude + Codex

The skill builds a real prompt file, includes project instructions, invokes the
selected CLI safely, and synthesizes the external result instead of blindly
passing it through.

## Why use it

Single-model workflows have blind spots. The same agent that proposed a fix can
miss regressions, overfit to its own theory, or fail to notice a simpler
approach. `llm-assist` adds an independent model so your agent can:

- cross-check code review findings
- compare debugging theories
- validate plans before implementation
- verify a fix before declaring success
- escape repeated failure loops

The goal is not outsourcing judgment. The goal is better judgment through
independent comparison.

## Supported providers

Use `--provider` to select the external CLI explicitly when needed.

| Provider | Typical use | Default behavior | Install | Auth |
|----------|-------------|------------------|---------|------|
| `claude` | External reviewer when the current agent is Codex | Default inside Codex | `npm i -g @anthropic-ai/claude-code` | Claude Code auth |
| `codex` | External reviewer when the current agent is Claude | Default inside Claude | `npm i -g @openai/codex` | ChatGPT subscription or OpenAI API key |
| `opencode` | Explicit opt-in or approved fallback | Never automatic unless the user approves fallback from missing Claude/Codex | `npm i -g opencode` | Configured in `~/.config/opencode/opencode.json` |
| `all` | Highest-confidence cross-check | Explicit only | Install both Claude Code and Codex | Both configured |

### Default provider rules

- If the running agent is Codex, default to `claude`
- If the running agent is Claude, default to `codex`
- If the running agent is OpenCode, ask which external model to use
- If the preferred Claude/Codex CLI is unavailable, check that first and offer
  OpenCode only as an explicit fallback
- If `--provider all` is used, run Claude and Codex in parallel and label
  findings as `CLAUDE`, `CODEX`, or `BOTH`

## Modes

| Mode | Use case |
|------|----------|
| `review` | Code review with false-positive filtering and independent validation |
| `debug` | Independent bug investigation and diagnosis comparison |
| `plan` | Second opinion on architecture or implementation approach |
| `verify` | Confirm a fix resolves the issue without new regressions |
| `rca` | Root cause analysis with competing theories |
| `rescue` | Delegate after repeated failed attempts |
| `ask` | Freeform second opinion on code, platform behavior, or libraries |

In Codex, all modes except `rescue` use a read-only sandbox. `rescue` may use
workspace-write because it can require concrete implementation work.

## Install

```bash
npx skills add wnz99/claude-skills/llm-assist -g
```

Or copy `skills/llm-assist/` into your local skills directory, such as:

- `~/.claude/skills/`
- `~/.codex/skills/`
- `~/.agents/skills/`

## Prerequisites

Install and authenticate at least one supported provider.

### Claude Code

```bash
npm i -g @anthropic-ai/claude-code
```

### Codex

```bash
npm i -g @openai/codex
codex login
```

### OpenCode

```bash
npm i -g opencode
```

OpenCode must also be configured with a provider and model in:

```text
~/.config/opencode/opencode.json
```

## Usage

### Default behavior

```text
/llm-assist review
/llm-assist debug "host disconnects after codec renegotiation"
/llm-assist plan "compare WebRTC vs QUIC control channel approach"
/llm-assist verify "confirm this reconnect fix covers the original bug"
/llm-assist rca "why does capture freeze only on macOS"
/llm-assist rescue
/llm-assist ask "is this ffmpeg API usage still valid"
```

### Explicit provider selection

```text
/llm-assist --provider claude review
/llm-assist --provider codex review
/llm-assist --provider opencode review
/llm-assist --provider all review
```

### When to use `--provider all`

Use `--provider all` when:

- the task is high-stakes and benefits from two independent reviewers
- you want a cleaner signal on whether a review finding is real
- you want Codex and Claude to analyze the same issue in parallel

`all` means Claude + Codex. OpenCode is not included unless you explicitly ask
for it.

## When the skill should trigger

This skill is designed for both explicit invocation and proactive use by the
running agent.

Typical proactive triggers:

- the agent has failed the same approach 3+ times
- the agent has low confidence in its own answer
- the issue is platform-specific and hard to verify locally
- the change is security-sensitive or otherwise high-stakes
- the problem looks like a race condition or another subtle systems bug
- the user explicitly asks for a second opinion, Codex review, Claude review,
  OpenCode review, or external help

When triggered proactively, the agent should say what it is doing and why
before invoking the external model.

## How it works

1. The agent collects only the context needed for the selected mode.
2. It writes a structured prompt to a temp file.
3. It includes project instructions from `CLAUDE.md` or the equivalent project
   instruction file.
4. It invokes the selected CLI headlessly and monitors output in real time.
5. It reads the results and synthesizes them with its own analysis.
6. It reports agreement, disagreement, missed findings, and next actions.

The external output is never meant to be pasted back raw. The running agent is
still responsible for evaluating and integrating the result.

## Prompt and shell safety

One of the main reasons to use this skill instead of an ad hoc CLI call is safe
prompt transport.

The skill is built to handle:

- multi-line diffs
- stack traces
- markdown fences
- apostrophes and quotes
- shell-looking fragments like `$()` and backticks

Core safety rules:

- do not inline arbitrary prompts into shell command strings
- do not rely on `echo "$PROMPT"` for complex prompt bodies
- use temp files and single-quoted heredocs for static prompt sections
- append dynamic content with `printf '%s\n' "$value"` or `cat file`
- pass prompts via stdin or attached files
- avoid brittle command-string concatenation and `eval`

## Review behavior

`review` mode is more than “ask another model to look at the diff.”

The current workflow is designed to:

- determine the correct review scope
- include project-specific coding standards
- prefer the `code-reviewer` skill if the external environment has it
- compare external findings against the invoking agent's own analysis
- classify findings by agreement level instead of dumping raw output

When running with `--provider all`, issues found by both providers carry the
highest confidence.

## Operational behavior

The current skill text also adds two important behavioral rules:

- Prefer this skill over ad hoc direct CLI calls when the task clearly matches
  review, debug, planning, verification, RCA, rescue, or freeform external
  consultation.
- After invoking the external model, wait for the result or a real failure
  before starting unrelated new code changes.

That makes the workflow more reliable and avoids fake “the external LLM hung”
diagnoses caused by poor prompt assembly or missing context.

## Output model

The external provider is a thinking partner, not an oracle.

Expected outputs include:

- `CONFIRMED`: both agents agree
- `PROVIDER-ONLY`: the external model found something the caller missed
- `CLAUDE-ONLY` or analogous caller-only findings: the caller found something
  the provider missed
- `DISAGREEMENT`: both sides analyzed the issue differently

The final answer should explain those differences, not hide them.

## License

MIT
