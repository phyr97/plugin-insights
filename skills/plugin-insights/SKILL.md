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
3. ALWAYS write an HTML report file. No chat-only output.
4. Respect batch sizes: max 15 sessions per parser, max 15 assessments per facet-extractor.
5. ALWAYS read the reference template before writing the report. Use EXACTLY the CSS and HTML structure from the template. Do not invent your own styles or layout.

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

## Phase 4: Aggregate

1. Compute per-plugin aggregates:
   - Total invocations, sessions, unique projects
   - Token consumption (input + output, main + subagents)
   - Error rate (errors / invocations)
   - Average correction_score, average completion_score (exclude default 1.0 scores if possible)
   - Top friction points (group similar ones by keyword overlap)
   - Top strengths (group similar ones)
   - Trend over time: bucket sessions by ISO week, compute invocations per week

2. Mark sessions from projects whose path contains the plugin name as "dev sessions". Report dev vs. production stats separately.

3. Generate improvement suggestions per plugin:
   - High correction scores (avg > 1.5) -> "Plugin prompt needs refinement"
   - High error rates (> 20%) -> "Tool configuration or permission issues"
   - Low completion (avg < 1.0) -> "Scope or reliability problems"

## Phase 5: Write HTML report

1. Read the reference template:
   ```
   Read("${CLAUDE_SKILL_DIR}/report-template.html")
   ```

2. Write the final report using the Write tool to `~/.claude/plugin-insights/report.html`. Create the directory with `Bash(mkdir -p ~/.claude/plugin-insights)` first.

### Report structure rules (follow exactly)

The report MUST be a single self-contained HTML file. Copy the EXACT `<style>` block from the reference template including all CSS variables, media queries, and class definitions. Do not modify, simplify, or omit any CSS.

The HTML body structure:
- `<h1>Plugin Insights Report</h1>`
- `<p class="subtitle">` with generation date and analysis period
- `<div class="grid">` with exactly 4 `.card` elements: plugins analyzed, sessions with plugins, total tokens, error rate
- One `<section>` per plugin containing:
  - `<h2>` with plugin name
  - `.stats-row` with invocations, sessions, projects, error rate
  - `.stats-row` with score badges (`.score-badge .score-good`/`.score-ok`/`.score-bad`)
  - `<h3>Agent types used</h3>` with `.plugin-tag` spans
  - `<h3>Weekly trend</h3>` with `.trend-bar` containing `.week` divs (height proportional to max week)
  - `<h3>Projects</h3>` with `<ul>` listing projects and their invocation counts, dev sessions marked as "(dev)"
  - `<h3>Friction points</h3>` with `<ul>` if any, omit section if none
  - `<h3>Strengths</h3>` with `<ul>` if any, omit section if none
  - `<h3>Suggestions</h3>` with `.suggestion` divs if any
- If no plugin activity found: `<div class="empty-state">No plugin activity found in the selected period</div>`
- `<p class="footer">Generated by plugin-insights v0.1.0</p>`

### Score badge rules

- correction_score: 0-0.5 = `.score-good`, 0.5-1.5 = `.score-ok`, >1.5 = `.score-bad`
- completion_score: 1.5-2.0 = `.score-good`, 1.0-1.5 = `.score-ok`, <1.0 = `.score-bad`
- error_rate: 0-5% = `.score-good`, 5-20% = `.score-ok`, >20% = `.score-bad`

### Trend bar rules

Show the last 7 weeks. Each `.week` div height is `(week_count / max_week_count) * 100%` of the `.trend-bar` height (30px). Minimum height 2px.

3. Print: "Report written to ~/.claude/plugin-insights/report.html"

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
