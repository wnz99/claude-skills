# codex-assist

An agent skill that spawns OpenAI Codex CLI as a cross-model thinking partner. Works with Claude Code, Cursor, Cline, Codex, and 30+ other AI coding agents.

## Why?

Using a single model to write and review code creates blind spots. Different model architectures catch different classes of bugs. This skill gives your AI agent an independent second opinion by shelling out to Codex CLI.

## Modes

| Mode | Use case |
|------|----------|
| `review` | Code review with cross-model false-positive filtering |
| `debug` | Independent bug investigation — compare diagnoses |
| `plan` | Second opinion on architecture or implementation approach |
| `verify` | Confirm a fix resolves the issue without regressions |
| `rca` | Root cause analysis with competing theories |
| `rescue` | Delegate to Codex when stuck after repeated failures |
| `ask` | Freeform question — get a second perspective |

## Install

```bash
npx skills add wnz99/claude-skills/codex-assist -g
```

Or manually copy the `codex-assist/` directory to `~/.claude/skills/`.

## Prerequisites

- [Codex CLI](https://github.com/openai/codex) installed and authenticated
- ChatGPT Plus/Pro subscription or OpenAI API key

```bash
npm i -g @openai/codex
codex login
```

## Usage

### Explicit invocation

```
/codex-assist review          — review uncommitted changes
/codex-assist debug <desc>    — investigate a bug
/codex-assist plan <desc>     — get feedback on an approach
/codex-assist verify <desc>   — validate a fix
/codex-assist rca <desc>      — root cause analysis
/codex-assist rescue          — hand off when stuck
/codex-assist ask <question>  — freeform question
```

### Proactive triggering

The skill is designed so your agent can invoke it automatically when it:
- Has failed the same approach 3+ times
- Is unsure about platform-specific behavior
- Needs independent validation for security-sensitive changes
- Wants to cross-validate a code review

## How it works

1. Agent assembles context (diff, error messages, plan, etc.)
2. Writes a structured prompt to a temp file
3. Runs `codex exec` in read-only sandbox (headless, non-interactive)
4. Reads the output and **synthesizes** — comparing Codex's findings with its own analysis
5. Presents a unified result with agreement/disagreement labels

The agent never just passes through raw Codex output. It always compares, validates, and synthesizes.

## License

MIT
