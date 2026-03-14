# plugin-insights

Analyzes JSONL session transcripts from Claude Code and generates an HTML report showing how well installed plugins perform in practice.

## Commands

- `/plugin-insights` -- Analyze all plugins from the last 7 days
- `/plugin-insights <name>` -- Analyze a specific plugin (e.g. `deep-research`)
- `/plugin-insights --days N` -- Adjust the analysis period (default: 7 days)

## Architecture

Three-tier agent hierarchy:

1. Orchestrator (session model): Coordinates the workflow, aggregates results, renders the HTML report via Python script
2. Parser agents (Haiku): Read JSONL files, extract plugin activity and token usage
3. Facet agents (Haiku): Assess plugin usage qualitatively based on message excerpts

The orchestrator never reads JSONL files directly. All parsing is delegated to parser agents. Template rendering runs via `python3` with `str.replace()`.

## Directory structure

```
commands/plugin-insights.md           # Orchestrator slash command
agents/pi-session-parser.md           # JSONL parser agent (Haiku)
agents/pi-facet-extractor.md          # Qualitative facet extraction (Haiku)
skills/plugin-insights/
  SKILL.md                            # Orchestration logic
  references/
    jsonl-schema.md                   # JSONL field documentation
    facet-dimensions.md               # Qualitative assessment dimensions
    detection-patterns.md             # Plugin detection patterns
  report-template.html                # HTML report template
```

## Output

Report is written to `~/.claude/plugin-insights/report.html`. Self-contained HTML file with embedded CSS, no JavaScript.
