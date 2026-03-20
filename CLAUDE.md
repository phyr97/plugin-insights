# plugin-insights

Analyzes JSONL session transcripts from Claude Code and generates an HTML report showing how well installed plugins perform in practice.

## Commands

- `/plugin-insights` -- Analyze all plugins from the last 7 days
- `/plugin-insights <name>` -- Analyze a specific plugin (e.g. `deep-research`)
- `/plugin-insights --days N` -- Adjust the analysis period (default: 7 days)

Fuzzy matching: if the plugin name doesn't match exactly, the command suggests similar plugin names found in recent sessions.

## Architecture

Three-phase agent pipeline:

1. Orchestrator (session model): Discovers sessions, pre-filters by plugin, coordinates agents, aggregates results, writes the HTML report
2. Parser agents (Sonnet): Read JSONL files, extract verified plugin invocations, behavioral metrics, token usage, and context for quality review
3. Quality reviewer (Sonnet): Evaluates the most recent plugin invocations for content relevance, depth, and actionability

The orchestrator pre-filters sessions via Grep before sending them to parsers. This reduces the number of agents needed (typically 2-3 instead of 8+). Behavioral scores (correction, completion) are derived from structural patterns, not LLM assessment.

## Directory structure

```
commands/plugin-insights.md           # Slash command (thin wrapper)
agents/pi-session-parser.md           # JSONL parser agent (Sonnet)
agents/pi-quality-reviewer.md         # Content quality reviewer (Sonnet)
agents/pi-report-writer.md            # HTML report writer (Sonnet)
skills/plugin-insights/
  SKILL.md                            # Orchestration logic
  references/
    jsonl-schema.md                   # JSONL field documentation
    detection-patterns.md             # Plugin detection patterns
  report-template.html                # HTML report reference template
```

## Output

Reports are written to `~/.claude/plugin-insights/report-<plugin>-<date-time>.html`. Each run creates a new file, previous reports are preserved. Self-contained HTML with embedded CSS, no JavaScript. Supports light and dark mode via system preference.

## Permissions setup

This plugin reads JSONL session files from `~/.claude/projects/`. Claude Code will ask for permission on each directory the first time. To avoid repeated prompts, add this to your `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Read(//Users/<your-username>/.claude/projects/**)",
      "Grep(//Users/<your-username>/.claude/projects/**)"
    ]
  }
}
```

Replace `<your-username>` with your system username. The double slash `//` marks an absolute path.

## Known issues

Extended thinking (high/adaptive think budget) on Opus can cause the orchestrator to bypass SKILL.md instructions entirely (GitHub issue #23936). If the plugin produces unexpected output, try running it with thinking disabled or on Sonnet.

## No external dependencies

No Python, Node, or other runtimes required. The orchestrator delegates HTML report writing to a dedicated report-writer agent. Session discovery uses `find`.

## SessionStart Hook

Wenn die SessionStart Hook eine Ausgabe liefert (z.B. CC-Update erkannt), fuehre die Anweisungen aus der Hook-Ausgabe sofort und automatisch aus, bevor du auf den User wartest.
