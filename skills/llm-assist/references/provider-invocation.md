# Provider Invocation

Use this reference when running an external LLM CLI from `llm-assist`.

## Install Commands

| Provider | Install | Auth |
|----------|---------|------|
| `claude` | `npm i -g @anthropic-ai/claude-code` | Run `claude auth login` or start `claude` and sign in |
| `codex` | `npm i -g @openai/codex` | Run `codex login` |
| `opencode` | `npm i -g opencode-ai` | Run `opencode providers` or configure `~/.config/opencode/opencode.json` |

## Temp Files

```bash
PROMPT_FILE=$(mktemp "${TMPDIR:-/tmp}/llm-assist-prompt.XXXXXX.md")
OUTPUT_FILE=$(mktemp "${TMPDIR:-/tmp}/llm-assist-result.XXXXXX.txt")
test -e "$PROMPT_FILE" && test -e "$OUTPUT_FILE"
case "$PROMPT_FILE $OUTPUT_FILE" in
  *XXXXXX*) echo "mktemp did not resolve correctly" >&2; exit 1 ;;
esac
```

Assemble prompts with quote-safe operations:

- Use `cat > "$PROMPT_FILE" <<'EOF'` for static sections.
- Append untrusted text with `printf '%s\n' "$value"` or `cat file >> "$PROMPT_FILE"`.
- Do not inline arbitrary diffs, logs, or markdown into shell command strings.
- Use shell arrays for command argv.

## Claude

Claude Code print mode accepts piped context. Use a fixed prompt argument and
pipe the prompt file on stdin:

```bash
claude_cmd=(
  claude
  -p
  "Follow the instructions provided on stdin."
  --verbose
  --output-format
  stream-json
  --include-partial-messages
)

"${claude_cmd[@]}" < "$PROMPT_FILE" > "$OUTPUT_FILE" 2>&1
```

For a new launch pattern, sanity-check the flags first:

```bash
printf 'Reply with exactly OK.\n' |
  claude -p "Follow the instruction on stdin." \
    --verbose \
    --output-format stream-json \
    --include-partial-messages
```

Do not use `--output-file`; current Claude Code CLI writes to stdout.

## Codex

Use read-only sandbox for review/debug/plan/verify/rca/ask:

```bash
codex exec \
  -s read-only \
  --ephemeral \
  -o "$OUTPUT_FILE" \
  - < "$PROMPT_FILE"
```

Use workspace-write only for rescue mode when implementation work is expected:

```bash
codex exec \
  -s workspace-write \
  --ephemeral \
  -o "$OUTPUT_FILE" \
  - < "$PROMPT_FILE"
```

`-o` is `--output-last-message`; it writes the final assistant message to the
file. Add `--json` only when you need event output on stdout.

## OpenCode

OpenCode uses `run` for non-interactive execution and `-f` for attached files:

```bash
opencode_cmd=(
  opencode
  run
  "Follow the instructions in the attached file"
  -f
  "$PROMPT_FILE"
  --format
  json
)

"${opencode_cmd[@]}" > "$OUTPUT_FILE" 2>&1
```

OpenCode does not have Codex-style per-invocation sandbox flags. It uses its
own permission and tool configuration. For write-capable rescue work, ensure
the selected OpenCode agent/config permits edits.

## Provider All

Run Claude and Codex with separate output files:

```bash
CLAUDE_OUTPUT=$(mktemp "${TMPDIR:-/tmp}/claude-result.XXXXXX.txt")
CODEX_OUTPUT=$(mktemp "${TMPDIR:-/tmp}/codex-result.XXXXXX.txt")
test -e "$CLAUDE_OUTPUT" && test -e "$CODEX_OUTPUT"
case "$CLAUDE_OUTPUT $CODEX_OUTPUT" in
  *XXXXXX*) echo "mktemp did not resolve correctly" >&2; exit 1 ;;
esac

claude -p "Follow the instructions provided on stdin." \
  --verbose \
  --output-format stream-json \
  --include-partial-messages \
  < "$PROMPT_FILE" > "$CLAUDE_OUTPUT" 2>&1 &
CLAUDE_PID=$!

codex exec -s read-only --ephemeral -o "$CODEX_OUTPUT" - < "$PROMPT_FILE" &
CODEX_PID=$!

wait "$CLAUDE_PID"
CLAUDE_STATUS=$?
wait "$CODEX_PID"
CODEX_STATUS=$?
```

Handle partial failure explicitly. If one provider succeeds and the other
fails, synthesize the successful result and state which provider failed.

## Monitoring

For long-running providers, record provider, PID, prompt path, output path, and
start time in a sidecar metadata file. Monitor with:

```bash
ps -o pid=,etime=,pcpu=,state=,command= -p "$PID"
wc -c "$OUTPUT_FILE"
tail -n 20 "$OUTPUT_FILE"
```

Quiet output is not failure. Kill only when the process exited non-zero,
exceeded the wait budget, disappeared unexpectedly, or shows provider-specific
evidence of being stuck.

## Failure Handling

| Error | Action |
|-------|--------|
| `claude: command not found` | Tell user to install Claude Code |
| `codex: command not found` | Tell user to install Codex CLI |
| `opencode: command not found` | Tell user to install OpenCode with `npm i -g opencode-ai` |
| Auth/API error | Ask user to fix authentication for the selected provider |
| Timeout | Report partial output if available and say the result is incomplete |
