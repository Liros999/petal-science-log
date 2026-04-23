# Conversation transcripts — full Claude Code session archives

These `.jsonl` files are the complete, turn-by-turn Claude Code session
transcripts corresponding to Science Log entries. Each message, tool
call, and tool result is preserved in the JSON lines format, timestamped.

## Index

| Session ID | Date | Topic | Science Log Entries |
|---|---|---|---|
| `2ff73718-c8a2-4d78-99fd-d96bd5dec143` | 2026-04-22 / 2026-04-23 | 24-hour triangle / clique / pivot / backbone / cycles / ploidy sprint (exp 131–219) | Entries 23–31 |

## How to read

Each line is a JSON object with a `type` field (user, assistant, tool_use,
tool_result) and a `message` / `content` payload. To inspect:

```bash
jq -c '.[type]' conversation_*.jsonl | sort | uniq -c
jq 'select(.type == "user") | .message.content' conversation_*.jsonl
```

Or re-ingest into any LLM with session-resume support.

## Why archive this

The 24-hour triangle sprint produced 50+ experiments and ~12 paper-ready
claims. The transcript captures the full reasoning trail — which ideas
worked, which were falsified, which data we checked, what the user's
corrections were. Useful for:

- Writing the paper methods/results sections (exact statistical tests, n,
  p, OR, CI — all recorded here)
- Reproducing experiments from the prompts that produced the scripts
- Reviewing user's "no not that" corrections for methodology rules
