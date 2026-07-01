---
description: Create a Jira task from a pasted Figma comment. Usage: /ftj <comment text and/or screenshot> <figma link> [EPIC-KEY]
argument-hint: <comment/link> [EPIC-KEY]
---

Create a Jira task from the following input using the **create flow** in the
`figma-to-jira` skill:

$ARGUMENTS

Parse the input as defined by the skill:
- Find the Figma link (required — if missing or not a comment link, ask for a
  screenshot, the text, or a corrected link, then stop).
- Detect an optional epic key matching `[A-Z][A-Z0-9]+-\d+` (e.g. SHELL-141). If
  present, attach directly to that epic and skip recommendation.
- Treat the rest (text and/or any pasted screenshot) as the comment content.

Then write a concise, outcome-focused task (title, compact description as a user
story, 2–4 acceptance criteria, Figma link as source), recommend an epic if none
was given, show a preview for my approval, and create the issue on approval —
exactly per the figma-to-jira skill. If `./.figma-to-jira/config.json` is
missing, tell me to run /ftj-setup first.
