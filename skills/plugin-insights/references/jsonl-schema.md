# JSONL session transcript schema

Each line in a session JSONL file is a JSON object. Relevant fields:

## Top-level fields

- `type`: One of `"user"`, `"assistant"`, `"progress"`, `"result"`
- `message.content`: String or array of content blocks
- `message.usage`: Token usage object (on assistant messages)
  - `input_tokens`: Tokens consumed reading input
  - `output_tokens`: Tokens generated
  - `cache_creation_input_tokens`: Tokens written to cache
  - `cache_read_input_tokens`: Tokens read from cache
- `timestamp`: ISO-8601 datetime string
- `sessionId`: Session identifier
- `uuid`, `parentUuid`: Message chaining
- `toolUseID`, `parentToolUseID`: Tool call correlation

## Content block types

Assistant messages contain an array of content blocks:

- `{ "type": "tool_use", "name": "Agent", "input": { "subagent_type": "...", "prompt": "...", "model": "..." } }` -- Agent spawning
- `{ "type": "tool_use", "name": "Skill", "input": { "skill": "...", "args": "..." } }` -- Skill invocation
- `{ "type": "tool_result", "content": "..." }` -- Tool execution result
- `{ "type": "text", "text": "..." }` -- Plain text output

## Token usage

Sum `message.usage` fields across all `assistant` messages to get session totals. Each assistant turn reports cumulative or incremental usage depending on the API mode.

## Subagent files

When agents are spawned, their transcripts are stored alongside the main session:

- Path: `<session-dir>/subagents/agent-<id>.jsonl`
- Metadata: `<session-dir>/subagents/agent-<id>.meta.json` containing `{ "agentType": "..." }`
- Same JSONL format as the main session
- Token usage available per turn

## Agent ID correlation

The main session's tool_result block for an Agent call contains the agent ID (format: `agentId: <hex-id>`). This ID matches the filename `agent-<id>.jsonl` in the `subagents/` directory. Use this to attribute subagent token usage to the correct plugin.

## Session duration

No explicit "session end" field exists. Calculate duration as the difference between the first and last timestamp in the JSONL file.

## Project identification

The project name can be derived from the JSONL file path: `~/.claude/projects/<project-path-encoded>/<session-id>.jsonl`. The project path uses hyphens to encode the directory separators.
