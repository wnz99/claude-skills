# claude-skills

A collection of Agent Skills for cross-model code review, debugging, and validation. These skills target agents that support the [Agent Skills](https://agentskills.io/) format or can load `SKILL.md` folders.

## Skills

| Skill | Description |
|-------|-------------|
| [llm-assist](skills/llm-assist/) | Spawn an external LLM CLI (Claude, Codex, or OpenCode) as a cross-model thinking partner for review, debug, plan, verify, RCA, rescue, and ask modes |
| [cross-review-pr](skills/cross-review-pr/) | Bidirectional comparative PR review between any two LLMs (Claude, Codex, OpenCode) with cross-validation and confidence scoring |
| [code-reviewer](skills/code-reviewer/) | Structured code review for local changes and remote PRs (based on [google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli) code-reviewer) |

## Install

### All skills

```bash
npx skills add wnz99/claude-skills -g
```

### Individual skills

```bash
npx skills add wnz99/claude-skills/llm-assist -g
npx skills add wnz99/claude-skills/cross-review-pr -g
npx skills add wnz99/claude-skills/code-reviewer -g
```

Or manually copy any `skills/<name>/` directory to your agent's skills
directory, such as `~/.claude/skills/`, `~/.codex/skills/`, or
`~/.agents/skills/`.

## Prerequisites

- **llm-assist** and **cross-review-pr** require at least one external LLM CLI installed:
  - [Claude Code](https://code.claude.com/docs/en/cli-reference): `npm i -g @anthropic-ai/claude-code && claude auth login`
  - [Codex CLI](https://github.com/openai/codex): `npm i -g @openai/codex && codex login`
  - [OpenCode](https://dev.opencode.ai/docs/): `npm i -g opencode-ai`
- **cross-review-pr** also requires [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated
- **code-reviewer** works standalone with no extra dependencies

## How it works

These skills are designed to complement each other:

1. **code-reviewer** provides structured single-model review (correctness, security, maintainability, etc.)
2. **llm-assist** adds cross-model validation by running analysis through a different LLM architecture (Codex or OpenCode)
3. **cross-review-pr** orchestrates both: one LLM reviews first, then the other validates findings and adds its own, producing a unified report with confidence scores

**cross-review-pr** supports any combination of Claude, Codex, and OpenCode as primary reviewer or validator via `--from` / `--to` flags. The default is `--from claude --to codex`. Running `--from codex --to claude` reverses the roles — Codex reviews first, Claude validates.

The cross-model approach helps reduce sycophancy bias and can catch bugs that any single model might miss.

## Attribution

The **code-reviewer** skill is based on the code-reviewer from [google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli), adapted for use with Claude Code and cross-model workflows.

## License

MIT
