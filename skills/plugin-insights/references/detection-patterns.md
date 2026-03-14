# Plugin detection patterns

Three signal sources identify plugin activity in JSONL session transcripts.

## 1. subagent_type with colon separator

In `type: "assistant"` messages, look for content blocks with `type: "tool_use"` where `name: "Agent"`. The `input.subagent_type` field uses the format `<plugin-name>:<agent-type>`.

Examples:
- `deep-research:dr-analyst` -> plugin `deep-research`
- `kickstart:init-verifier` -> plugin `kickstart`
- `review:scope-checker` -> plugin `review`

Ignore built-in types without a colon: `Explore`, `Plan`, `general-purpose`, `Bash`, and any other single-word types.

## 2. command-name tags

In `type: "user"` messages, look for `<command-name>` XML tags. Format: `<command-name>/plugin:command</command-name>`. The plugin name is the part before the colon.

Examples:
- `<command-name>/deep-research:deep-research</command-name>` -> plugin `deep-research`
- `<command-name>/kickstart:new</command-name>` -> plugin `kickstart`
- `<command-name>/review:review</command-name>` -> plugin `review`

## 3. hook_progress entries

In `type: "progress"` entries with `data.hookName`. Plugin-specific hooks reference the plugin name in the command path (`${CLAUDE_PLUGIN_ROOT}/...`). These are less reliable since the plugin name only appears in the path string.

## Attribution rule

A plugin counts as "active in session X" when at least one signal of type 1 or 2 appears in that session. Type 3 signals are supplementary and not sufficient on their own.

When extracting the plugin name, always take the part before the first colon in `subagent_type` or between the last `/` and `:` in `command-name` tags.
