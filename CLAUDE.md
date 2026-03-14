# plugin-insights

Analysiert JSONL-Session-Transkripte von Claude Code und generiert einen HTML-Report, der zeigt, wie gut installierte Plugins in der Praxis funktionieren.

## Befehle

- `/plugin-insights` -- Alle Plugins der letzten 7 Tage analysieren
- `/plugin-insights <name>` -- Nur ein bestimmtes Plugin analysieren (z.B. `deep-research`)
- `/plugin-insights --days N` -- Analysezeitraum anpassen (Standard: 7 Tage)

## Architektur

Dreistufige Agent-Hierarchie:

1. Orchestrator (Sonnet): Koordiniert den Ablauf, aggregiert Ergebnisse, rendert den HTML-Report per Python-Script
2. Parser-Agents (Haiku): Lesen JSONL-Dateien, extrahieren Plugin-Aktivitaet und Token-Usage
3. Facet-Agents (Haiku): Bewerten Plugin-Nutzung qualitativ anhand von Message-Excerpts

Der Orchestrator liest nie selbst JSONL-Dateien. Alles Parsing wird an Parser-Agents delegiert. Template-Rendering laeuft ueber `python3` mit `str.replace()`.

## Verzeichnisstruktur

```
commands/plugin-insights.md           # Orchestrator slash command
agents/pi-session-parser.md           # JSONL parser agent (Haiku)
agents/pi-facet-extractor.md          # Qualitative facet extraction (Haiku)
skills/plugin-insights/
  references/
    jsonl-schema.md                   # JSONL field documentation
    facet-dimensions.md               # Qualitative assessment dimensions
    detection-patterns.md             # Plugin detection patterns
  report-template.html                # HTML report template
```

## Output

Report wird geschrieben nach `~/.claude/plugin-insights/report.html`. Selbststaendige HTML-Datei mit eingebettetem CSS, kein JavaScript.
