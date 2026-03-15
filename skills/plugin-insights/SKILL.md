---
name: plugin-insights
description: |
  Analyze how well Claude Code plugins perform across sessions.
  Use when: "plugin insights", "plugin performance", "how are my plugins doing",
  "plugin analysis", "plugin report", "which plugins work well".
  Generates an HTML report from JSONL session transcripts.
allowed-tools: Read, Glob, Grep, Bash(find:*), Bash(echo:*), Bash(ls:*), Bash(wc:*), Bash(mkdir:*), Agent, Write
---

## Iron Laws (NON-NEGOTIABLE)

1. NEVER read full JSONL files yourself. Delegate ALL detailed parsing to pi-session-parser agents.
2. NEVER do content quality assessment yourself. Delegate to pi-quality-reviewer agents.
3. ALWAYS write an HTML report file. No chat-only output.
4. Respect batch sizes: max 15 sessions per parser, max 15 reviews per quality-reviewer.
5. NEVER write the HTML report yourself. ALWAYS delegate report writing to the pi-report-writer agent. It has a fresh context and will follow the template exactly.

## Phase 1: Discovery and pre-filtering

1. Parse arguments: extract optional plugin filter and --days N (default 7).

2. If a plugin filter is given, validate it:
   - Use Grep to scan a sample of recent JSONL files (up to 20) for the exact plugin name in `subagent_type` or `command-name` patterns.
   - Collect all unique plugin names found (extract the part before the colon from `subagent_type` values).
   - If the given filter matches no plugin exactly, check for fuzzy matches:
     - Substring matches (e.g. "research" matches "deep-research")
     - Common abbreviations (e.g. "dr" could mean "deep-research")
   - If fuzzy matches are found, present them to the user: "Plugin 'X' not found. Did you mean one of these? 1. deep-research  2. review  3. ..." and wait for confirmation.
   - If no matches at all, report "No plugin matching 'X' found in recent sessions" and stop.

3. List all JSONL session files modified within the --days range:
   ```bash
   find ~/.claude/projects -name "*.jsonl" -maxdepth 2 -mtime -DAYS
   ```
   This returns only session files modified within the last N days. No Python needed.

4. Pre-filter sessions (ORCHESTRATOR does this, not agents):
   If a plugin filter is set, use Grep to scan each session file for the structural pattern `"name":\s*"Agent".*subagent_type.*<plugin-name>:` OR `<command-name>.*<plugin-name>:`. Use output_mode "count" for speed. Only keep files with at least 1 match.

   Report: "Found N sessions total, M contain <plugin-name> activity. Sending M sessions to parsers..."

   If no plugin filter: skip pre-filtering, send all sessions to parsers.

5. Group the filtered files into batches of 15.

## Phase 2: Parse (parallel batches)

Spawn pi-session-parser agents (model: sonnet). Each agent extracts invocations, behavioral metrics, token usage, and brief quality notes.

```
Agent(
  subagent_type: "plugin-insights:pi-session-parser",
  model: "sonnet",
  prompt: "Parse these JSONL files for plugin activity.\n\nFiles:\n<file-list>\n\n<optional: Only extract data for plugin: X>"
)
```

- Spawn up to 4 parser agents in parallel per round
- Collect structured JSON results from each agent
- On agent failure: log the error, continue with remaining batches

## Phase 3: Quality review (dynamic sampling)

Collect all `invocation_details` from parser results. Sort by timestamp (most recent first).

Determine sample size:
- Total invocations <= 10: review ALL
- Total invocations 11-30: review the 10 most recent
- Total invocations > 30: review the 10 most recent

Spawn a single pi-quality-reviewer agent (model: sonnet):

```
Agent(
  subagent_type: "plugin-insights:pi-quality-reviewer",
  model: "sonnet",
  prompt: "Review these plugin invocations for content quality.\n\nInvocations:\n<json-invocation-details>"
)
```

The quality reviewer receives per invocation:
- user_question (up to 800 chars)
- plugin_output_summary (up to 2000 chars)
- quality_note from parser (1-2 sentences)
- after_messages (5 messages)

It returns: relevance score (0-3), depth score (0-3), actionability score (0-3), what was missing, what was done well, and an overall verdict per invocation.

## Phase 4: Aggregate

1. Compute per-plugin aggregates from parser data:
   - Total invocations, sessions, unique projects
   - Token consumption (input + output)
   - Error rate (errors / invocations)
   - Behavioral metrics averages:
     - Retry rate (sessions with >1 invocation / total sessions)
     - Avg continuation messages (higher = user kept working = good)
     - Post-plugin error rate
   - Trend over time: bucket sessions by ISO week

2. Derive correction and completion scores from behavioral data (no LLM needed):
   - correction_score per session:
     - 0 (no correction): no same-topic retries AND errors_after == 0 AND next_actions contains no Edit/Write
     - 1 (minor): next_actions contains Edit or Write (manual follow-up) but no same-topic retry
     - 2 (significant): retry_same_topic == true (user re-ran with same question)
     - 3 (abandoned): continuation_messages == 0 AND session ended shortly after plugin call
   - completion_score per session:
     - 0 (abandoned): continuation_messages == 0 AND session ended within 2 minutes
     - 1 (partial): continuation_messages 1-2 AND next_actions contains manual work
     - 2 (complete): continuation_messages >= 1 AND no same-topic retry AND no errors_after (user moved on = task was done)
   - Note: "user moved on" is the strongest completion signal. If the user sends a new unrelated message or starts a new task, the plugin did its job.

   Retry rate calculation: only count sessions with retry_same_topic == true, not sessions where the user asked a different question to the same plugin.

3. Compute quality aggregates from reviewer data:
   - Average relevance, depth, actionability scores
   - Common "missing" themes
   - Common strengths
   - Notable verdicts (best and worst)

4. Mark sessions from projects whose path contains the plugin name as "dev sessions". Report dev vs. production stats separately.

5. Generate improvement suggestions:
   - Low relevance (avg < 1.5) -> "Plugin often misses what the user is asking for"
   - Low depth (avg < 1.5) -> "Results are too superficial, consider increasing analyst count or search depth"
   - Low actionability (avg < 1.5) -> "Results need more concrete recommendations or code examples"
   - High correction score (avg > 1.5) -> "Users frequently correct or redo plugin output"
   - Low completion (avg < 1.0) -> "Plugin results are often abandoned"
   - High retry rate (> 30%) -> "Users frequently need to re-run the plugin"
   - High error rate (> 20%) -> "Tool configuration or permission issues"

## Phase 5: Write HTML report (DELEGATED to report-writer agent)

Do NOT write the HTML report yourself. Delegate to a dedicated pi-report-writer agent. This agent has a fresh context with only the template and data, preventing instruction drift.

1. Determine the output file path:
   - Format: `~/.claude/plugin-insights/report-<plugin-name>-<YYYY-MM-DD-HHmm>.html`
   - With plugin filter: `report-deep-research-2026-03-15-1430.html`
   - Without filter: `report-all-2026-03-15-1430.html`

2. Build the aggregated data as a JSON string containing all metrics, scores, reviews, suggestions, and metadata (version "0.3.0", generated_date, date_range, etc.).

3. Spawn the report-writer agent:

```
Agent(
  subagent_type: "plugin-insights:pi-report-writer",
  model: "sonnet",
  prompt: "Write the plugin insights HTML report.\n\nTemplate file: <path-to-report-template.html>\nOutput file: <output-path>\n\nData:\n<json-aggregated-data>"
)
```

The template path is `${CLAUDE_SKILL_DIR}/report-template.html`. Pass the resolved absolute path to the agent.

4. Print the full path to the generated report file.

## Error handling

- If no JSONL files found in date range: report "No sessions found" and stop.
- If no plugin activity found: write a minimal report with the empty state message.
- If all parser batches fail: write an error report listing which batches failed and why.
- Individual session failures are acceptable. Log them but continue processing.

## Optional: Plugin-specific metrics

After Phase 2, check for plugin-specific metric files:
- `~/.claude/deep-research/metrics.jsonl`
- Similar patterns for other known plugins

If found, read and merge into aggregate data.
