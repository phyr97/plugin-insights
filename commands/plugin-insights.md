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
---

Invoke the `plugin-insights:plugin-insights` skill with the following input:

- Arguments: $ARGUMENTS
- Parse any options (--days) from the arguments
- If a plugin name is given without flag, use it as the plugin filter

The skill handles everything: session discovery, parser dispatch, facet extraction, aggregation, and HTML report generation.
