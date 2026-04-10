# claude-skills

A collection of agent skills for cross-model code review, debugging, and validation. Works with Claude Code, Cursor, Cline, Codex, and other AI coding agents that support the [Agent Skills](https://github.com/anthropics/skills) format.

## Skills

| Skill | Description |
|-------|-------------|
| [codex-assist](skills/codex-assist/) | Spawn OpenAI Codex CLI as a cross-model thinking partner for review, debug, plan, verify, RCA, rescue, and ask modes |
| [cross-review-pr](skills/cross-review-pr/) | Comparative PR review using Claude + Codex with cross-validation and confidence scoring |
| [code-reviewer](skills/code-reviewer/) | Structured code review for local changes and remote PRs (based on [google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli) code-reviewer) |

## Install

### All skills

```bash
npx skills add wnz99/claude-skills -g
```

### Individual skills

```bash
npx skills add wnz99/claude-skills/codex-assist -g
npx skills add wnz99/claude-skills/cross-review-pr -g
npx skills add wnz99/claude-skills/code-reviewer -g
```

Or manually copy any `skills/<name>/` directory to `~/.claude/skills/`.

## Prerequisites

- **codex-assist** and **cross-review-pr** require [Codex CLI](https://github.com/openai/codex) installed and authenticated:
  ```bash
  npm i -g @openai/codex
  codex login
  ```
- **cross-review-pr** also requires [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated
- **code-reviewer** works standalone with no extra dependencies

## How it works

These skills are designed to complement each other:

1. **code-reviewer** provides structured single-model review (correctness, security, maintainability, etc.)
2. **codex-assist** adds cross-model validation by running the same analysis through a different LLM architecture
3. **cross-review-pr** orchestrates both: runs code-reviewer first, then sends findings to Codex for independent validation and its own review, producing a unified report with confidence scores

The cross-model approach eliminates sycophancy bias and catches bugs that any single model might miss.

## Attribution

The **code-reviewer** skill is based on the code-reviewer from [google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli), adapted for use with Claude Code and cross-model workflows.

## License

MIT
