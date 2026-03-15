---
name: plugin-insights
description: |
  Analyze how well Claude Code plugins perform across sessions.
  Use when: "plugin insights", "plugin performance", "how are my plugins doing",
  "plugin analysis", "plugin report", "which plugins work well".
  Generates an HTML report from JSONL session transcripts.
allowed-tools: Read, Glob, Grep, Bash(find:*), Bash(echo:*), Bash(ls:*), Bash(wc:*), Bash(mkdir:*), Agent, Write
---

## STOP — Parse arguments first

Arguments received: $ARGUMENTS

Before ANY other action, extract from the arguments above:
- **Plugin filter**: any word that is not a flag (e.g. "review", "deep-research"). This is the PLUGIN_FILTER.
- **--days N**: number of days to scan. Default: 7.

If a plugin filter is present, set PLUGIN_FILTER to that word and use it in ALL subsequent phases.
If no arguments or empty: PLUGIN_FILTER = none, scan all plugins.

## REQUIRED OUTPUT (print this before doing anything else)

```
PLUGIN_FILTER = <extracted filter or "none">
DAYS = <extracted days or "7">
```

You MUST print the block above as your first output. If you cannot fill in both values, re-read the $ARGUMENTS line. You CANNOT call any tool, run any command, or proceed to Phase 1 until this block is printed.

## Iron Laws (NON-NEGOTIABLE)

1. NEVER read full JSONL files yourself. Delegate ALL detailed parsing to pi-session-parser agents.
2. NEVER do content quality assessment yourself. Delegate to pi-quality-reviewer agents.
3. ALWAYS write an HTML report file. No chat-only output.
4. Respect batch sizes: max 15 sessions per parser, max 15 reviews per quality-reviewer.
5. NEVER write the HTML report yourself. ALWAYS delegate to pi-report-writer agent.
6. ALWAYS pass PLUGIN_FILTER to parser agents with exact phrase: "Only extract data for plugin: PLUGIN_FILTER".
7. ALWAYS use model: "sonnet" when spawning pi-session-parser and pi-quality-reviewer agents. Never inherit the parent model.
8. Phase 3 (Quality Review) is NEVER optional. Even with 1 invocation, spawn the quality-reviewer.
9. Report output path is ALWAYS `~/.claude/plugin-insights/report-<name>-<YYYY-MM-DD-HHmm>.html`. Never write to the current working directory.
10. ALWAYS pass the template file path AND structured JSON data to the report-writer agent.

## Anti-skip rules

| Thought | Reality |
|---------|---------|
| "No plugin filter given" | Check the REQUIRED OUTPUT block. If PLUGIN_FILTER is not "none", use it. |
| "I'll filter later during aggregation" | Pre-filtering in Phase 1 saves 80% of token costs. Do it now. |
| "Let me start finding files first" | STOP. Print the REQUIRED OUTPUT block first. |
| "I'll scan everything and filter later" | NO. Pre-filter is MANDATORY when PLUGIN_FILTER is set. |
| "Only 3 invocations, skip quality review" | NO. Phase 3 is mandatory regardless of count (Iron Law 8). |
| "I'll use the default model" | NO. Explicitly set model: "sonnet" on every agent spawn (Iron Law 7). |
| "I'll write the report to the current directory" | NO. Always ~/.claude/plugin-insights/ (Iron Law 9). |

## Phase 1: Discovery and pre-filtering

1. Verify you printed the REQUIRED OUTPUT block. If not, go back and print it now.

2. If PLUGIN_FILTER is set, validate it:
   - Use Grep to scan a sample of recent JSONL files (up to 20) for the exact plugin name in `subagent_type` or `command-name` patterns.
   - Collect all unique plugin names found (extract the part before the colon from `subagent_type` values).
   - If the given filter matches no plugin exactly, check for fuzzy matches:
     - Substring matches (e.g. "research" matches "deep-research")
     - Common abbreviations (e.g. "dr" could mean "deep-research")
   - If fuzzy matches are found, present them to the user: "Plugin 'X' not found. Did you mean one of these? 1. deep-research  2. review  3. ..." and wait for confirmation.
   - If no matches at all, report "No plugin matching 'X' found in recent sessions" and stop.

3. List all JSONL session files modified within the DAYS range:
   ```bash
   find ~/.claude/projects -name "*.jsonl" -maxdepth 2 -mtime -DAYS ! -path "*/subagents/*"
   ```

4. **MANDATORY pre-filter** (if PLUGIN_FILTER is set):
   Use the Grep tool (NOT bash grep) on each session file. Copy-paste this exact pattern:

   ```
   Grep(pattern: "subagent_type.*PLUGIN_FILTER:", path: "<file>", output_mode: "count")
   ```

   Also check for command-name:
   ```
   Grep(pattern: "command-name.*PLUGIN_FILTER:", path: "<file>", output_mode: "count")
   ```

   Only files where either grep returns >= 1 match go to parsers. Discard all others.

   Print: "Found N sessions total, M contain PLUGIN_FILTER activity. Sending M to parsers in B batches..."

   If PLUGIN_FILTER is "none": skip pre-filtering, send all sessions to parsers.

5. Group the filtered files into batches of 15.

## Phase 2: Parse (parallel batches)

Spawn pi-session-parser agents. MUST use model: "sonnet" explicitly.

```
Agent(
  subagent_type: "plugin-insights:pi-session-parser",
  model: "sonnet",
  prompt: "Parse these JSONL files for plugin activity.\n\nFiles:\n<file-list>\n\nOnly extract data for plugin: PLUGIN_FILTER"
)
```

If PLUGIN_FILTER is "none", omit the "Only extract data for plugin" line.

- Spawn up to 4 parser agents in parallel per round
- Collect structured JSON results from each agent
- On agent failure: log the error, continue with remaining batches

## Phase 3: Quality review (MANDATORY, never skip)

This phase is NOT optional. Even with 1 invocation, run the quality reviewer.

Collect all `invocation_details` from parser results. Sort by timestamp (most recent first).

Determine sample size:
- Total invocations <= 10: review ALL
- Total invocations 11-30: review the 10 most recent
- Total invocations > 30: review the 10 most recent

Spawn a single pi-quality-reviewer agent. MUST use model: "sonnet" explicitly:

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
     - Retry rate (sessions with retry_same_topic == true / total sessions)
     - Avg continuation messages
     - Post-plugin error rate
   - Trend over time: bucket sessions by ISO week

2. Derive correction and completion scores from behavioral data:
   - correction_score per session:
     - 0: no same-topic retries AND errors_after == 0 AND next_actions contains no Edit/Write
     - 1: next_actions contains Edit or Write but no same-topic retry
     - 2: retry_same_topic == true
     - 3: continuation_messages == 0 AND session ended shortly after
   - completion_score per session:
     - 0: continuation_messages == 0 AND session ended within 2 minutes
     - 1: continuation_messages 1-2 AND next_actions contains manual work
     - 2: continuation_messages >= 1 AND no same-topic retry AND no errors_after

3. Compute quality aggregates from reviewer data:
   - Average relevance, depth, actionability scores
   - Common "missing" themes
   - Common strengths
   - Notable verdicts (best and worst)

4. Mark sessions from projects whose path contains the plugin name as "dev sessions". Report separately.

5. Generate improvement suggestions per the threshold rules:
   - Low relevance (avg < 1.5) -> "Plugin often misses what the user is asking for"
   - Low depth (avg < 1.5) -> "Results are too superficial"
   - Low actionability (avg < 1.5) -> "Results need more concrete recommendations"
   - High correction (avg > 1.5) -> "Users frequently correct or redo plugin output"
   - Low completion (avg < 1.0) -> "Plugin results are often abandoned"
   - High retry rate (> 30%) -> "Users frequently need to re-run the plugin"
   - High error rate (> 20%) -> "Tool configuration or permission issues"

6. Build the aggregated data as a JSON object with this exact structure:

```json
{
  "metadata": {
    "version": "0.3.1",
    "generated_date": "YYYY-MM-DD",
    "date_range": "YYYY-MM-DD to YYYY-MM-DD",
    "days": 7,
    "plugin_filter": "review",
    "total_sessions_scanned": 113,
    "sessions_with_activity": 5
  },
  "plugins": {
    "<name>": {
      "invocations": 10,
      "sessions": 5,
      "projects": 3,
      "error_rate": 0.0,
      "tokens": { "input": 50000, "output": 12000 },
      "behavioral": {
        "retry_rate": 0.1,
        "avg_continuation": 5.2,
        "post_error_rate": 0.0,
        "avg_correction_score": 0.4,
        "avg_completion_score": 1.8
      },
      "quality": {
        "avg_relevance": 2.5,
        "avg_depth": 2.3,
        "avg_actionability": 2.1
      },
      "agent_types": ["type-a", "type-b"],
      "trend_weeks": [{"week": "W10", "count": 3}, {"week": "W11", "count": 7}],
      "projects_list": [{"name": "...", "sessions": 2, "invocations": 5, "is_dev": false}],
      "quality_reviews": [...],
      "strengths": ["..."],
      "improvements": ["..."],
      "suggestions": ["..."]
    }
  }
}
```

Pass this JSON (as a string) to the report-writer in Phase 5.

## Phase 5: Write HTML report (DELEGATED to report-writer agent)

Do NOT write the HTML report yourself.

1. Output file path: `~/.claude/plugin-insights/report-<PLUGIN_FILTER>-<YYYY-MM-DD-HHmm>.html`
   (or `report-all-...` if no filter). Create directory with `Bash(mkdir -p ~/.claude/plugin-insights)`.

2. Resolve the template path: `${CLAUDE_SKILL_DIR}/report-template.html` (absolute path).

3. Spawn the report-writer agent:

```
Agent(
  subagent_type: "plugin-insights:pi-report-writer",
  model: "sonnet",
  prompt: "Write the plugin insights HTML report.\n\nTemplate file: <ABSOLUTE-PATH-TO-TEMPLATE>\nOutput file: <ABSOLUTE-OUTPUT-PATH>\n\nData:\n<JSON-STRING-FROM-PHASE-4>"
)
```

The prompt MUST contain:
- The absolute template file path (not a relative path, not ${CLAUDE_SKILL_DIR})
- The absolute output file path under ~/.claude/plugin-insights/
- The complete JSON data string from Phase 4

4. Print the full path to the generated report file.

## Error handling

- If no JSONL files found in date range: report "No sessions found" and stop.
- If no plugin activity found: write a minimal report with the empty state message via report-writer.
- If all parser batches fail: write an error report listing which batches failed and why.
- Individual session failures are acceptable. Log them but continue processing.

## Optional: Plugin-specific metrics

After Phase 2, check for plugin-specific metric files:
- `~/.claude/deep-research/metrics.jsonl`
- Similar patterns for other known plugins

If found, read and merge into aggregate data.
