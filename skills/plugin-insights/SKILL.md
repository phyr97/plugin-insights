---
name: plugin-insights
description: |
  Analyze how well Claude Code plugins perform across sessions.
  Use when: "plugin insights", "plugin performance", "how are my plugins doing",
  "plugin analysis", "plugin report", "which plugins work well".
  Generates an HTML report from JSONL session transcripts.
allowed-tools: Read, Glob, Grep, Bash(python3:*), Bash(echo:*), Bash(ls:*), Bash(wc:*), Bash(mkdir:*), Agent, Write
---

## Iron Laws (NON-NEGOTIABLE)

1. NEVER read full JSONL files yourself. Delegate ALL parsing to pi-session-parser agents.
2. NEVER do qualitative assessment yourself. Delegate to pi-facet-extractor agents.
3. ALWAYS write an HTML report using the template file. NEVER generate your own HTML structure.
4. Respect batch sizes: max 15 sessions per parser, max 15 assessments per facet-extractor.
5. ALWAYS use Bash(python3:...) to read the template file and replace placeholders via str.replace(). The python script must read the ACTUAL template file from disk. NEVER write HTML yourself.
6. ALWAYS read the template from `${CLAUDE_SKILL_DIR}/report-template.html`. Pass this exact path to the python script.

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

Spawn pi-session-parser agents. Each agent has inline references in its definition, so no external files need to be passed. For each batch:

```
Agent(
  subagent_type: "plugin-insights:pi-session-parser",
  model: "haiku",
  prompt: "Parse these JSONL files for plugin activity.\n\nFiles:\n<file-list>\n\n<optional: Only extract data for plugin: X>"
)
```

- Spawn up to 4 parser agents in parallel per round
- Collect structured JSON results from each agent
- If a plugin filter was given, discard sessions without that plugin
- On agent failure: log the error, continue with remaining batches

## Phase 3: Facet extraction (parallel)

Collect all `excerpts` from parser results. Bundle into batches of max 15 assessments.

Spawn pi-facet-extractor agents. Each agent has inline assessment dimensions, so no external files need to be passed:

```
Agent(
  subagent_type: "plugin-insights:pi-facet-extractor",
  model: "haiku",
  prompt: "Assess these plugin usage excerpts.\n\nExcerpts:\n<json-excerpts>"
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
   - Average correction_score, average completion_score (exclude default 1.0 scores from averages if possible)
   - Top friction points (group similar ones by keyword overlap)
   - Top strengths (group similar ones)
   - Trend over time: bucket sessions by ISO week, compute invocations per week for the last 7 weeks

2. Mark sessions from projects whose path contains the plugin name itself as "dev sessions" (e.g. sessions from `*-deep-research*` when analyzing deep-research). Report dev vs. production stats separately.

3. Generate improvement suggestions per plugin:
   - High correction scores (avg > 1.5) -> "Plugin prompt needs refinement"
   - High error rates (> 20%) -> "Tool configuration or permission issues"
   - Low completion (avg < 1.0) -> "Scope or reliability problems"

4. Build a JSON object containing all aggregated data. The JSON must include these keys matching template placeholders:
   - `generated_date`, `date_range`
   - `total_plugins`, `total_sessions`, `total_sessions_scanned`
   - `total_tokens` (formatted like "1.2M" or "450K", or "N/A" if unavailable)
   - `error_rate`, `error_rate_class` ("score-good"/"score-ok"/"score-bad"), `total_errors`
   - `plugin_sections`: a complete HTML string for all per-plugin sections
   - `empty_state`: empty string if plugins found, or `<div class="empty-state">No plugin activity found</div>` if none
   Write this JSON to `/tmp/plugin-insights-data.json`.

5. The `plugin_sections` HTML string must use the CSS classes defined in the template:
   - `.stats-row` with `.stat-label` and `.stat-value` spans
   - `.score-badge` with `.score-good`/`.score-ok`/`.score-bad`
   - `.plugin-tag` for agent types
   - `.suggestion` for improvement suggestions
   - `.trend-bar` with `.week` divs for weekly trend (height as percentage of max week)
   - `<section>` wrapper with `<h2>` plugin name and `<h3>` sub-headings

6. Read the HTML template file and render the report using Bash(python3:...):
   ```python
   import json, os
   with open('/tmp/plugin-insights-data.json') as f:
       data = json.load(f)
   with open('<TEMPLATE_PATH>') as f:
       html = f.read()
   for key, value in data.items():
       html = html.replace('{{' + key + '}}', str(value))
   os.makedirs(os.path.expanduser('~/.claude/plugin-insights'), exist_ok=True)
   with open(os.path.expanduser('~/.claude/plugin-insights/report.html'), 'w') as f:
       f.write(html)
   ```
   TEMPLATE_PATH is the path to `report-template.html` read via `${CLAUDE_SKILL_DIR}/report-template.html`.

7. Print: "Report written to ~/.claude/plugin-insights/report.html"

## Error handling

- If no JSONL files found in date range: report "No sessions found" and stop.
- If no plugin activity found: write a minimal report using the template with `{{empty_state}}` filled and `{{plugin_sections}}` empty.
- If all parser batches fail: write an error report listing which batches failed and why. Do not produce an empty report silently.
- Individual session failures are acceptable. Log them but continue processing.

## Optional: Plugin-specific metrics

After Phase 2, check for plugin-specific metric files:
- `~/.claude/deep-research/metrics.jsonl`
- Similar patterns for other known plugins

If found, read and merge into aggregate data.
