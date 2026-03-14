# Facet extraction dimensions

Qualitative dimensions for assessing plugin usage from message excerpts.

## Dimensions

### 1. User correction after plugin usage

Did the user correct, reject, or manually redo work after a plugin invocation?

Look for signals like "nein", "nicht das", "stopp", "falsch", "lass das", "anders", "vergiss", undo requests, or the user manually performing what the plugin was supposed to do.

- Score 0: No correction needed
- Score 1: Minor correction or clarification
- Score 2: Significant rework required
- Score 3: Plugin output abandoned or session aborted in frustration

### 2. Task completion

Was the task the plugin was invoked for completed successfully?

- Score 0: Aborted, plugin output not used at all
- Score 1: Partially completed, user had to finish manually
- Score 2: Fully completed as intended

### 3. Friction points

Recurring problems observed during plugin usage. Free text, max 1 sentence. Examples: "Plugin ignored explicit instructions", "Output was unusable", "Took too many turns", "Wrong tool selection". Set to null if no friction observed.

### 4. Strengths

What worked consistently well? Free text, max 1 sentence. Examples: "Research results were comprehensive", "Code was production-ready on first try", "Fast parallel execution". Set to null if nothing stood out positively.

## Scope rule

Only assess messages surrounding plugin invocations: 5 messages before and 10 messages after the plugin call. Do not assess the entire session, only the plugin-relevant window.

## Ambiguity rule

When the context is unclear or insufficient to make a confident judgment, set correction_score to 1 and completion_score to 1. Do not guess.
