---
name: plugin-insights
description: Analyze how well Claude Code plugins perform across sessions
arguments:
  - name: plugin
    description: Optional plugin name to filter (e.g. "deep-research")
    required: false
  - name: --days
    description: Number of days to analyze (default 7)
    required: false
allowed-tools: Read, Glob, Grep, Bash(python3:*), Bash(echo:*), Bash(ls:*), Bash(wc:*), Bash(mkdir:*), Agent, Write
---

## Iron Laws (NON-NEGOTIABLE)

1. NEVER read full JSONL files yourself. Delegate ALL parsing to pi-session-parser agents.
2. NEVER do qualitative assessment yourself. Delegate to pi-facet-extractor agents.
3. ALWAYS write an HTML report. No chat-only output.
4. Respect batch sizes: max 15 sessions per parser, max 15 assessments per facet-extractor.
5. ALWAYS use Bash(python3:...) for template placeholder replacement. NEVER fill HTML templates by string manipulation yourself.

## Phase 1: Discovery

1. Parse arguments: extract optional plugin filter and --days N (default 7).
2. List all JSONL session files:
   ```
   Glob("~/.claude/projects/*/*.jsonl")
   ```
3. Filter by timestamp (file mtime within --days range) using Bash(python3:...):
   ```python
   import os, time; cutoff = time.time() - DAYS*86400
   # filter files where mtime >= cutoff
   ```
4. Group filtered files into batches of 15.
5. Report to user: "Found N sessions in M projects from the last D days. Processing in B batches..."

## Phase 2: Parse (parallel batches)

Spawn pi-session-parser agents. For each batch:

```
Agent(
  subagent_type: "plugin-insights:pi-session-parser",
  model: "haiku",
  prompt: "Parse these JSONL files for plugin activity.\n\nFiles:\n<file-list>\n\nPlugin root: <PLUGIN_ROOT_PATH>\n\n<optional: Only extract data for plugin: X>"
)
```

- Spawn up to 4 parser agents in parallel per round
- Collect structured JSON results from each agent
- If a plugin filter was given, discard sessions without that plugin
- On agent failure: log the error, continue with remaining batches

## Phase 3: Facet extraction (parallel)

Collect all `excerpts` from parser results. Bundle into batches of max 15 assessments.

Spawn pi-facet-extractor agents:

```
Agent(
  subagent_type: "plugin-insights:pi-facet-extractor",
  model: "haiku",
  prompt: "Assess these plugin usage excerpts.\n\nPlugin root: <PLUGIN_ROOT_PATH>\n\nExcerpts:\n<json-excerpts>"
)
```

- Spawn up to 4 facet agents in parallel
- Collect structured assessment JSON
- On agent failure: log the error, continue with remaining batches

## Phase 4: Aggregate and report

1. Compute per-plugin aggregates:
   - Total invocations, sessions, unique projects
   - Token consumption (input + output, main + subagents)
   - Error rate (errors / invocations)
   - Average correction_score, average completion_score
   - Top friction points (group similar ones by keyword overlap)
   - Top strengths (group similar ones)
   - Trend over time (weekly buckets based on session timestamps)

2. Generate improvement suggestions per plugin:
   - High correction scores (avg > 1.5) -> "Plugin prompt needs refinement"
   - High error rates (> 20%) -> "Tool configuration or permission issues"
   - Low completion (avg < 1.0) -> "Scope or reliability problems"

3. Build a JSON object containing all aggregated data and write it to `/tmp/plugin-insights-data.json`

4. Read the HTML template:
   ```
   Read("{{PLUGIN_ROOT}}/skills/plugin-insights/report-template.html")
   ```

5. Use Bash(python3:...) to render the report:
   ```python
   import json
   with open('/tmp/plugin-insights-data.json') as f:
       data = json.load(f)
   with open('<TEMPLATE_PATH>') as f:
       html = f.read()
   # Replace all {{placeholder}} values using str.replace()
   # For {{plugin_sections}}, build HTML string from data
   # Write to ~/.claude/plugin-insights/report.html
   ```

6. Print: "Report written to ~/.claude/plugin-insights/report.html"

## Error handling

- If no JSONL files found in date range: report "No sessions found" and stop.
- If no plugin activity found: write a minimal HTML report stating "Keine Plugin-Aktivitaet im Zeitraum gefunden" and print the path.
- If all parser batches fail: write an error report listing which batches failed and why. Do not produce an empty report silently.
- Individual session failures are acceptable. Log them but continue processing.

## Optional: Plugin-specific metrics

After Phase 2, check for plugin-specific metric files:
- `~/.claude/deep-research/metrics.jsonl`
- Similar patterns for other known plugins

If found, read and merge relevant metrics into the aggregate data.
