# llm-assist

An agent skill that spawns an external LLM CLI as a cross-model thinking partner. It works with Claude Code, Cursor, Cline, Codex, and 30+ other AI coding agents.

## Why?

Using a single model to write and review code creates blind spots. Different model architectures catch different classes of bugs. This skill gives your AI agent an independent second opinion by shelling out to another LLM CLI such as Codex or OpenCode.

## Modes

| Mode | Use case |
|------|----------|
| `review` | Code review with cross-model false-positive filtering |
| `debug` | Independent bug investigation — compare diagnoses |
| `plan` | Second opinion on architecture or implementation approach |
| `verify` | Confirm a fix resolves the issue without regressions |
| `rca` | Root cause analysis with competing theories |
| `rescue` | Delegate to another LLM when stuck after repeated failures |
| `ask` | Freeform question — get a second perspective |

## Install

```bash
npx skills add wnz99/claude-skills/llm-assist -g
```

Or manually copy the `llm-assist/` directory to `~/.claude/skills/`.

## Prerequisites

Install and authenticate at least one supported provider:

- [Codex CLI](https://github.com/openai/codex) with a ChatGPT Plus/Pro subscription or OpenAI API key
- OpenCode CLI with a configured provider and model

```bash
npm i -g @openai/codex
codex login
```

```bash
npm i -g opencode
```

## Providers

- `codex` is the default provider
- `opencode` runs the same workflow through OpenCode
- `all` runs both and cross-compares the results

## Usage

### Explicit invocation

```
/llm-assist review                     — review uncommitted changes
/llm-assist debug <desc>               — investigate a bug
/llm-assist plan <desc>                — get feedback on an approach
/llm-assist verify <desc>              — validate a fix
/llm-assist rca <desc>                 — root cause analysis
/llm-assist rescue                     — hand off when stuck
/llm-assist ask <question>             — freeform question
/llm-assist --provider opencode review — run with OpenCode
/llm-assist --provider all review      — run both providers and compare
```

### Proactive triggering

The skill is designed so your agent can invoke it automatically when it:
- Has failed the same approach 3+ times
- Is unsure about platform-specific behavior
- Needs independent validation for security-sensitive changes
- Wants to cross-validate a code review

## How it works

1. Agent assembles context (diff, error messages, plan, etc.)
2. Writes a structured prompt to a temp file using quote-safe shell patterns
3. Runs the selected LLM CLI headlessly
4. Reads the output and **synthesizes** — comparing the external findings with its own analysis
5. Presents a unified result with agreement/disagreement labels

The agent never just passes through raw output. It always compares, validates, and synthesizes.

## Prompt Safety

The skill is designed to transport prompts safely even when they contain
apostrophes, backticks, shell-looking fragments, markdown fences, or large
multi-line diffs.

- Use a temp prompt file, not an inline shell string
- Use `cat <<'EOF'` for static template text
- Use `printf '%s\n' "$value"` or `cat file` for dynamic content
- Pass prompts via stdin or attached files to the external CLI
- Avoid `echo` for arbitrary prompt bodies and avoid `eval`

## License

MIT
